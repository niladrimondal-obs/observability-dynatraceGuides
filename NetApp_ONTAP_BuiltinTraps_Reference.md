# NetApp ONTAP Built-in Trap Reference

> **Source:** NetApp ONTAP MIB v2.20 (July 2022)
>
> **Platform:** Dynatrace with SNMP ActiveGate integration
>
> **Total traps:** 258 &nbsp;|&nbsp; **Alerted ✓:** 132 &nbsp;|&nbsp; **Suppressed ✗:** 126

---

## Alert Coverage

**132** traps generate a Dynatrace alert. **126** are suppressed — recovery states, informational events, warnings with a guaranteed critical escalation trap already covered, or events outside the storage team's operational scope.

### Rule Legend

| Rule | Dynatrace Rule Name | Scope |
|---|---|---|
| **Primary** | `NetApp_SNMP_HighSeverity_Traps` | Generic severity traps 789.0.11–18 at Severity 1–4, plus specific built-in traps whose OID last-digit is 1–4 |
| **Exception A** | `NetApp_SNMP_HardwareWarnings` | Disk PFA / degraded-IO · ECC high-frequency · IO card failure · SAS cable |
| **Exception B** | `NetApp_SNMP_ClusterNetworkSAN` | HA takeover · all-NICs-down · vFiler stopped · FC failed-to-online · Flash Cache failure |
| **Exception C** | `NetApp_SNMP_VolumeCapacityDataProtection` | Volume near-full (95%) · UPS battery warning · directory full · SnapMirror out-of-sync |
| **Exception D** | `NetApp_SNMP_AVSecurityCompliance` | AV scanner disconnected · AV service disabled cluster-wide |

---

## Trap Reference

### Generic Severity

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.2` | `userDefined` | Polling-style trap via 'snmp traps'. Last digit of OID encodes severity (1=Emergency…8=Debug). | ✓ | Primary |
| `789.0.11` | `emergencyTrap` | Extremely urgent situation requiring immediate action. Severity 1. | ✓ | Primary |
| `789.0.12` | `alertTrap` | Condition needing immediate correction. Severity 2. | ✓ | Primary |
| `789.0.13` | `criticalTrap` | Critical condition such as hard device error. Severity 3. | ✓ | Primary |
| `789.0.14` | `errorTrap` | Error condition such as configuration mistake. Severity 4. | ✓ | Primary |
| `789.0.15` | `warningTrap` | Warning condition. Severity 5 — suppressed. | ✗ | Primary |
| `789.0.16` | `notificationTrap` | Notification event. Severity 6 — suppressed. | ✗ | Primary |
| `789.0.17` | `informationalTrap` | Informational message. Severity 7 — suppressed. | ✗ | Primary |
| `789.0.18` | `dbgTrap` | Debug trap. Severity 8 — suppressed. | ✗ | Primary |

### Disk

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.6` | `dhmNoticeDegradedIO` | Disk Health Monitor — degraded I/O event. Only early signal; no guaranteed escalation trap. | ✓ | Exception A |
| `789.0.7` | `dhmNoticePFAEvent` | Disk Health Monitor — predictive failure. Schedule proactive replacement before diskFailed fires. | ✓ | Exception A |
| `789.0.21` | `diskFailedShutdown` | System shutting down — degraded mode for 24 hours due to disk failure. | ✓ | Primary |
| `789.0.22` | `diskFailed` | One or more disks failed. | ✓ | Primary |
| `789.0.26` | `diskRepaired` | Failed disks repaired. Placeholder — not currently sent by ONTAP. | ✗ | — |
| `789.0.562` | `diskMultipathOneSwitch` | Multipathed disk only connected to one switch. | ✓ | Primary |
| `789.0.563` | `diskMultipathNoTakeover` | Multipath disks/LUNs not detected for partner — HA takeover not possible. | ✓ | Primary |
| `789.0.565` | `diskMultipathWarning` | Sync mirroring on but disks not multipathed. Dropped — diskMultipathNoTakeover (Primary) covers actual takeover failure. | ✗ | — |
| `789.0.574` | `driveDisableErr` | Drive disabled by shelf module due to hardware errors. | ✓ | Primary |
| `789.0.1002` | `diskFailedRemoveCarrier` | All disks in a multi-disk carrier failed — carrier must be removed. | ✓ | Primary |

