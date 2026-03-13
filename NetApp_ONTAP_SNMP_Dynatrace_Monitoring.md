# NetApp ONTAP SNMP Trap Monitoring — Dynatrace Implementation Guide

> **Who this is for:**  
> - **Storage Engineers / NetApp Admins** — configure SNMP on your ONTAP cluster and understand what gets alerted and why  
> - **Observability / Platform Engineers** — understand the Dynatrace processing pipeline, event rules, noise reduction strategy, and how to deploy/maintain this at scale  
> - **SRE / Operations Teams** — understand severity mapping, exclusion logic, and how incidents flow from storage trap → ITSM ticket

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture & Signal Flow](#architecture--signal-flow)
3. [NetApp ONTAP SNMP — Fundamentals](#netapp-ontap-snmp--fundamentals)
   - [SNMP Versions Supported](#snmp-versions-supported)
   - [MIB Structure & Key OIDs](#mib-structure--key-oids)
   - [Trap Types](#trap-types)
   - [Severity Model](#severity-model)
4. [Enabling SNMP on NetApp ONTAP](#enabling-snmp-on-netapp-ontap)
5. [Dynatrace Implementation](#dynatrace-implementation)
   - [Design Philosophy: Why Processing Rules Instead of 100 Event Rules](#design-philosophy-why-processing-rules-instead-of-100-event-rules)
   - [Processing Rule 1 — Severity Derivation](#processing-rule-1--severity-derivation)
   - [Processing Rule 2 — Noise Exclusions](#processing-rule-2--noise-exclusions)
   - [Event Extraction Rule — High Severity Trap Alerting](#event-extraction-rule--high-severity-trap-alerting)
   - [Exception Rules — Specific Built-in Traps That Escape Severity Derivation](#exception-rules--specific-built-in-traps-that-escape-severity-derivation)
6. [Severity Filtering Strategy](#severity-filtering-strategy)
7. [Noise Exclusion Strategy](#noise-exclusion-strategy)
8. [DataFabric Manager Trap Support](#datafabric-manager-trap-support)
9. [ServiceNow CI Mapping](#servicenow-ci-mapping)
10. [Troubleshooting](#troubleshooting)
11. [Key OID Reference](#key-oid-reference)
12. [ITSM Integration — Incident Lifecycle Design](#itsm-integration--incident-lifecycle-design)
13. [Best Practices](#best-practices)

---

## Overview

This solution implements **enterprise-grade SNMP trap monitoring for NetApp FAS/AFF storage systems** using Dynatrace as the observability platform. It covers:

- Full SNMP trap ingestion from NetApp ONTAP clusters via Dynatrace ActiveGate — **each trap is converted into a Dynatrace log record** (this is the key architectural fact: from Dynatrace's perspective, traps are logs, not metrics or custom events)
- Intelligent severity classification directly from the trap OID structure
- Pre-event noise suppression using Dynatrace Log Processing (DPP) rules
- High-signal event extraction targeting only Emergency, Alert, Critical, and Error severities
- Automated routing logic for ServiceNow incident creation

The design is optimised for **managed/MSP environments** where the same pipeline serves multiple customer storage estates simultaneously, but the approach applies equally to single-tenant deployments.

**Scale context this was built for:**  
- 150+ enterprise storage customers on a shared observability platform  
- Trap volumes in the tens of thousands per day across the fleet  
- Zero-touch incident noise suppression: ~85% trap noise eliminated before incident creation  

---

## Architecture & Signal Flow

```
NetApp ONTAP Cluster
  │
  │  SNMP Trap (UDP 162)
  ▼
Dynatrace ActiveGate (SNMP Trap Receiver)
  │
  │  Raw trap ingested as log record in Dynatrace
  │  log.source = "snmptraps"
  │  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.<trap_number>"
  ▼
DPP Processing Rule 1 — Severity Derivation
  │  Extracts last digit of snmpv2-mib::snmptrapoid as integer "Severity"
  ▼
DPP Processing Rule 2 — Noise Exclusion
  │  Parses enterprises.789.1.1.12.0 (trap message)
  │  Sets snmp_netapp_Exclude_parameter = true/false
  ▼
  ├──► Primary Event Rule — NetApp_SNMP_HighSeverity_Traps
  │      Severity IN (1,2,3,4) AND Exclude = false
  │      Covers: all generic severity traps (789.0.11–18)
  │
  ├──► Exception Rule A — NetApp_SNMP_HardwareWarnings
  │       5 specific OIDs: disk PFA/degraded-IO, ECC high-freq, IO card, SAS cable
  │
  ├──► Exception Rule B — NetApp_SNMP_ClusterNetworkSAN
  │       5 specific OIDs: HA takeover, all-NICs-down, FC fail, Flash Cache, vFiler
  │
  ├──► Exception Rule C — NetApp_SNMP_VolumeCapacityDataProtection
  │       4 specific OIDs: near-full, UPS battery, dir-full, SnapMirror OOS
  │
  └──► Exception Rule D — NetApp_SNMP_AVSecurityCompliance
          2 specific OIDs: AV scanner disconnected, AV service disabled
  │
  ▼
Dynatrace Problem / Alert
  │
  ▼
ServiceNow ITSM — Incident Creation & Routing
```

> **Key insight:** NetApp ONTAP sends two distinct categories of traps under `1.3.6.1.4.1.789.0.*`. The **generic severity traps** (`789.0.11` through `789.0.18`) follow a fixed scheme where the last digit of the OID directly encodes severity: 1=Emergency, 2=Alert, 3=Critical, 4=Error, 5=Warning, 6=Notification, 7=Informational, 8=Debug. When ONTAP raises any internal event, it fires one of these generic severity traps carrying the event detail in `productTrapData` (OID `789.1.1.12.0`). The **specific built-in traps** (`789.0.21`, `789.0.72`, `789.0.82`, etc.) are named hardware events — their OID numbers are arbitrary identifiers, NOT severity codes. The processing pipeline exploits the generic severity trap structure: extract the last digit, filter on 1–4, and one rule covers the entire ONTAP trap estate.

> **For Dynatrace users — traps are logs:** The Dynatrace ActiveGate SNMP extension converts every incoming SNMP trap into a **Dynatrace log record**. This is the foundational architectural fact of this entire implementation. Once the trap lands in Dynatrace it is a log — queryable via DQL with `filter log.source == "snmptraps"`, enrichable via Log Processing (DPP) rules, and alertable via Log Events (event extraction rules). If you are a Dynatrace engineer approaching this for the first time: think **Logs → DPP → Log Events**, not metrics or custom events. This is why all processing rules, exclusion logic, and event queries in this guide use log-layer constructs.

---

## NetApp ONTAP SNMP — Fundamentals

### SNMP Versions Supported

| ONTAP Version | SNMPv2c | SNMPv3 | Notes |
|---|---|---|---|
| 8.2.x (7-Mode & C-Mode) | ✅ | ✅ | Last version supporting 7-Mode |
| 8.3.x – 9.0.x (C-Mode only) | ✅ | ✅ | 7-Mode deprecated |
| 9.1.x and above (C-Mode only) | ❌ | ✅ | SNMPv2c dropped; SNMPv3 mandatory |

> **Important:** If you are running ONTAP 9.1+, SNMPv3 is required. SNMPv1 is technically supported by NetApp but should never be used — it has no security model and sends community strings in plaintext.

- **SNMP port:** UDP 161 (polling), UDP 162 (traps)
- **Cluster-mode** supports up to 12 nodes for SAN and 24 nodes for NAS  
- **7-Mode** is a deprecated two-node legacy architecture — migrate to C-Mode if still in use

---

### MIB Structure & Key OIDs

NetApp's enterprise OID root is `.1.3.6.1.4.1.789` (Enterprise ID: 789).

```
enterprises (1.3.6.1.4.1)
└── netapp (789)
    ├── netapp0 (789.0)      ← TRAP definitions live here
    ├── netapp1 (789.1)      ← Current MIB data objects
    │   └── product (789.1.1)
    │       ├── productSerialNum    (789.1.1.9.0)   ← Node serial number
    │       └── productTrapData     (789.1.1.12.0)  ← Human-readable trap message (KEY FIELD)
    ├── netappDataFabricManager (789.3)  ← DFM 5.x traps
    ├── netappOnCommand (789.5)          ← DFM 6.x / OnCommand traps
    └── netappProducts (789.2)           ← sysObjectID values
```

**The two OIDs you will use most in your observability rules:**

| OID | Field Name | Purpose |
|---|---|---|
| `1.3.6.1.4.1.789.1.1.9.0` | `productSerialNum` | Uniquely identifies the node — use for CI correlation |
| `1.3.6.1.4.1.789.1.1.12.0` | `productTrapData` | Human-readable message describing the event — parse this for context |

---

### Trap Types

NetApp ONTAP emits three categories of traps:

**1. Standard SNMP Traps (RFC 1215)**  
Vendor-neutral traps defined in the SNMP standard. NetApp emits: `coldStart`, `warmStart`, `linkDown`, `linkUp`, `authenticationFailure`.

**2. Built-in ONTAP Traps**  
Predefined in the ONTAP firmware. Automatically generated when specific hardware or software events occur. OID numbers are sequential catalogue IDs assigned by NetApp — they do NOT encode severity.

The complete built-in trap catalogue (258 traps across 38 categories, sourced from MIB v2.20) is maintained as a separate reference document in this repository, with per-trap alert status and rule assignment:

**→ [`NetApp_ONTAP_BuiltinTraps_Reference.md`](./NetApp_ONTAP_BuiltinTraps_Reference.md)**

That document covers every trap with: full OID, trap name, category, description, whether it generates a Dynatrace alert (✅/❌), and which of the five event rules catches it.

**Summary by domain:**

| Category | Traps | Alerted |
|---|:---:|:---:|
| Disk | 10 | 9 |
| Fan | 4 | 3 |
| Power Supply | 5 | 4 |
| HA Cluster / Failover | 10 | 6 |
| Volume & Capacity | 27 | 20 |
| Temperature | 7 | 5 |
| Disk Shelf & Enclosure | 18 | 12 |
| Chassis | 21 | 16 |
| Fibre Channel & SAN | 21 | 13 |
| Antivirus / VScan | 29 | 21 |
| CIFS / Active Directory | 9 | 8 |
| All other categories | 97 | 58 |
| **Total** | **258** | **175** |


---

### Severity Model

NetApp ONTAP uses two distinct OID schemes under `789.0.*` and it is critical to understand the difference — confusing them leads to incorrect alerting logic.

#### Generic Severity Traps — `789.0.11` to `789.0.18`

These eight traps are the backbone of ONTAP's alerting model. When any internal event occurs on the cluster, ONTAP fires one of these generic traps and carries the full event description as a text string in `productTrapData` (`789.1.1.12.0`). The OID itself encodes the severity — specifically, the **units digit of the full OID number** maps directly to severity level 1–8:

| OID | Trap Name | Severity | Action |
|---|---|---|---|
| `789.0.11` | `emergencyTrap` | 1 — Emergency | ✅ Alert |
| `789.0.12` | `alertTrap` | 2 — Alert | ✅ Alert |
| `789.0.13` | `criticalTrap` | 3 — Critical | ✅ Alert |
| `789.0.14` | `errorTrap` | 4 — Error | ✅ Alert |
| `789.0.15` | `warningTrap` | 5 — Warning | ❌ Suppressed |
| `789.0.16` | `notificationTrap` | 6 — Notification | ❌ Suppressed |
| `789.0.17` | `informationalTrap` | 7 — Informational | ❌ Suppressed |
| `789.0.18` | `dbgTrap` | 8 — Debug | ❌ Suppressed |

The processing rule `SUBSTR(snmpv2-mib::snmptrapoid, -1)` works here because for this specific OID range (789.0.1x), the last digit of the OID string always equals the severity level. For example:
```
.1.3.6.1.4.1.789.0.14  →  last digit = 4  →  Severity = Error  →  ✅ Alert
.1.3.6.1.4.1.789.0.15  →  last digit = 5  →  Severity = Warning →  ❌ Suppressed
```

#### Specific Built-in Traps — all other OIDs under `789.0.*`

These are named, firmware-defined hardware events (`diskFailed`, `clusterNodeFailed`, `volumeFull`, etc.). Their OID numbers are arbitrary sequential identifiers assigned by NetApp — the digits carry **no severity meaning**. For example, `diskFailed` is `789.0.22` — the `2` here does not mean "Alert severity". These traps fire for specific hardware conditions independently of the generic severity scheme.

> **Why does the processing rule still work for built-in traps?**  
> It largely does not need to. In practice, when a significant built-in hardware event fires (e.g., `diskFailed` at `789.0.22`), ONTAP simultaneously or subsequently fires the corresponding generic severity trap (e.g., `alertTrap` at `789.0.12`) carrying the same event detail in `productTrapData`. The generic severity trap is what the processing pipeline captures and alerts on. The specific built-in traps in the table above are documented here for completeness and reference — they are the underlying named events, but alerting is driven by the generic severity trap that accompanies them.

> **Why suppress severities 5–8?** In a large managed environment, Warning and below generate enormous volume with very low actionability. Severities 1–4 represent conditions that require a human response. Severities 5–8 indicate informational state changes — valuable for debugging but not for operational incident management. The suppression is configurable and can be relaxed per customer if specific warning-level events are operationally justified.

---

## Enabling SNMP on NetApp ONTAP

### Cluster-Mode (ONTAP 8.3+)

**Step 1: Enable SNMP on the cluster**
```bash
# Enable SNMP
system snmp init -init 1

# Verify SNMP is enabled
system snmp show
```

**Step 2: Set community string (SNMPv2c) or user (SNMPv3)**
```bash
# SNMPv2c — set community string
system snmp community add -community-name <community_string> -type ro

# SNMPv3 — create SNMP user (recommended for ONTAP 9.1+)
security login create -username snmpv3user -application snmp \
  -authmethod usm -role readonly
```

**Step 3: Configure trap hosts (the Dynatrace ActiveGate IP)**
```bash
# Add the SNMP trap receiver
system snmp traphost add -peer-address <activegate_ip>

# Verify traphost configuration
system snmp traphost show
```

**Step 4: Test trap delivery**
```bash
# Generate a test trap
system snmp test
```

### 7-Mode (Legacy — Migrate if possible)

```bash
# Enable SNMP
options snmp.enable on

# Set trap host
snmp traphost add <activegate_ip>

# Test
snmp test
```

> **Note:** Screenshots of the storage UI configuration are not included here as they require access to a live ONTAP system. The CLI commands above are the canonical, version-stable approach.

---

## Dynatrace Implementation

### Design Philosophy: Why Processing Rules Instead of 100 Event Rules

The naive approach to SNMP trap monitoring is to create one event extraction rule per OID. For NetApp alone this means 100+ rules — all requiring individual maintenance, individual testing, and individual tuning as trap definitions evolve across ONTAP versions.

This implementation takes a different approach: **use Dynatrace Log Processing (DPP) rules as a pre-filter layer** so that a minimal set of event extraction rules handles the entire NetApp trap estate.

> **Why DPP rules? Because traps are logs.** The Dynatrace ActiveGate SNMP extension converts every SNMP trap into a log record — it does not create a metric data point or a custom event directly. This means every trap is immediately available in the Dynatrace log pipeline, and Log Processing (DPP) rules can enrich, transform, or flag it **before** any event extraction rule evaluates it. This is the architectural leverage point that makes the entire noise-reduction strategy possible. Without this conversion, you would have no pre-processing layer and would be forced into the naive one-rule-per-OID approach.

```
Naive Approach:          This Approach:
100+ event rules    →    2 DPP processing rules
                         + 5 event extraction rules
                         = complete coverage, 95% less maintenance
```

The two processing rules work as a pre-filter pipeline for all five event rules:
1. **Severity Rule** — classifies every incoming trap with a numeric severity derived from the OID last digit
2. **Exclusion Rule** — flags known-noisy traps before they reach event extraction
3. **Primary event rule** — fires on `Severity IN (1,2,3,4) AND excluded = false`, catching all generic severity traps
4. **Exception rules A–D** — four additional rules covering the 59 specific built-in traps that are operationally critical but have OID last-digits of 5–8, grouped by operational domain: Hardware, Cluster/Network/SAN, Volume/Capacity/Data-Protection, and AV/Security/Compliance

---

### Processing Rule 1 — Severity Derivation

**Rule name:** `SNMP_NetApp_Processing-netapp-severity`  
**Scope:** Environment  
**Applies to:** `snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.*"`

**Processing logic:**
```
USING(IN 'snmpv2-mib::snmptrapoid':STRING)
| FIELDS_ADD(Severity: INT(SUBSTR(COLUMN('snmpv2-mib::snmptrapoid'),-1)))
```

**What it does:**  
Takes the full trap OID value from the `snmpv2-mib::snmptrapoid` field and extracts the last character as an integer. For the generic severity trap OID range (`789.0.11` through `789.0.18`), the last digit directly equals the severity level. This becomes a new field `Severity` on the log record.

**Example — generic severity trap:**
```
snmpv2-mib::snmptrapoid = ".1.3.6.1.4.1.789.0.14"
                                              ↑
                              Last digit = 4 = ERROR severity
Result: Severity = 4  →  ✅ passes the event rule filter
```

```
snmpv2-mib::snmptrapoid = ".1.3.6.1.4.1.789.0.15"
                                              ↑
                              Last digit = 5 = WARNING severity
Result: Severity = 5  →  ❌ suppressed by event rule filter
```

**Important scope note:**  
This severity derivation is valid and meaningful only for the generic severity traps (`789.0.11`–`789.0.18`). For specific built-in traps with multi-digit OID numbers (`789.0.72`, `789.0.103`, etc.), the last digit is an arbitrary part of an identifier — not a severity code. The processing rule applies to all `789.0.*` traps, but the event extraction rule's severity filter (`Severity IN 1,2,3,4`) will coincidentally suppress many built-in traps whose last digit falls outside that range. This is acceptable because ONTAP fires the corresponding generic severity trap alongside major hardware events — that generic trap is what drives alerting.

**Sample log record (before processing) — errorTrap example:**
```json
{
  "log.source": "snmptraps",
  "snmp.trap_oid": "SNMPv2-SMI::enterprises.789.0.14",
  "snmpv2-mib::snmptrapoid": ".1.3.6.1.4.1.789.0.14",
  "device.address": "10.x.x.x",
  "SNMPv2-SMI::enterprises.789.1.1.12.0": "<human-readable event message>",
  "snmpv2-smi::enterprises.789.1.1.9.0": "<node serial number>"
}
```

**After processing:**
```json
{
  ...(all fields above),
  "Severity": 4
}
```

---

### Processing Rule 2 — Noise Exclusions

**Rule name:** `SNMP_NetApp_Processing-netapp-exclusions`  
**Scope:** Environment  
**Applies to:** `snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.*"`

**Processing logic:**
```
USING(IN 'SNMPv2-SMI::enterprises.789.1.1.12.0')
| PARSE(COLUMN('SNMPv2-SMI::enterprises.789.1.1.12.0'), "LD:Exclude_parameter")
| FIELDS_ADD(snmp_netapp_Exclude_parameter:(
    CONTAINS(Exclude_parameter, "security.invalid.login") OR
    CONTAINS(Exclude_parameter, "mgmtgwd.certificate.expired") OR
    CONTAINS(Exclude_parameter, "Nblade.cifsShrConnectFailed") OR
    CONTAINS(Exclude_parameter, "Nblade.noSMBVerNegotiated") OR
    CONTAINS(Exclude_parameter, "secd.rpc.authRequest.blocked") OR
    CONTAINS(Exclude_parameter, "secd.cifsAuth.problem") OR
    CONTAINS(Exclude_parameter, "wafl.dir.surprair.filename") OR
    CONTAINS(Exclude_parameter, "quota.parse.error") OR
    CONTAINS(Exclude_parameter, "dns.server.timed.out") OR
    CONTAINS(Exclude_parameter, "sshd.loginGraceTime.expired")
))
```

**What it does:**  
Parses the human-readable event message from `enterprises.789.1.1.12.0` and pattern-matches against known non-actionable event types. Sets `snmp_netapp_Exclude_parameter` to `true` if any match is found, `false` otherwise.

**Result field usage:**  
The event extraction rule then checks `snmp_netapp_Exclude_parameter = "false"` as a filter condition, meaning excluded traps never generate alerts regardless of their severity classification.

> **Important:** Both processing rules must be deployed and enabled. They operate on the same scope and complement each other. The exclusion rule runs on all `789.0.*` traps — it is not restricted to high-severity ones — so noise is eliminated before any downstream processing occurs.

---

### Event Extraction Rule — High Severity Trap Alerting

**Rule name:** `NetApp_SNMP_HighSeverity_Traps`  
**Schema:** `builtin:logmonitoring.log-events`  
**Scope:** Environment  
**Status:** Enabled

**Query:**
```
log.source = "snmptraps"
AND snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.*"
AND snmp_netapp_Exclude_parameter = "false"
AND (Severity = "1" OR Severity = "2" OR Severity = "3" OR Severity = "4")
```

**Event template:**
```
Title:       SNMP_Storage_NetApp: Details={snmp.1.3.6.1.4.1.789.0}
Description: Triggers on ONTAP traps under netapp-0 (1.3.6.1.4.1.789.0)
             for high severity events, excluding StorageGRID and known
             low-value messages.
Event type:  CUSTOM_ALERT
Davis merge: false
```

**Metadata injected on each event:**

| Key | Value |
|---|---|
| `Node_IP` | `{device.address}` |
| `Enterprise_ID` | `.1.3.6.1.4.1.789` |
| `dt.event.timeout` | `15` (minutes) |
| `dt.event.allow_frequent_issue_detection` | `false` |
| `dt.entity.host` | Linked via SNMP trap custom device entity |

**Davis Merge = false — why?**  
Storage trap events are discrete, specific hardware/software events. They should not be merged with application performance anomalies by Davis AI. Each trap represents a distinct condition on a specific storage component and warrants its own problem card for tracking and ITSM routing.

---

### Exception Rules — Specific Built-in Traps That Escape Severity Derivation

The primary rule above catches all generic severity traps (`789.0.11`–`789.0.18`) cleanly via the last-digit method. Specific built-in traps with OID last-digits of 5–8 that are operationally actionable for a storage team are covered by four additional event extraction rules. Two filtering passes were applied to keep the rule set lean:

**Pass 1 — Escalation coverage:** Warning traps with a guaranteed critical escalation trap already in the Primary rule are excluded. Example: `fanWarning` dropped because `fanFailed` and `fanFailureShutdown` already alert.

**Pass 2 — Storage team actionability:** Traps where the storage ops team has no meaningful real-time action (wrong team, physical-only fix, procurement items, system self-resolves) are excluded. Example: all AV license expiry traps dropped — procurement team action, not storage ops.

All four exception rules reuse `snmp_netapp_Exclude_parameter` from the DPP pipeline — noise exclusion still applies.

**Complete pipeline with exception rules:**
```
Primary Rule     → catches all generic severity traps (789.0.11–18) at Severity 1–4
                   + specific built-in traps whose OID last-digit is 1–4
Exception Rule A →  5 hardware traps: disk PFA/degraded-IO, ECC high-freq, IO card, SAS cable
Exception Rule B →  5 cluster/HA/network/SAN traps: HA takeover, all-NICs-down, FC fail, Flash Cache, vFiler
Exception Rule C →  4 volume/capacity traps: near-full, UPS battery, dir-full, SnapMirror OOS
Exception Rule D →  2 security traps: AV scanner disconnected, AV service disabled
──────────────────────────────────────────────────────────────────────────────────────────────
Event extraction rules: 5   Exception OIDs: 16   DPP processing rules: 2   Pipeline total: 7
```

---

#### Exception Rule A — Hardware: Disk, Memory, IO

**Rule name:** `NetApp_SNMP_HardwareWarnings`  
**Event type:** `CUSTOM_ALERT` | **Davis merge:** `false`  
**Purpose:** Disk predictive failure and degraded I/O (the only early warning before `diskFailed` fires), high-frequency ECC memory errors (node-dying signal, distinct from correctable ECC in `eccSummary`), IO card degraded, and SAS cable failure. All other hardware warning traps (fan, PSU, temperature, shelf) are excluded — their critical escalation traps are already in the Primary rule.

**Query:**
```
log.source = "snmptraps"
AND snmp_netapp_Exclude_parameter = "false"
AND (
  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.6"    OR  /* dhmNoticeDegradedIO - Disk degraded I/O — only early signal, no guaranteed escalation trap */
  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.7"    OR  /* dhmNoticePFAEvent   - Predictive disk failure — schedule replacement before diskFailed */
  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.195"  OR  /* eccMasked           - ECC error rate so high BIOS masking them — node at risk */
  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.525"  OR  /* ioCardFailed        - IO card degraded — connectivity impact, no higher trap exists */
  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.1045"     /* sasConnectorWarn    - SAS cable removed/failed — shelf at risk, no escalation trap */
)
```

---

#### Exception Rule B — Cluster, HA, Network & SAN

**Rule name:** `NetApp_SNMP_ClusterNetworkSAN`  
**Event type:** `CUSTOM_ALERT` | **Davis merge:** `false`  
**Purpose:** HA takeover complete (giveback must be triggered manually), all NICs down (complete network loss), FC adapter failed to come online (stays offline silently), Flash Cache card failure, and vFiler stopped (7-Mode environments). Excluded: CPU busy (performance planning not ops), primary NIC down (backup holds, `vifAllLinksFailed` covers total loss), CIFS exhaustion (noisy, hard to action), HBA offline (duplicate risk with `fcOfflineError`), cache offline (follows card error or intentional), QoS maxed (self-resolves).

**Query:**
```
log.source = "snmptraps"
AND snmp_netapp_Exclude_parameter = "false"
AND (
  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.75"  OR  /* clusterNodeTakenOver - HA takeover complete — storage team must trigger giveback */
  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.238" OR  /* vifAllLinksFailed    - ALL NICs down — complete network loss, no escalation trap */
  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.245" OR  /* vfStopped            - vFiler stopped — 7-Mode tenant services down */
  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.595" OR  /* fcOnlineFailWarning  - FC adapter failed to come online — stays offline without intervention */
  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.676"     /* extcacheCardError    - Flash Cache / PAM card failure — performance impact, no higher trap */
)
```

---

#### Exception Rule C — Volume, Capacity & Data Protection

**Rule name:** `NetApp_SNMP_VolumeCapacityDataProtection`  
**Event type:** `CUSTOM_ALERT` | **Davis merge:** `false`  
**Purpose:** Volume near-full at 95% (meaningful advance notice — time to grow/move data before `volumeFull` fires at 98%), UPS battery warning (call facilities before `upsBatteryCritical` fires), directory full (writes failing now, no escalation trap), and SM-S (SnapMirror Synchronous) CG relationship out-of-sync (RPO=0 violated, no escalation trap). Excluded: audit log nearly full (compliance team), quota exceeded (user management — fires for soft, hard, and threshold quotas; not a storage ops action), root volume conflict (auto-resolves at boot), backup snapshot limits (backup team), SnapVault snapshot limit (backup team), WAFL reserve grew (covered by `volumePhysicalOverallocated` in Primary).

**Query:**
```
log.source = "snmptraps"
AND snmp_netapp_Exclude_parameter = "false"
AND (
  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.85"  OR  /* volumeNearlyFull  - Volume >95% — advance notice before volumeFull at 98% */
  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.145" OR  /* upsBatteryWarning - UPS battery getting critical — act before upsBatteryCritical */
  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.187" OR  /* waflDirFull       - Directory full — writes failing now, no escalation trap */
  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.905"     /* smSyncOutOfSyncWarn - SM-S CG relationship out of sync — RPO=0 violated, no escalation trap */
)
```

---

#### Exception Rule D — Security

**Rule name:** `NetApp_SNMP_AVSecurityCompliance`  
**Event type:** `CUSTOM_ALERT` | **Davis merge:** `false`  
**Purpose:** Two traps only — AV scanner disconnected (files going unscanned, security gap the storage team can immediately remediate) and AV service disabled cluster-wide (active security posture change). All AV license/engine expiry events excluded (procurement team, not storage ops). AutoSupport misconfiguration excluded (admin/change item). DC disconnect and password sync excluded (AD infrastructure team).

**Query:**
```
log.source = "snmptraps"
AND snmp_netapp_Exclude_parameter = "false"
AND (
  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.255" OR  /* vscanDisConnection - AV scanner disconnected — files unscanned, storage team re-connects */
  snmp.trap_oid = "SNMPv2-SMI::enterprises.789.0.805"     /* avDisable          - AV service disabled cluster-wide — immediate security gap */
)
```

---

## Severity Filtering Strategy

The severity filter `Severity IN (1,2,3,4)` is the primary signal-to-noise mechanism. Here's the operational rationale for each decision:

| Severity | Label | Include? | Rationale |
|---|---|---|---|
| 1 | Emergency | ✅ | System failure / impending shutdown. Always page. |
| 2 | Alert | ✅ | Immediate corrective action required (disk failure, power failure). Always alert. |
| 3 | Critical | ✅ | Hard device error. Creates an incident. |
| 4 | Error | ✅ | Configuration or operational error. Creates an incident. |
| 5 | Warning | ❌ default | Often transient. Very high volume. Can be enabled per-customer if operationally justified. |
| 6 | Notification | ❌ default | Informational state changes. No action needed. |
| 7 | Information | ❌ default | High volume. Useful in debugging sessions, not in incident management. |
| 8 | Debug | ❌ default | Never for production alerting. |

**Enabling lower severities (if needed):**  
Severities 5–8 are intentionally suppressed but not permanently blocked. To enable them:
1. Modify the event extraction rule query to add `OR Severity = "5"` (etc.)
2. Or create a separate, scoped event rule for a specific customer/host group
3. Deploy via configuration management — do not make UI changes directly in managed environments

---

## Noise Exclusion Strategy

The exclusion rule filters trap messages containing these substrings from `enterprises.789.1.1.12.0`:

| Excluded Pattern | What it represents | Why excluded |
|---|---|---|
| `security.invalid.login` | Failed authentication attempt | High volume, not a storage health event — handled by security tooling |
| `mgmtgwd.certificate.expired` | Expired management certificate | Certificate renewal is a change management item, not an operational alert |
| `Nblade.cifsShrConnectFailed` | CIFS share connection failure | Usually a client-side or network issue, not a storage fault |
| `Nblade.noSMBVerNegotiated` | SMB version negotiation failure | Legacy client compatibility issue — not an incident |
| `secd.rpc.authRequest.blocked` | Security daemon auth block | Security/AD infrastructure issue, not storage |
| `secd.cifsAuth.problem` | CIFS authentication problem | AD/Kerberos issue — not actionable from storage side |
| `wafl.dir.surprair.filename` | WAFL directory filename issue | Transient filesystem event — rarely actionable |
| `quota.parse.error` | Quota configuration parse error | Configuration issue requiring admin review, not ops incident |
| `dns.server.timed.out` | DNS timeout from storage node | Network/DNS infrastructure issue — not storage health |
| `sshd.loginGraceTime.expired` | SSH login grace period expired | Security audit event — not operational |

**All of the above are Severity 4 (Error) or below.** Without exclusion, they would pass the severity filter and generate incidents. This exclusion layer prevents ticket storms from authentication noise, network transients, and configuration drift — events that are real but non-actionable from a storage operations standpoint.

**Extending the exclusion list:**  
The exclusion rule is designed to grow. As new noise patterns are identified operationally, add them to the `FIELDS_ADD` block:
```
CONTAINS(Exclude_parameter, "your.new.pattern.here") OR
```

---

## DataFabric Manager Trap Support

Older environments may still run **NetApp DataFabric Manager (DFM)** — a legacy management layer on top of ONTAP. DFM sends its own traps via different OID sub-trees. Two additional rules cover these:

### DFM 6.x (OnCommand Unified Manager)

**Rule:** `SNMP_NetApp_DataFabric6.x_alertTrap`  
**Status:** Disabled by default (enable only if DFM 6.x is present)  
**Query:** Matches `enterprises.789.5.1.2.2 = "5"` OR `"4"` (severity 4/5 in DFM schema)

**Event title template:**
```
EventName={789.5.1.2.1}
EventMessage={789.5.1.2.5}
EventMessageDetails={789.5.1.2.6}
EventSourceFullName={789.5.1.2.8}
EventSourceScopedFullName={789.5.1.2.12}
```

### DFM 5.x

**Rule:** `SNMP_NetApp_DataFabric5.x_alertTrap`  
**Status:** Disabled by default (enable only if DFM 5.x is present)  
**Query:** Matches `enterprises.789.3.11.1.3 = "5"` OR `"4"` OR `"6"`

> **Note:** These rules are disabled by default because DataFabric Manager is end-of-life. They exist as a compatibility layer for environments that have not yet fully migrated to Cluster-mode ONTAP with direct trap generation.

---

## ServiceNow CI Mapping

Incident routing from Dynatrace to ServiceNow requires a clean CI record for each NetApp system.

### Primary CI

Each NetApp cluster or node should have a CI in the **Mass Storage Device** class in ServiceNow. The CI's primary identifier should be the physical **serial number** (`productSerialNum` / `enterprises.789.1.1.9.0`). This is the master record.

### Multi-IP Challenge

NetApp controllers can send SNMP traps from multiple IP addresses — for example, the cluster management LIF and individual node management LIFs may be different IPs. This creates a routing problem: Dynatrace sees the trap from IP `x.x.x.x`, but ServiceNow's CI incident identification may only know the cluster management IP.

**Solution — Out-of-Band CI Records:**  
Create supplementary CI records in ServiceNow for each additional IP address a NetApp controller might send traps from. These out-of-band CIs contain:
- The IP address as the primary identifier
- Minimal additional data
- The correct Assignment Group

This ensures that regardless of which IP the trap arrives from, the incident routes to the correct team while the full asset management remains on the primary serial-number-based CI.

---

## Troubleshooting

### No traps appearing in Dynatrace

1. Verify the NetApp cluster has the ActiveGate IP as a traphost: `system snmp traphost show`
2. Verify SNMP is enabled on the cluster: `system snmp show`
3. Check the ActiveGate SNMP trap extension is running and configured to listen on UDP 162
4. Run `system snmp test` on the NetApp — check if a log record appears in Dynatrace under `log.source = "snmptraps"`

### Traps arriving but Severity field missing

Processing rule `SNMP_NetApp_Processing-netapp-severity` is not deployed or not enabled. Deploy it and verify it's active. The `Severity` field will only appear on logs ingested after the rule is enabled — it does not backfill.

### High-severity traps not creating events

Check:
1. Is `Severity` field populated? (see above)
2. Is `snmp_netapp_Exclude_parameter = "false"`? If the field is missing entirely, the processing rule isn't running.
3. Is the event extraction rule enabled?
4. Is the `snmpv2-mib::snmptrapoid` field present? The severity rule depends on this field name (lowercase). Different SNMP agents may produce this field with different capitalisation.

### Too many incidents / ticket storm

The exclusion list may need expansion. Identify the noisy pattern:
```sql
fetch logs
| filter log.source == "snmptraps"
| filter matchesPhrase(snmp.trap_oid, "enterprises.789.0")
| filter Severity == "4" or Severity == "3"
| fields `SNMPv2-SMI::enterprises.789.1.1.12.0`
| summarize count(), by: `SNMPv2-SMI::enterprises.789.1.1.12.0`
| sort count() desc
| limit 20
```

Identify high-volume, non-actionable patterns and add them to the exclusion rule.

### Incidents routing to wrong team in ServiceNow

Check out-of-band CI records. The trap may be arriving from an IP not present in ServiceNow. Create an out-of-band CI for that IP mapped to the correct Assignment Group.

---

## Key OID Quick Reference

The most critical OIDs for observability rule configuration. For the complete trap catalogue see the Built-in ONTAP Traps section above.

**Generic Severity Traps — the alerting backbone**

| Full OID | Name | Severity | Alerted? |
|---|---|---|---|
| `1.3.6.1.4.1.789.0.11` | `emergencyTrap` | 1 — Emergency | ✅ |
| `1.3.6.1.4.1.789.0.12` | `alertTrap` | 2 — Alert | ✅ |
| `1.3.6.1.4.1.789.0.13` | `criticalTrap` | 3 — Critical | ✅ |
| `1.3.6.1.4.1.789.0.14` | `errorTrap` | 4 — Error | ✅ |
| `1.3.6.1.4.1.789.0.15` | `warningTrap` | 5 — Warning | ❌ suppressed |
| `1.3.6.1.4.1.789.0.16` | `notificationTrap` | 6 — Notification | ❌ suppressed |
| `1.3.6.1.4.1.789.0.17` | `informationalTrap` | 7 — Informational | ❌ suppressed |
| `1.3.6.1.4.1.789.0.18` | `dbgTrap` | 8 — Debug | ❌ suppressed |

**Key data OIDs carried in every trap**

| Full OID | Field Name | Purpose |
|---|---|---|
| `1.3.6.1.4.1.789.1.1.9.0` | `productSerialNum` | Node serial number — use as CI correlation key |
| `1.3.6.1.4.1.789.1.1.12.0` | `productTrapData` | Human-readable event message — parse for context and exclusions |

**OID tree roots**

| OID | Description |
|---|---|
| `1.3.6.1.4.1.789` | NetApp enterprise root |
| `1.3.6.1.4.1.789.0` | All trap definitions |
| `1.3.6.1.4.1.789.1` | MIB data objects (polled metrics) |
| `1.3.6.1.4.1.789.3` | DataFabric Manager 5.x trap tree |
| `1.3.6.1.4.1.789.5` | DataFabric Manager 6.x / OnCommand trap tree |

---

## ITSM Integration — Incident Lifecycle Design

### The Problem: `dt.event.timeout` and Phantom Auto-Closure

Every Dynatrace log-based event carries a `dt.event.timeout` property (default **15 minutes**, maximum **360 minutes**). This timeout controls when Dynatrace auto-closes the problem if no new event fires to refresh it. For metric-based alerting this works correctly — when the metric recovers below threshold, Dynatrace closes the problem, and the ITSM integration propagates the closure to ServiceNow, resolving the incident.

**SNMP traps do not work this way.** A trap fires once when a fault condition is detected. If the fault persists — a failed disk, a degraded shelf, an HA node waiting for giveback — ONTAP does not keep re-sending the trap. It fired once; the problem is still there. The result:

```
T+0       Trap fires  →  Dynatrace problem created  →  ServiceNow incident P2 opened
T+15min   dt.event.timeout expires
          Dynatrace auto-closes the problem  →  ServiceNow incident auto-resolved ✗
T+?       Fault still present. No one is working it.
```

The incident closes with the fault unresolved. The storage engineer sees a resolved ticket. The disk is still failed.

### The Solution: Two Configuration Changes

#### 1. Dedicated Alert Profile for SNMP Traps

Create a separate Dynatrace Alert Profile scoped exclusively to the NetApp SNMP trap event rules (and any other log/trap-based event sources). Do **not** mix this with metric-based alert profiles. Keep `dt.event.timeout` at the **default 15 minutes** — do not increase it. Increasing it causes its own problems: stale open problems in Dynatrace, Davis AI correlation noise, and hitting environment event limits. The timeout value is irrelevant to the ITSM problem once the fix in Step 2 is applied.

**Settings → Alerting → Alert profiles → New profile:**

| Setting | Value |
|---|---|
| Profile name | `SNMP Traps — Storage` |
| Scope | Filter to event titles matching `NetApp_SNMP_*` |
| Severity rules | Include: `Custom alert` |
| `dt.event.timeout` | Leave at default **15 minutes** — do not change |

#### 2. Suppress Problem-Close Notifications in the ServiceNow Integration Rule

In the Dynatrace ServiceNow integration (or whichever ITSM connector is in use), each Problem Notification is linked to an Alert Profile. The notification fires for three lifecycle events: problem opened, problem updated, and problem closed. **For the trap-scoped alert profile, uncheck the problem-close notification.** This is the actual fix.

**Settings → Integration → Problem notifications → [your ServiceNow integration]:**

| Trigger | Metric-based alert profile | SNMP Trap alert profile |
|---|---|---|
| Problem opened | ✅ Send | ✅ Send |
| Problem updated | ✅ Send | ✅ Send |
| Problem closed | ✅ Send | **❌ Uncheck — do NOT send** |

With problem-close suppressed: the Dynatrace problem quietly times out after 15 minutes (no re-firing trap = no refresh), but ServiceNow **never receives the closure notification**. The incident stays open until a human operator investigates, resolves the fault, and manually closes the ticket.

### Operational Model

```
SNMP Trap fires
      │
      ▼
Dynatrace Problem created
      │
      ├──► ServiceNow Incident opened (P2/P3 based on severity)
      │         │
      │         │   ← operator picks up ticket
      │         │   ← investigates storage fault
      │         │   ← resolves hardware issue
      │         │   ← manually closes ServiceNow incident
      │         ▼
      │    ServiceNow Incident CLOSED ✓ (human-driven)
      │
      └──► Dynatrace Problem times out after 15 min (default dt.event.timeout)
           No closure notification sent to ServiceNow — incident stays open
```

This is the correct model for any **discrete, fire-once signal** — SNMP traps, syslog alerts, webhook notifications. It differs from metric-based alerting where automatic closure is safe and expected.

### Contrast: Metric-Based vs Trap-Based Lifecycle

| | Metric-based alerts | SNMP trap alerts |
|---|---|---|
| Signal type | Continuous (value above/below threshold) | Discrete (fires once on event) |
| Auto-recovery signal | Yes — metric drops below threshold | No — trap does not re-fire on recovery |
| Dynatrace auto-close | Correct — metric confirms recovery | **Wrong** — timeout ≠ fault resolved |
| ServiceNow auto-close | ✅ Appropriate | **❌ Must be suppressed** |
| Incident closure | System-driven | Operator-driven |

### Future Work

> 📋 **Planned:** A dedicated companion document covering Dynatrace–ServiceNow / ITSM integration best practices will be published in this repository. It will cover: alert profile design patterns, multi-tier severity routing, bidirectional sync configuration, assignment group mapping from CMDB CI, and handling of mixed metric+trap problem cards. Until that document is available, the configuration described in this section represents the recommended baseline.

---

## Best Practices

**For Storage Engineers:**

- Use SNMPv3 on ONTAP 9.1+. Never use SNMPv1 in production.
- Configure traphosts at the cluster level, not just per-node. This ensures HA failover events are still reported.
- Ensure the management LIF and all node management IPs are known to your observability platform — a trap arriving from an unregistered IP will fail to route correctly to its CI.
- Test trap delivery after every ONTAP upgrade. OID structures are stable but verify your traphost configuration survives cluster upgrades.
- Keep the `productSerialNum` field in your CMDB/ServiceNow as the canonical CI identifier. IPs change; serial numbers don't.

**For Observability / Platform Engineers:**

- **Always use processing rules as a pre-filter layer** when dealing with high-volume SNMP trap estates. The pattern here (derive a computed field, then filter on it in the event rule) is reusable for any vendor's SNMP traps.
- **Invert your allow-list thinking.** Instead of maintaining a list of OIDs to include, process everything and exclude the noise. This is more resilient to vendor firmware updates that introduce new trap types.
- **Never set `davisMerge = true` for storage trap events.** Storage hardware events are discrete facts, not symptoms to be correlated with application performance anomalies.
- **Use configuration-as-code (Monaco or equivalent)** for all rule deployments — never make UI changes directly in managed environments. This ensures repeatability and auditability across customer estates.
- **Build a noise analysis query** (see Troubleshooting section) and run it periodically. The exclusion list should grow as operational experience accumulates.
- **Do NOT rely on `dt.event.timeout` for ITSM auto-closure on trap-based incidents.** See the [ITSM Integration — Incident Lifecycle Design](#itsm-integration--incident-lifecycle-design) section below for the correct approach.
- **`productSerialNum` is your entity correlation key.** When enriching your SNMP entity in Dynatrace, map the serial number to the ServiceNow CI's `serial_number` field for automated CI identification.

---

## References

### Companion Documents in This Repository

| File | Description |
|---|---|
| [`NetApp_ONTAP_BuiltinTraps_Reference.md`](./NetApp_ONTAP_BuiltinTraps_Reference.md) | Complete reference table of all 258 built-in ONTAP traps from MIB v2.20, with per-trap alert status (✅/❌) and the specific Dynatrace rule that catches each one |
| `netapp.mib` | NetApp ONTAP MIB v2.20, July 2022 — authoritative source for all OID definitions used in this implementation |

### External References

- [ONTAP SNMP Configuration Overview](https://docs.netapp.com/us-en/ontap/networking/commands_for_managing_snmp.html)
- [How to Configure SNMP Monitoring on ONTAP C-Mode](https://kb.netapp.com/onprem/ontap/os/How_to_configure_SNMP_monitoring_on_DATA_ONTAP)
- [Configure Trap Hosts to Receive SNMP Notifications](https://docs.netapp.com/us-en/ontap/networking/configure_traphosts_to_receive_snmp_notifications.html)
- [SNMP Support in Data ONTAP — TR-4220](https://www.netapp.com/pdf.html?item=/media/16417-tr-4220pdf.pdf)
- [ONTAP Event Severity Levels](https://library.netapp.com/ecmdocs/ECMP1222473/html/GUID-CA1C776E-4DF3-40EC-BDC7-2D741186772B.html)
- [ONTAP MIB Download (NetApp Support login required)](https://mysupport.netapp.com/site/tools/tool-eula/ontap-mibs)
- [Dynatrace Log Processing Rules](https://www.dynatrace.com/support/help/observe-and-explore/logs/log-processing)