### Fan

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.31` | `fanFailureShutdown` | Critical fans failed — system shutting down. | ✓ | Primary |
| `789.0.33` | `fanFailed` | One or more chassis fans failed. | ✓ | Primary |
| `789.0.35` | `fanWarning` | Fan warning. Dropped — fanFailed and fanFailureShutdown already cover escalation. | ✗ | — |
| `789.0.36` | `fanRepaired` | All fans repaired. | ✗ | — |

### Power Supply

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.41` | `powerSupplyFailureShutdown` | Critical power supplies failed — system shutting down. | ✓ | Primary |
| `789.0.43` | `powerSupplyFailed` | One or more redundant power supplies failed. | ✓ | Primary |
| `789.0.45` | `powerSupplyWarning` | PSU warning. Dropped — powerSupplyFailed and powerSupplyFailureShutdown cover escalation. | ✗ | — |
| `789.0.46` | `powerSupplyRepaired` | Power supplies repaired. | ✗ | — |
| `789.0.521` | `powerSupplyFanFailxMinShutdown` | Multiple PSU fans failed — system will shut down in minutes. | ✓ | Primary |

### CPU

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.55` | `cpuTooBusy` | CPU >90%. Dropped — not enabled by default; performance planning item, not ops alert. | ✗ | — |
| `789.0.56` | `cpuOk` | CPU utilisation dropped below 90%. | ✗ | — |

### NVRAM

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.62` | `nvramBatteryDischarged` | NVRAM battery fully discharged. | ✓ | Primary |
| `789.0.63` | `nvramBatteryLow` | NVRAM battery charge low. | ✓ | Primary |

### HA Cluster

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.72` | `clusterNodeFailed` | HA node failure — partner assumes service. | ✓ | Primary |
| `789.0.75` | `clusterNodeTakenOver` | HA takeover complete. Storage team must trigger giveback. | ✓ | Exception B |
| `789.0.76` | `clusterNodeRepaired` | Cluster node resumed operation. | ✗ | — |
| `789.0.77` | `clusterNodeRepairing` | Node rebooted — giveback started. | ✗ | — |
| `789.0.741` | `scsibladeOutOfQuorum` | Node lost cluster connectivity — FCP and iSCSI halted. | ✓ | Primary |
| `789.0.746` | `scsibladeInQuorum` | Node re-established cluster connectivity. | ✗ | — |
| `789.0.767` | `sfoAggregateRelocated` | SFO aggregate permanently relocated to another node. | ✗ | — |
| `789.0.973` | `clusterLinkDown` | Cluster interconnect link lost. | ✓ | Primary |
| `789.0.983` | `clusterL2ConnFail` | L2 ping failing between cluster ports. | ✓ | Primary |
| `789.0.993` | `clusterPingDropLarge` | Large packet pings failing between cluster ports. | ✓ | Primary |

### Volume & Capacity

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.82` | `volumeFull` | Volume >98% full. | ✓ | Primary |
| `789.0.85` | `volumeNearlyFull` | Volume >95% full. Advance notice — time to act before volumeFull fires at 98%. | ✓ | Exception C |
| `789.0.86` | `volumeRepaired` | All volumes back under 95%. | ✗ | — |
| `789.0.87` | `volumesStillFull` | Some volumes recovered but others still full. Dropped — volumeFull keeps firing while condition persists. | ✗ | — |
| `789.0.187` | `waflDirFull` | Directory filled to limit — write failures occurring now. No escalation trap. | ✓ | Exception C |
| `789.0.272` | `volumeRestrictedByMirrorBigIo` | Volume restricted due to medium error during reconstruction. | ✓ | Primary |
| `789.0.274` | `volumeInconsistentUmount` | Volume unmounted due to inconsistency. | ✓ | Primary |
| `789.0.275` | `volumeStateChanged` | Volume going offline/restricted. Dropped — volumeOffline and volumeRestricted (Primary) fire on completion. | ✗ | — |
| `789.0.276` | `volumeOnline` | Volume is online. | ✗ | — |
| `789.0.294` | `volumeRemoteUnreachable` | Error communicating with remote volume. | ✓ | Primary |
| `789.0.296` | `volumeRemoteOk` | Communication with remote volume restored. | ✗ | — |
| `789.0.297` | `volumeRemoteRestored` | Data on remote volume fully restored. | ✗ | — |
| `789.0.298` | `volumeRemoteRestoreBegin` | Restore-on-Demand restoration started. | ✗ | — |
| `789.0.304` | `volumeRestrictedRootConflict` | Volume restricted due to root volume conflict. | ✓ | Primary |
| `789.0.314` | `volumeOfflineTooBig` | Volume cannot come online — raw size exceeds maximum allowed. | ✓ | Primary |
| `789.0.324` | `volumeOffline` | Volume being taken offline. | ✓ | Primary |
| `789.0.334` | `volumeRestricted` | Volume being restricted. | ✓ | Primary |
| `789.0.344` | `volumeDegradedDirty` | Volume degraded with dirty parity — WAFL check required. | ✓ | Primary |
| `789.0.354` | `volumeError` | Volume cannot come online due to error. | ✓ | Primary |
| `789.0.356` | `volumeSelectedRootConflict` | Multiple root volumes at boot — auto-selects one. Dropped — rare, auto-resolves. | ✗ | — |
| `789.0.482` | `maxDirSizeAlert` | Directory reached maxdirsize limit. | ✓ | Primary |
| `789.0.485` | `maxDirSizeWarning` | Directory approaching maxdirsize. Dropped — maxDirSizeAlert (Primary) covers reaching the limit. | ✗ | — |
| `789.0.646` | `volumeCloneCreate` | Volume clone created. | ✗ | — |
| `789.0.666` | `volumeAutogrow` | Volume auto-grown. | ✗ | — |
| `789.0.873` | `volumeLogicalOverallocated` | Volume over-allocated — may not honour file reservations. | ✓ | Primary |
| `789.0.875` | `volumeReserveGrew` | WAFL reserve grew. Dropped — volumePhysicalOverallocated (Primary) covers the critical state. | ✗ | — |
| `789.0.882` | `volumePhysicalOverallocated` | Volume/aggregate dangerously low on free blocks. | ✓ | Primary |

### Temperature

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.91` | `overTempShutdown` | System temperature critically high — shutting down. | ✓ | Primary |
| `789.0.95` | `overTemp` | System temperature warning. Dropped — overTempShutdown (Primary) covers escalation. | ✗ | — |
| `789.0.96` | `overTempRepaired` | System temperature returned to normal. | ✗ | — |
| `789.0.371` | `chassisTemperatureShutdown` | Chassis temperature extreme — shutdown initiated. | ✓ | Primary |
| `789.0.372` | `chassisTemperatureWarning` | Chassis temperature too high or too low. | ✓ | Primary |
| `789.0.375` | `chassisTemperatureUnknown` | Chassis temperature sensor unreadable. Dropped — no remote action possible. | ✗ | — |
| `789.0.376` | `chassisTemperatureOk` | Chassis temperature OK. | ✗ | — |

### Disk Shelf & Enclosure

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.103` | `shelfFault` | Disk shelf fault — drive, fans, power or temperature. | ✓ | Primary |
| `789.0.105` | `shelfWarning` | Disk shelf warning. Dropped — shelfFault (Primary) covers escalation. | ✗ | — |
| `789.0.106` | `shelfRepaired` | Previously-reported shelf fault corrected. | ✗ | — |
| `789.0.464` | `shelfSESElectronicsFailed` | Enclosure services device failed. | ✓ | Primary |
| `789.0.465` | `shelfSESElectronicsWarning` | SES electronics warning. Dropped — shelfSESElectronicsFailed (Primary) covers escalation. | ✗ | — |
| `789.0.467` | `shelfSESElectronicsInfo` | Previously-reported SES electronics failure corrected. | ✗ | — |
| `789.0.473` | `shelfIFModuleFailed` | Storage interface module in disk shelf failed. | ✓ | Primary |
| `789.0.475` | `shelfIFModuleWarning` | Shelf IF module warning. Dropped — shelfIFModuleFailed (Primary) covers escalation. | ✗ | — |
| `789.0.477` | `shelfIFModuleInfo` | Previously-reported shelf IF module failure corrected. | ✗ | — |
| `789.0.1045` | `sasConnectorWarn` | SAS cable removed or failed. No escalation trap — this is the only signal. | ✓ | Exception A |
| `789.0.1047` | `sasConnectorInfo` | Previously-reported SAS cable error corrected. | ✗ | — |
| `789.0.1054` | `sesShelfPowerSupplyError` | Shelf power supply failed or removed. | ✓ | Primary |
| `789.0.1055` | `sesShelfPowerSupplyWarn` | Shelf PSU warning. Dropped — sesShelfPowerSupplyError (Primary) covers escalation. | ✗ | — |
| `789.0.1057` | `sesShelfPowerSupplyInfo` | Shelf power supply error corrected. | ✗ | — |
| `789.0.1064` | `volRehostFailed` | Volume rehost failed — available under source Vserver. | ✓ | Primary |
| `789.0.1065` | `sesShelfVoltageWarning` | Shelf voltage warning. Dropped — no remote fix; shelf failure traps cover escalation. | ✗ | — |
| `789.0.1067` | `volRehostSucceeded` | Volume rehost successful. | ✗ | — |
| `789.0.1164` | `sesShelfDcmError` | Shelf DCM not installed or not supported. | ✓ | Primary |
| `789.0.1165` | `sesShelfDcmWarn` | Shelf DCM warning. Dropped — sesShelfDcmError (Primary) covers escalation. | ✗ | — |
| `789.0.1167` | `sesShelfDcmInfo` | Shelf DCM error corrected. | ✗ | — |

### Chassis

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.381` | `chassisCPUFanStopped` | One or more CPU fans stopped — shutdown initiated. | ✓ | Primary |
| `789.0.383` | `chassisCPUFanSlow` | CPU fan spinning too slowly. | ✓ | Primary |
| `789.0.386` | `chassisCPUFanOk` | All CPU fans functioning properly. | ✗ | — |
| `789.0.391` | `chassisPowerSuppliesFailed` | Multiple chassis power supplies failed. | ✓ | Primary |
| `789.0.392` | `chassisPowerSupplyDegraded` | One or more chassis power supplies degraded. | ✓ | Primary |
| `789.0.393` | `chassisPowerSupplyFailed` | One chassis power supply failed. | ✓ | Primary |
| `789.0.394` | `chassisPowerSupplyRemoved` | One or more chassis power supplies removed. | ✓ | Primary |
| `789.0.395` | `chassisPowerSupplyOff` | PSU off. Dropped — intentional or fault; PSU failure traps already in Primary. | ✗ | — |
| `789.0.396` | `chassisPowerSuppliesOk` | All chassis power supplies functioning properly. | ✗ | — |
| `789.0.397` | `chassisPowerSupplyOk` | This chassis power supply functioning properly. | ✗ | — |
| `789.0.403` | `chassisPowerDegraded` | Chassis power degraded. | ✓ | Primary |
| `789.0.406` | `chassisPowerOk` | Chassis power functioning properly. | ✗ | — |
| `789.0.412` | `chassisFanDegraded` | A chassis fan degraded. | ✓ | Primary |
| `789.0.413` | `chassisFanRemoved` | A chassis fan FRU removed. | ✓ | Primary |
| `789.0.414` | `chassisFanStopped` | One or more chassis fans stopped. | ✓ | Primary |
| `789.0.415` | `chassisFanWarning` | Chassis fan warning. Dropped — chassisFanStopped and chassisFanDegraded cover escalation. | ✗ | — |
| `789.0.416` | `chassisFanOk` | All chassis fans functioning properly. | ✗ | — |
| `789.0.501` | `chassisPSRemovedxMinShutdown` | PSU removed — system shuts down in minutes if not reinserted. | ✓ | Primary |
| `789.0.505` | `chassisPSUsMismatch` | PSU type mismatch. Dropped — post-maintenance config defect; change ticket not incident. | ✗ | — |
| `789.0.511` | `chassisFanFailxMinShutdown` | Multiple chassis fan failure — shutdown in minutes. | ✓ | Primary |
| `789.0.515` | `chassisPSUwrongInput` | PSU on wrong power source. Dropped — physical install defect; on-site fix only. | ✗ | — |

### Global Status

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.111` | `globalStatusNonRecoverable` | Appliance status non-recoverable — shutdown initiated. | ✓ | Primary |
| `789.0.113` | `globalStatusCritical` | Appliance status critical — immediate attention required. | ✓ | Primary |
| `789.0.115` | `globalStatusNonCritical` | Appliance status non-critical. Dropped — globalStatusCritical (Primary) covers escalation. | ✗ | — |
| `789.0.116` | `globalStatusOk` | Appliance status returned to normal. | ✗ | — |

### Quota

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.126` | `softQuotaExceeded` | User exceeded soft quota limit. | ✗ | — |
| `789.0.127` | `softQuotaNormal` | User back under soft quota. | ✗ | — |
| `789.0.176` | `quotaExceeded` | Quota limit exceeded (hard, soft, or threshold) — users may be blocked from writing. Dropped — user quota management; storage ops cannot adjust quotas mid-incident. | ✗ | — |
| `789.0.177` | `quotaNormal` | Quota returned to normal. | ✗ | — |

### AutoSupport

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.134` | `autosupportSendError` | AutoSupport send failed. | ✓ | Primary |
| `789.0.135` | `autosupportConfigurationError` | AutoSupport possibly misconfigured. Dropped — admin/change item; NetApp support flags independently. | ✗ | — |
| `789.0.136` | `autosupportSent` | AutoSupport sent successfully. | ✗ | — |

### UPS

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.142` | `upsLinePowerOff` | UPS: line power failed — on battery. | ✓ | Primary |
| `789.0.143` | `upsBatteryCritical` | UPS: battery nearly exhausted — graceful shutdown starting. | ✓ | Primary |
| `789.0.144` | `upsShuttingDown` | UPS: shutting down — battery exhausted. | ✓ | Primary |
| `789.0.145` | `upsBatteryWarning` | UPS battery time getting critical. Call facilities/power team before upsBatteryCritical fires. | ✓ | Exception C |
| `789.0.146` | `upsLinePowerRestored` | UPS: line power restored. | ✗ | — |

### Application

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.151` | `appEmergency` | Application — extremely urgent. Severity 1. | ✓ | Primary |
| `789.0.152` | `appAlert` | Application — immediate correction needed. Severity 2. | ✓ | Primary |
| `789.0.153` | `appCritical` | Application — critical condition. Severity 3. | ✓ | Primary |
| `789.0.154` | `appError` | Application — error condition. Severity 4. | ✓ | Primary |
| `789.0.155` | `appWarning` | Application — warning. Severity 5 — suppressed. | ✗ | — |
| `789.0.156` | `appNotice` | Application — notification event. | ✗ | — |
| `789.0.157` | `appInfo` | Application — informational message. | ✗ | — |
| `789.0.158` | `appTrap` | Application — debug trap. | ✗ | — |

### Audit Log

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.162` | `alfFilewrap` | Internal audit log wrapped — records are being lost. | ✓ | Primary |
| `789.0.166` | `alfFileSaved` | Audit log auto-saved to external file. | ✗ | — |
| `789.0.167` | `alfFileNearlyFull` | Audit log nearly full. Dropped — compliance/audit team action; storage ops cannot expand mid-flight. | ✗ | — |

### Memory

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.192` | `eccSummary` | Memory ECC — new correctable ECC errors detected. | ✓ | Primary |
| `789.0.195` | `eccMasked` | Memory ECC error rate so high BIOS is masking them — node at risk of failure. | ✓ | Exception A |

### Write & IO

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.424` | `writeVerificationFailed` | Write failed verification test on SnapValidator-enabled volume. | ✓ | Primary |

### FTP

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.204` | `ftpdError` | FTP daemon service stopped. | ✓ | Primary |
| `789.0.206` | `ftpdMaxConnNotice` | FTP connections hit maximum. | ✗ | — |
| `789.0.216` | `ftpdMaxConnThresholdNotice` | FTP connections approaching maximum. | ✗ | — |

### Fibre Channel & SAN

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.222` | `scsitgtFCPLinkBreak` | FCP adapter link break. | ✓ | Primary |
| `789.0.224` | `scsitgtPartnerPathMisconfigured` | FCP partner path misconfigured. | ✓ | Primary |
| `789.0.226` | `scsitgtThrottleNotice` | SCSI target throttle limit event. | ✗ | — |
| `789.0.587` | `hbaOfflineInformation` | HBA port disabled. Dropped — duplicate risk with fcOfflineError; fires for user-initiated actions too. | ✗ | — |
| `789.0.594` | `fcOfflineError` | FibreChannel adapter port gone offline due to error. | ✓ | Primary |
| `789.0.595` | `fcOnlineFailWarning` | FC adapter failed to come online — stays offline without intervention. | ✓ | Exception B |
| `789.0.596` | `fcOffLineNotification` | FC adapter gone offline. Dropped — fcOfflineError (Primary) covers error-caused offline. | ✗ | — |
| `789.0.597` | `fcOnlineInformation` | FibreChannel adapter port came online. | ✗ | — |
| `789.0.1211` | `fcBridgeTempEmergency` | FC-to-SAS bridge reached critical temperature — will shut down. | ✓ | Primary |
| `789.0.1212` | `fcBridgeTempAlert` | FC-to-SAS bridge at elevated temperature — action needed. | ✓ | Primary |
| `789.0.1216` | `fcBridgeTempNotice` | FC-to-SAS bridge returned to normal temperature. | ✗ | — |
| `789.0.1222` | `fcBridgeFcPortAlert` | FC port gone offline. | ✓ | Primary |
| `789.0.1226` | `fcBridgeFcPortNotice` | FC port came back online. | ✗ | — |
| `789.0.1232` | `fcBridgeSASPortAlert` | SAS port gone offline. | ✓ | Primary |
| `789.0.1236` | `fcBridgeSASPortNotice` | SAS port came back online. | ✗ | — |
| `789.0.1242` | `fcBridgeThroughputAlert` | FC-to-SAS bridge experiencing degraded throughput. | ✓ | Primary |
| `789.0.1246` | `fcBridgeThroughputNotice` | FC-to-SAS bridge recovered from degraded throughput. | ✗ | — |
| `789.0.1252` | `fcBridgePowerSupplyAlert` | FC-to-SAS bridge power supply failed. | ✓ | Primary |
| `789.0.1256` | `fcBridgePowerSupplyNotice` | FC-to-SAS bridge power supply returned to normal. | ✗ | — |
| `789.0.1262` | `fcBridgeSasPhyTransitionAlert` | SAS port on bridge transitioned to offline. | ✓ | Primary |
| `789.0.1266` | `fcBridgeSasPhyTransitionNotice` | SAS port on bridge transitioned to online. | ✗ | — |

### Network / VIF

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.237` | `vifPrimaryLinkFailed` | Primary NIC failed. Dropped — backup link holds; vifAllLinksFailed (Exception B) covers total loss. | ✗ | — |
| `789.0.238` | `vifAllLinksFailed` | ALL NICs down — complete network loss. No escalation trap. | ✓ | Exception B |

### vFiler

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.245` | `vfStopped` | vFiler stopped — 7-Mode tenant services down. No escalation trap. | ✓ | Exception B |
| `789.0.246` | `vfStarted` | vFiler started. | ✗ | — |

### Antivirus / VScan

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.254` | `vscanVirusDetectedError` | Virus detected by scanner. | ✓ | Primary |
| `789.0.255` | `vscanDisConnection` | AV scanner dropped connection — files going unscanned. No escalation trap. | ✓ | Exception D |
| `789.0.256` | `vscanConfigurationChange` | VScan configuration changed. | ✗ | — |
| `789.0.257` | `vscanConnection` | Virus scanner connected. | ✗ | — |
| `789.0.266` | `vscanServerUpgrade` | VScan server upgraded. | ✗ | — |
| `789.0.705` | `avTrendLicenseExpired` | Trend AV license expired. Dropped — procurement team action. | ✗ | — |
| `789.0.706` | `avTrendLicenseExpiring` | Trend AV license expiring. Dropped — procurement team action. | ✗ | — |
| `789.0.773` | `avUpdateFailed` | Antivirus software update failed. | ✓ | Primary |
| `789.0.774` | `avLicenseCheckFailed` | Antivirus license validation failed. | ✓ | Primary |
| `789.0.775` | `avMcAfeeProductExpired` | McAfee product expired. Dropped — procurement team action. | ✗ | — |
| `789.0.776` | `avRemedy` | Remedy action taken — file repaired/deleted/quarantined. | ✗ | — |
| `789.0.777` | `avLicenseCheck` | License validation successful. | ✗ | — |
| `789.0.783` | `avRemedyFailure` | Remedy action failed. | ✓ | Primary |
| `789.0.785` | `avMcAfeeEngineExpired` | McAfee engine expired. Dropped — procurement team action. | ✗ | — |
| `789.0.786` | `avMcAfeeProductExpiring` | McAfee product expiring. Dropped — procurement team action. | ✗ | — |
| `789.0.793` | `av2gbFileNotScanned` | File >2GB not scanned — marked clean. | ✓ | Primary |
| `789.0.796` | `avMcAfeeEngineExpiring` | McAfee engine expiring. Dropped — procurement team action. | ✗ | — |
| `789.0.802` | `avVirusfound` | Virus found while scanning. | ✓ | Primary |
| `789.0.803` | `avMcAfeeLicenseFailed` | McAfee license activation failed. | ✓ | Primary |
| `789.0.804` | `avDisableFailed` | Antivirus service disabling failed. | ✓ | Primary |
| `789.0.805` | `avDisable` | AV service disabled cluster-wide — active security gap. No escalation trap. | ✓ | Exception D |
| `789.0.806` | `avMcAfeeLicenseExpiring` | McAfee license expiring. Dropped — procurement team action. | ✗ | — |
| `789.0.807` | `avEnable` | Antivirus service enabled. | ✗ | — |
| `789.0.812` | `avSpywareFound` | Spyware found while scanning. | ✓ | Primary |
| `789.0.813` | `avEnableFailed` | Antivirus service enabling failed. | ✓ | Primary |
| `789.0.814` | `avRollbackFailed` | Antivirus software rollback failed. | ✓ | Primary |
| `789.0.816` | `avRollback` | Antivirus software rolled back. | ✗ | — |
| `789.0.817` | `avUpdate` | Antivirus software updated. | ✗ | — |
| `789.0.1072` | `offboxvscanVirusDetectedError` | Off-box VScan — possible virus detected. | ✓ | Primary |

### Aggregate / Plex / Mirror

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.364` | `snapmirrorSyncFailed` | Synchronous SnapMirror failed — switched to asynchronous. | ✓ | Primary |
| `789.0.366` | `snapmirrorSyncOk` | Synchronous SnapMirror returned to sync mode. | ✗ | — |
| `789.0.444` | `plexFailed` | One plex of a mirrored aggregate failed. | ✓ | Primary |
| `789.0.454` | `plexOffline` | A plex went offline. | ✓ | Primary |
| `789.0.895` | `smVaultSnapWarnLimit` | SnapVault snapshot count limit reached. Dropped — backup team responsibility. | ✗ | — |
| `789.0.905` | `smSyncOutOfSyncWarn` | SM-S (SnapMirror Synchronous) CG relationship transitioned out-of-sync and remained so longer than expected — RPO=0 violated. No escalation trap. | ✓ | Exception C |

### IO Card

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.525` | `ioCardFailed` | IO card degraded — connectivity impact. No higher trap exists. | ✓ | Exception A |
| `789.0.526` | `ioCardOk` | IO card functioning properly. | ✗ | — |

### Remote Management

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.283` | `rmcCardNeedsReplacement` | Remote Management Controller card needs replacement. | ✓ | Primary |
| `789.0.284` | `rmcCardMissingCables` | RMC card missing internal/LAN/power cable. | ✓ | Primary |
| `789.0.532` | `remoteSystemMgtAlert` | Remote management detected system down event (alert). | ✓ | Primary |
| `789.0.535` | `remoteSystemMgmtWarning` | Remote management system down warning. Dropped — remoteSystemMgtAlert (Primary) covers escalation. | ✗ | — |
| `789.0.536` | `remoteSystemMgmtNotification` | Remote management system down notification. | ✗ | — |
| `789.0.547` | `remoteSystemMgmtPeriodic` | Periodic keepalive from remote management. | ✗ | — |
| `789.0.556` | `remotesystemMgmtTest` | Test trap from remote management. | ✗ | — |

### Clone & Snapshot

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.606` | `lunSnapRestoreStatus` | LUN snap restore status. | ✗ | — |
| `789.0.616` | `lunCloneCreate` | LUN clone created. | ✗ | — |
| `789.0.626` | `lunCloneSplitStart` | LUN clone split started. | ✗ | — |
| `789.0.627` | `lunCloneSplitComplete` | LUN clone split completed. | ✗ | — |
| `789.0.636` | `flexCloneSplitStart` | FlexClone split started. | ✗ | — |
| `789.0.637` | `flexCloneSplitComplete` | FlexClone split completed. | ✗ | — |
| `789.0.656` | `snapAutoDelete` | Snapshot auto-deleted. | ✗ | — |
| `789.0.837` | `lunDestroy` | LUN destroyed. | ✗ | — |
| `789.0.896` | `lunRelocationCompletion` | LUN relocation completed successfully. | ✗ | — |

### Volume Move

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.724` | `volMoveCutoverDeferred` | Volume move cannot complete — cutover deferred. | ✓ | Primary |
| `789.0.734` | `volMoveCutoverFailed` | Volume move cutover failed. | ✓ | Primary |
| `789.0.736` | `volMoveDone` | Volume move completed successfully. | ✗ | — |
| `789.0.737` | `volMoveCutoverDeferredWait` | Volume move waiting for user to trigger cutover. | ✗ | — |

### External Cache

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.676` | `extcacheCardError` | Flash Cache / PAM card failure — performance impact. No higher trap. | ✓ | Exception B |
| `789.0.686` | `extcacheCardOffline` | External cache offline. Dropped — follows extcacheCardError (kept) or is intentional admin action. | ✗ | — |

### QoS

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.755` | `qosMonitorMemoryMaxed` | QoS dynamic memory at limit. Dropped — self-resolves; no storage team action. | ✗ | — |
| `789.0.757` | `qosMonitorMemoryAbated` | QoS dynamic memory no longer at limit. | ✗ | — |

### Health Monitor

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.862` | `healthMonitorAlertRaised` | Health Monitor created an alert. | ✓ | Primary |
| `789.0.867` | `healthMonitorAlertCleared` | Health Monitor cleared an alert. | ✗ | — |

### NTP

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.674` | `timedTargetNotResponding` | NTP daemon lost contact with configured time target. | ✓ | Primary |

### System Reboot

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.712` | `rebootAbnormal` | System rebooted abnormally (watchdog reset, panic, etc.). | ✓ | Primary |
| `789.0.716` | `rebootNormal` | System rebooted normally. | ✗ | — |

### Backup

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.695` | `svBackupSnapWarningLimit` | Remaining snapshots for backup schedule below warning limit. Dropped — backup team responsibility. | ✗ | — |

### CIFS / Active Directory

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.434` | `prefDCDisconnect` | All connections to preferred domain controllers lost. | ✓ | Primary |
| `789.0.435` | `domainControllerDisconnect` | Individual DC connection failed. Dropped — AD infrastructure team; prefDCDisconnect covers total loss. | ✗ | — |
| `789.0.436` | `dcPasswdChangeFailed` | DC password sync failed. Dropped — AD infrastructure team action. | ✗ | — |
| `789.0.437` | `domainControllerConnected` | CIFS domain controller connection established. | ✗ | — |
| `789.0.497` | `cifsStatsExhaustMemCtrlBlk` | CIFS control blocks exhausted. Dropped — hard to action in real-time; noisy environment. | ✗ | — |
| `789.0.1121` | `noNisServersAvailable` | No configured NIS servers reachable. | ✓ | Primary |
| `789.0.1131` | `noLdapServersAvailable` | No configured LDAP servers reachable. | ✓ | Primary |
| `789.0.1141` | `noLsaServersAvailable` | No configured LSA servers reachable. | ✓ | Primary |
| `789.0.1151` | `noNetlogonServersAvailable` | No configured Netlogon servers reachable. | ✓ | Primary |

### SNMP Infrastructure

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.824` | `snmpBusy` | SNMP server too busy. | ✓ | Primary |
| `789.0.1103` | `snmpTestTraphost` | SNMP test trap generated. | ✓ | Primary |
| `789.0.1114` | `snmpFipsSupport` | ONTAP in FIPS mode but SNMPv3 configured with non-compliant settings. | ✓ | Primary |
| `789.0.1194` | `snmpSnmpv3Enable` | SNMPv3 disabled due to FIPS mode cluster upgrade. | ✓ | Primary |
| `789.0.1204` | `snmpFipsObjsDelFailed` | Failed to delete non-FIPS-compliant SNMP traphosts/users. | ✓ | Primary |

### Cloud / VSA

| OID | Trap Name | Description | Alert | Rule |
|---|---|---|:---:|:---|
| `789.0.1302` | `vsaCloudProviderScheduledEventScheduled` | Cloud provider maintenance event scheduled. | ✓ | Primary |
| `789.0.1303` | `vsaCloudProviderScheduledEventUpdate` | Scheduled cloud provider maintenance event status updated. | ✓ | Primary |

---

## Drop Rationale

Traps marked ✗ were excluded on one of three grounds:

### 1. Escalation coverage

Warning has a guaranteed critical escalation trap already in the Primary rule. Alerting on the warning creates a pre-ticket that gets superseded.

| Dropped warning | Covered by (Primary rule) |
|---|---|
| `fanWarning` | `fanFailed · fanFailureShutdown` |
| `powerSupplyWarning` | `powerSupplyFailed · powerSupplyFailureShutdown` |
| `overTemp` | `overTempShutdown` |
| `shelfWarning` | `shelfFault` |
| `chassisFanWarning` | `chassisFanStopped · chassisFanDegraded` |
| `shelfSESElectronicsWarning` | `shelfSESElectronicsFailed` |
| `shelfIFModuleWarning` | `shelfIFModuleFailed` |
| `remoteSystemMgmtWarning` | `remoteSystemMgtAlert` |
| `sesShelfPowerSupplyWarn` | `sesShelfPowerSupplyError` |
| `sesShelfDcmWarn` | `sesShelfDcmError` |
| `volumesStillFull` | `volumeFull (keeps firing while condition persists)` |
| `globalStatusNonCritical` | `globalStatusCritical` |
| `volumeStateChanged` | `volumeOffline · volumeRestricted` |
| `maxDirSizeWarning` | `maxDirSizeAlert` |
| `volumeReserveGrew` | `volumePhysicalOverallocated` |
| `fcOffLineNotification` | `fcOfflineError (error-caused path covered)` |

### 2. Storage team actionability

No meaningful real-time action available to the storage ops team.

| Dropped trap | Reason |
|---|---|
| `cpuTooBusy` | Performance planning item; not enabled by default |
| `chassisTemperatureUnknown` | Sensor fault — on-site hardware vendor only, no remote action |
| `chassisPowerSupplyOff` | Intentional or fault; PSU failure traps already in Primary |
| `chassisPSUsMismatch` | Post-maintenance config defect — change ticket, not incident |
| `chassisPSUwrongInput` | Physical install defect — on-site fix only |
| `vifPrimaryLinkFailed` | Backup link holds; vifAllLinksFailed (Exception B) covers total loss |
| `cifsStatsExhaustMemCtrlBlk` | Hard to action in real-time; high noise in CIFS environments |
| `diskMultipathWarning` | Redundancy risk only; diskMultipathNoTakeover (Primary) covers actual failure |
| `hbaOfflineInformation` | Duplicate risk with fcOfflineError; fires for user-initiated actions too |
| `extcacheCardOffline` | Follows extcacheCardError (kept) or intentional admin action |
| `qosMonitorMemoryMaxed` | Self-resolves when workload reduces; no storage team action |
| `volumeSelectedRootConflict` | Rare edge case; system auto-selects and boots |
| `alfFileNearlyFull` | Compliance/audit team action — storage ops cannot expand mid-flight |
| `quotaExceeded` | Fires for hard, soft, and threshold quotas — user quota management, not storage ops; storage cannot adjust quotas mid-incident |
| `svBackupSnapWarningLimit` | Backup team responsibility |
| `smVaultSnapWarnLimit` | Backup team responsibility |
| `domainControllerDisconnect` | AD infrastructure team; prefDCDisconnect (Primary) covers total DC loss |
| `dcPasswdChangeFailed` | AD infrastructure team — storage cannot fix AD password sync |
| `autosupportConfigurationError` | Admin/change item — NetApp support flags this independently |
| `sesShelfVoltageWarning` | Cannot fix voltage remotely; shelf hardware failure traps cover escalation |

### 3. Wrong team — AV licensing

Procurement and compliance team items. The storage ops team cannot renew a license from an incident ticket.

Dropped: `avTrendLicenseExpired` · `avTrendLicenseExpiring` · `avMcAfeeProductExpired` · `avMcAfeeEngineExpired` · `avMcAfeeProductExpiring` · `avMcAfeeEngineExpiring` · `avMcAfeeLicenseExpiring`

---

*Generated from NetApp ONTAP MIB v2.20 · Dynatrace SNMP ActiveGate pipeline*