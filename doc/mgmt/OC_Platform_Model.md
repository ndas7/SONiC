# OpenConfig support for Platform Components Model.

# High Level Design Document
#### Rev 0.1

# Table of Contents
  * [List of Tables](#list-of-tables)
  * [Revision](#revision)
  * [Overview](#overview)
  * [Definition/Abbreviation](#definitionabbreviation)
  * [1 Feature Overview](#1-feature-overview)
	* [1.1 Requirements](#11-requirements)
	  * [1.1.1 Functional Requirements](#111-functional-requirements)
	* [1.2 Design Overview](#12-design-overview)
	  * [1.2.1 Basic Approach](#121-basic-approach)
	  * [1.2.2 Container](#122-container)
  * [2 Functionality](#2-functionality)
	  * [2.1 Target Deployment Use Cases](#21-target-deployment-use-cases)
  * [3 Design](#3-design)
	* [3.1 Overview](#31-overview)
	* [3.2 User Interface](#33-user-interface)
	  * [3.2.1 Data Models](#331-data-models)
	  * [3.2.2 gNMI Support](#333-gnmi-support)
  * [4 Mapping between Openconfig YANG and Redis DB](#4-mapping-between-openconfig-yang-and-redis-db)
  * [5 Error Handling](#5-error-handling)
  * [6 Unit Test Cases](#6-unit-test-cases)
	* [6.1 Functional Test Cases](#61-functional-test-cases)
	* [6.2 Negative Test Cases](#62-negative-test-cases)
  
# List of Tables
[Table 1: Abbreviations](#table-1-abbreviations)

[Table 2: Mapping attributes between OpenConfig YANG and RedisDB](#table-2-mapping-attributes-between-openconfig-yang-and-redisdb)

# Revision
| Rev |     Date    |       Author          | Change Description                |
|:---:|:-----------:|:---------------------:|-----------------------------------|
| 0.1 | 08/20/2025  | Neha Das | Initial version                   |

# Overview
This document aims to describe the requirements for supporting the OpenConfig Platform Component models for configuration management and telemetry for component state information.

- Supported attributes in OpenConfig YANG tree:

```  
module: openconfig-platform
  +--rw components
     +--rw component* [name]
        +--rw name                                             -> ../config/name
        +--rw config
        |  +--rw name?   string
        +--ro state
        |  +--ro name?                                string
        |  +--ro type?                                union
        |  x--ro id?                                  string
        |  x--ro location?                            string
        |  +--ro install-position?                    string
        |  +--ro install-component?                   -> ../name
        |  +--ro description?                         string
        |  +--ro mfg-name?                            string
        |  +--ro mfg-date?                            oc-yang:date
        |  +--ro hardware-version?                    string
        |  +--ro firmware-version?                    string
        |  +--ro software-version?                    string
        |  +--ro serial-no?                           string
        |  +--ro part-no?                             string
        |  +--ro model-name                           string
        |  +--ro clei-code?                           string
        |  +--ro removable?                           boolean
        |  +--ro oper-status?                         identityref
        |  +--ro empty?                               boolean
        |  +--ro parent?                              -> ../../../component/config/name
        |  +--ro redundant-role?                      oc-platform-types:component-redundant-role
        |  +--ro last-poweroff-reason
        |  |  +--ro trigger?   component-last-poweroff-reason-trigger
        |  |  +--ro details?   string
        |  +--ro last-poweroff-time?                  oc-types:timeticks64
        |  +--ro last-switchover-reason
        |  |  +--ro trigger?   component-redundant-role-switchover-reason-trigger
        |  |  +--ro details?   string
        |  +--ro last-switchover-time?                oc-types:timeticks64
        |  +--ro last-reboot-reason?                  identityref
        |  x--ro last-reboot-time?                    oc-types:timeticks64
        |  +--ro boot-time?                           oc-types:timeticks64
        |  +--ro switchover-ready?                    boolean
        |  +--ro base-mac-address?                    oc-yang:mac-address
        |  +--ro temperature
        |  |  +--ro instant?                                        decimal64
        |  |  +--ro avg?                                            decimal64
        |  |  +--ro min?                                            decimal64
        |  |  +--ro max?                                            decimal64
        |  |  +--ro interval?                                       oc-types:stat-interval
        |  |  +--ro min-time?                                       oc-types:timeticks64
        |  |  +--ro max-time?                                       oc-types:timeticks64
        |  |  +--ro alarm-status?                                   boolean
        |  |  +--ro alarm-threshold?                                uint32
        |  |  +--ro alarm-severity?                                 identityref
        |  |  +--ro google-pins-platform:low-warn-threshold?        decimal64
        |  |  +--ro google-pins-platform:low-critical-threshold?    decimal64
        |  |  +--ro google-pins-platform:high-warn-threshold?       decimal64
        |  |  +--ro google-pins-platform:high-critical-threshold?   decimal64
        |  +--ro memory
        |  |  +--ro available?   uint64
        |  |  +--ro utilized?    uint64
        |  +--ro allocated-power?                     uint32
        |  +--ro used-power?                          uint32
        |  +--ro pcie
        |  |  +--ro fatal-errors
        |  |  |  +--ro total-errors?                   oc-yang:counter64
        |  |  |  +--ro undefined-errors?               oc-yang:counter64
        |  |  |  +--ro data-link-errors?               oc-yang:counter64
        |  |  |  +--ro surprise-down-errors?           oc-yang:counter64
        |  |  |  +--ro poisoned-tlp-errors?            oc-yang:counter64
        |  |  |  +--ro flow-control-protocol-errors?   oc-yang:counter64
        |  |  |  +--ro completion-timeout-errors?      oc-yang:counter64
        |  |  |  +--ro completion-abort-errors?        oc-yang:counter64
        |  |  |  +--ro unexpected-completion-errors?   oc-yang:counter64
        |  |  |  +--ro receiver-overflow-errors?       oc-yang:counter64
        |  |  |  +--ro malformed-tlp-errors?           oc-yang:counter64
        |  |  |  +--ro ecrc-errors?                    oc-yang:counter64
        |  |  |  +--ro unsupported-request-errors?     oc-yang:counter64
        |  |  |  +--ro acs-violation-errors?           oc-yang:counter64
        |  |  |  +--ro internal-errors?                oc-yang:counter64
        |  |  |  +--ro blocked-tlp-errors?             oc-yang:counter64
        |  |  |  +--ro atomic-op-blocked-errors?       oc-yang:counter64
        |  |  |  +--ro tlp-prefix-blocked-errors?      oc-yang:counter64
        |  |  +--ro non-fatal-errors
        |  |  |  +--ro total-errors?                   oc-yang:counter64
        |  |  |  +--ro undefined-errors?               oc-yang:counter64
        |  |  |  +--ro data-link-errors?               oc-yang:counter64
        |  |  |  +--ro surprise-down-errors?           oc-yang:counter64
        |  |  |  +--ro poisoned-tlp-errors?            oc-yang:counter64
        |  |  |  +--ro flow-control-protocol-errors?   oc-yang:counter64
        |  |  |  +--ro completion-timeout-errors?      oc-yang:counter64
        |  |  |  +--ro completion-abort-errors?        oc-yang:counter64
        |  |  |  +--ro unexpected-completion-errors?   oc-yang:counter64
        |  |  |  +--ro receiver-overflow-errors?       oc-yang:counter64
        |  |  |  +--ro malformed-tlp-errors?           oc-yang:counter64
        |  |  |  +--ro ecrc-errors?                    oc-yang:counter64
        |  |  |  +--ro unsupported-request-errors?     oc-yang:counter64
        |  |  |  +--ro acs-violation-errors?           oc-yang:counter64
        |  |  |  +--ro internal-errors?                oc-yang:counter64
        |  |  |  +--ro blocked-tlp-errors?             oc-yang:counter64
        |  |  |  +--ro atomic-op-blocked-errors?       oc-yang:counter64
        |  |  |  +--ro tlp-prefix-blocked-errors?      oc-yang:counter64
        |  |  +--ro correctable-errors
        |  |  |  +--ro total-errors?                oc-yang:counter64
        |  |  |  +--ro receiver-errors?             oc-yang:counter64
        |  |  |  +--ro bad-tlp-errors?              oc-yang:counter64
        |  |  |  +--ro bad-dllp-errors?             oc-yang:counter64
        |  |  |  +--ro relay-rollover-errors?       oc-yang:counter64
        |  |  |  +--ro replay-timeout-errors?       oc-yang:counter64
        |  |  |  +--ro advisory-non-fatal-errors?   oc-yang:counter64
        |  |  |  +--ro internal-errors?             oc-yang:counter64
        |  |  |  +--ro hdr-log-overflow-errors?     oc-yang:counter64
        |  +--ro oc-alarms:equipment-failure?         boolean
        |  +--ro oc-alarms:equipment-mismatch?        boolean
        |  +--ro oc-platform-ext:entity-id?           uint32
        +--rw fan
        |  +--rw config
        |  +--ro state
        |     +--ro google-pins-platform:max-speed?           uint32
        |     +--ro google-pins-platform:speed-control-pct?   uint32
        |     +--ro oc-fan:speed?                             uint32
        +--rw integrated-circuit
        |  +--rw config
        |  |  +--rw oc-p4rt:node-id?   uint64
        |  +--ro state
        |  |  +--ro oc-p4rt:node-id?                               uint64
        +--rw oc-transceiver:transceiver
        |  +--rw oc-transceiver:config
        |  |  +--rw oc-transceiver:enabled?                  boolean
        |  |  +--rw oc-transceiver:form-factor-preconf?      identityref
        |  |  +--rw oc-transceiver:ethernet-pmd-preconf?     identityref
        |  |  +--rw oc-transceiver:fec-mode?                 identityref
        |  |  +--rw oc-transceiver:module-functional-type?   identityref
        |  +--ro oc-transceiver:state
        |  |  +--ro oc-transceiver:enabled?                                   boolean
        |  |  +--ro oc-transceiver:form-factor-preconf?                       identityref
        |  |  +--ro oc-transceiver:ethernet-pmd-preconf?                      identityref
        |  |  +--ro oc-transceiver:fec-mode?                                  identityref
        |  |  +--ro oc-transceiver:module-functional-type?                    identityref
        |  |  +--ro oc-transceiver:present?                                   enumeration
        |  |  +--ro oc-transceiver:form-factor?                               identityref
        |  |  +--ro oc-transceiver:connector-type?                            identityref
        |  |  +--ro oc-transceiver:vendor?                                    string
        |  |  +--ro oc-transceiver:vendor-part?                               string
        |  |  +--ro oc-transceiver:vendor-rev?                                string
        |  |  +--ro oc-transceiver:ethernet-pmd?                              identityref
        |  |  +--ro oc-transceiver:sonet-sdh-compliance-code?                 identityref
        |  |  +--ro oc-transceiver:otn-compliance-code?                       identityref
        |  |  +--ro oc-transceiver:serial-no?                                 string
        |  |  +--ro oc-transceiver:date-code?                                 oc-yang:date-and-time
        |  |  +--ro oc-transceiver:fault-condition?                           boolean
        |  |  +--ro oc-transceiver:fec-status?                                identityref
        |  |  +--ro oc-transceiver:fec-uncorrectable-blocks?                  yang:counter64
        |  |  +--ro oc-transceiver:fec-uncorrectable-words?                   yang:counter64
        |  |  +--ro oc-transceiver:fec-corrected-bytes?                       yang:counter64
        |  |  +--ro oc-transceiver:fec-corrected-bits?                        yang:counter64
        |  |  +--ro oc-transceiver:pre-fec-ber
        |  |  |  +--ro oc-transceiver:instant?    decimal64
        |  |  |  +--ro oc-transceiver:avg?        decimal64
        |  |  |  +--ro oc-transceiver:min?        decimal64
        |  |  |  +--ro oc-transceiver:max?        decimal64
        |  |  |  +--ro oc-transceiver:interval?   oc-types:stat-interval
        |  |  |  +--ro oc-transceiver:min-time?   oc-types:timeticks64
        |  |  |  +--ro oc-transceiver:max-time?   oc-types:timeticks64
        |  |  +--ro oc-transceiver:post-fec-ber
        |  |  |  +--ro oc-transceiver:instant?    decimal64
        |  |  |  +--ro oc-transceiver:avg?        decimal64
        |  |  |  +--ro oc-transceiver:min?        decimal64
        |  |  |  +--ro oc-transceiver:max?        decimal64
        |  |  |  +--ro oc-transceiver:interval?   oc-types:stat-interval
        |  |  |  +--ro oc-transceiver:min-time?   oc-types:timeticks64
        |  |  |  +--ro oc-transceiver:max-time?   oc-types:timeticks64
        |  |  +--ro oc-transceiver:supply-voltage
        |  |  |  +--ro oc-transceiver:instant?    decimal64
        |  |  |  +--ro oc-transceiver:avg?        decimal64
        |  |  |  +--ro oc-transceiver:min?        decimal64
        |  |  |  +--ro oc-transceiver:max?        decimal64
        |  |  |  +--ro oc-transceiver:interval?   oc-types:stat-interval
        |  |  |  +--ro oc-transceiver:min-time?   oc-types:timeticks64
        |  |  |  +--ro oc-transceiver:max-time?   oc-types:timeticks64
        |  |  +--ro oc-transceiver:output-power
        |  |  |  +--ro oc-transceiver:instant?                         decimal64
        |  |  |  +--ro oc-transceiver:avg?                             decimal64
        |  |  |  +--ro oc-transceiver:min?                             decimal64
        |  |  |  +--ro oc-transceiver:max?                             decimal64
        |  |  |  +--ro oc-transceiver:interval?                        oc-types:stat-interval
        |  |  |  +--ro oc-transceiver:min-time?                        oc-types:timeticks64
        |  |  |  +--ro oc-transceiver:max-time?                        oc-types:timeticks64
        |  |  +--ro oc-transceiver:input-power
        |  |  |  +--ro oc-transceiver:instant?                         decimal64
        |  |  |  +--ro oc-transceiver:avg?                             decimal64
        |  |  |  +--ro oc-transceiver:min?                             decimal64
        |  |  |  +--ro oc-transceiver:max?                             decimal64
        |  |  |  +--ro oc-transceiver:interval?                        oc-types:stat-interval
        |  |  |  +--ro oc-transceiver:min-time?                        oc-types:timeticks64
        |  |  |  +--ro oc-transceiver:max-time?                        oc-types:timeticks64
        |  |  +--ro oc-transceiver:laser-bias-current
        |  |  |  +--ro oc-transceiver:instant?    decimal64
        |  |  |  +--ro oc-transceiver:avg?        decimal64
        |  |  |  +--ro oc-transceiver:min?        decimal64
        |  |  |  +--ro oc-transceiver:max?        decimal64
        |  |  |  +--ro oc-transceiver:interval?   oc-types:stat-interval
        |  |  |  +--ro oc-transceiver:min-time?   oc-types:timeticks64
        |  |  |  +--ro oc-transceiver:max-time?   oc-types:timeticks64
        |  +--rw oc-transceiver:physical-channels
        |  |  +--rw oc-transceiver:channel* [index]
        |  |     +--rw oc-transceiver:index     -> ../config/index
        |  |     +--rw oc-transceiver:config
        |  |     |  +--rw oc-transceiver:index?                        uint16
        |  |     |  +--rw oc-transceiver:associated-optical-channel?   -> /oc-platform:components/component/name
        |  |     |  +--rw oc-transceiver:description?                  string
        |  |     |  +--rw oc-transceiver:tx-laser?                     boolean
        |  |     |  +--rw oc-transceiver:target-output-power?          decimal64
        |  |     +--ro oc-transceiver:state
        |  |        +--ro oc-transceiver:index?                        uint16
        |  |        +--ro oc-transceiver:associated-optical-channel?   -> /oc-platform:components/component/name
        |  |        +--ro oc-transceiver:description?                  string
        |  |        +--ro oc-transceiver:tx-laser?                     boolean
        |  |        +--ro oc-transceiver:target-output-power?          decimal64
        |  |        +--ro oc-transceiver:laser-age?                    oc-types:percentage
        |  |        +--ro oc-transceiver:laser-temperature
        |  |        |  +--ro oc-transceiver:instant?    decimal64
        |  |        |  +--ro oc-transceiver:avg?        decimal64
        |  |        |  +--ro oc-transceiver:min?        decimal64
        |  |        |  +--ro oc-transceiver:max?        decimal64
        |  |        |  +--ro oc-transceiver:interval?   oc-types:stat-interval
        |  |        |  +--ro oc-transceiver:min-time?   oc-types:timeticks64
        |  |        |  +--ro oc-transceiver:max-time?   oc-types:timeticks64
        |  |        +--ro oc-transceiver:target-frequency-deviation
        |  |        |  +--ro oc-transceiver:instant?    decimal64
        |  |        |  +--ro oc-transceiver:avg?        decimal64
        |  |        |  +--ro oc-transceiver:min?        decimal64
        |  |        |  +--ro oc-transceiver:max?        decimal64
        |  |        |  +--ro oc-transceiver:interval?   oc-types:stat-interval
        |  |        |  +--ro oc-transceiver:min-time?   oc-types:timeticks64
        |  |        |  +--ro oc-transceiver:max-time?   oc-types:timeticks64
        |  |        +--ro oc-transceiver:tec-current
        |  |        |  +--ro oc-transceiver:instant?    decimal64
        |  |        |  +--ro oc-transceiver:avg?        decimal64
        |  |        |  +--ro oc-transceiver:min?        decimal64
        |  |        |  +--ro oc-transceiver:max?        decimal64
        |  |        |  +--ro oc-transceiver:interval?   oc-types:stat-interval
        |  |        |  +--ro oc-transceiver:min-time?   oc-types:timeticks64
        |  |        |  +--ro oc-transceiver:max-time?   oc-types:timeticks64
        |  |        +--ro oc-transceiver:tx-failure?                   boolean
        |  |        +--ro oc-transceiver:rx-los?                       boolean
        |  |        +--ro oc-transceiver:rx-cdr-lol?                   boolean
        |  |        +--ro oc-transceiver:output-frequency?             oc-opt-types:frequency-type
        |  |        +--ro oc-transceiver:output-power
        |  |        |  +--ro oc-transceiver:instant?    decimal64
        |  |        |  +--ro oc-transceiver:avg?        decimal64
        |  |        |  +--ro oc-transceiver:min?        decimal64
        |  |        |  +--ro oc-transceiver:max?        decimal64
        |  |        |  +--ro oc-transceiver:interval?   oc-types:stat-interval
        |  |        |  +--ro oc-transceiver:min-time?   oc-types:timeticks64
        |  |        |  +--ro oc-transceiver:max-time?   oc-types:timeticks64
        |  |        +--ro oc-transceiver:input-power
        |  |        |  +--ro oc-transceiver:instant?    decimal64
        |  |        |  +--ro oc-transceiver:avg?        decimal64
        |  |        |  +--ro oc-transceiver:min?        decimal64
        |  |        |  +--ro oc-transceiver:max?        decimal64
        |  |        |  +--ro oc-transceiver:interval?   oc-types:stat-interval
        |  |        |  +--ro oc-transceiver:min-time?   oc-types:timeticks64
        |  |        |  +--ro oc-transceiver:max-time?   oc-types:timeticks64
        |  |        +--ro oc-transceiver:laser-bias-current
        |  |        |  +--ro oc-transceiver:instant?    decimal64
        |  |        |  +--ro oc-transceiver:avg?        decimal64
        |  |        |  +--ro oc-transceiver:min?        decimal64
        |  |        |  +--ro oc-transceiver:max?        decimal64
        |  |        |  +--ro oc-transceiver:interval?   oc-types:stat-interval
        |  |        |  +--ro oc-transceiver:min-time?   oc-types:timeticks64
        |  |        |  +--ro oc-transceiver:max-time?   oc-types:timeticks64

```  

# Definition/Abbreviation
### Table 1: Abbreviations
| **Term**                 | **Definition**                         |
|--------------------------|-------------------------------------|
| YANG                     | Yet Another Next Generation: modular language representing data structures in an XML tree format        |
| gNMI                     | gRPC Network Management Interface: used to retrieve or manipulate the state of a device via telemetry or configuration data         |
| XML                      | eXtensible Markup Language   |

# 1 Feature Overview
## 1.1 Requirements
### 1.1.1 Functional Requirements
1. Provide support for OpenConfig YANG for the Platform Components model for the following components:
  1.1 Chassis
  1.2 Software Component
  1.3 Fan
  1.4 Transceiver
  1.5 Integrated-circuit
  1.6 Sensors
  1.7 Storage
  1.8 PCIE devices / errors
3. Support configuration through gNMI Set and telemetry through gNMI Get and gNMI Subscribe SAMPLE/TARGET_DEFINED.


## 1.2 Design Overview
### 1.2.1 Basic Approach
SONiC already supports telemetry for parameters under interfaces and system models with methods such as Get via REST and gNMI. This feature adds support for config and telemetry for the proposed components with gNMI and YANG models utilizing on translib infra.

### 1.2.2 Container
The code changes for this feature are part of *Management Framework* container which includes the REST server and *gnmi* container for gNMI support in *sonic-mgmt-common* repository.

# 2 Functionality
## 2.1 Target Deployment Use Cases
1. gNMI client with support for capabilities of GET / SUBSCRIBE ONCE, POLL, STREAM SAMPLE based on the supported YANG models.

# 3 Design
## 3.1 Overview
This HLD design is in line with the [Management Framework HLD](https://github.com/project-arlo/SONiC/blob/354e75b44d4a37b37973a3a36b6f55141b4b9fdf/doc/mgmt/Management%20Framework.md)

## 3.2 User Interface
### 3.2.1 Data Models
Data models for this feature are based on Openconfig-platform.yang(ver 0.27.0) and openconfig-platform-transceiver.yang(ver 0.14.0).

### 3.2.2 gNMI Support

#### 3.2.2.1 GET
Supported.

Sample GET output for platform/components/component[name=integrated_circuit0]/state/node-id node:
```

```

#### 3.2.2.2 SUBSCRIBE
Sample telemetry logs with once mode on platform/components/component[name=integrated_circuit0]/state/node-id node
```

```


# 4 Mapping between OpenConfig YANG and Redis DB
### Table 2: Mapping attributes between OpenConfig YANG and RedisDB:

<table rules="all">
	<tr style="background-color:turquoise">
		<th>OpenConfig Yang</th>
		<th colspan=3> Redis DB</th>
	</tr>
	<tr style="background-color:paleturquoise">
		<th>Node</th>
		<th align="center">OpenConfig Key Name</th>
    <th align="center">DB Name</th>
		<th align="center">Table Name</th>
		<th align="center">Object</th>
	</tr>
	<tr>
		<th align="left">openconfig-platform.yang</th>
		<td></td>
		<td></td>
		<td></td>
	</tr>
	<tr>
		<th align="left" style="padding-left: 1em;">components</th>
		<td></td>
		<td></td>
		<td></td>
	</tr>
	<tr>
		<th align="left" style="padding-left: 2em;">component</th>
		<td></td>
		<td></td>
		<td></td>
	</tr>
	<tr>
		<th align="left" style="padding-left: 3em;">state</th>
		<td></td>
		<td></td>
		<td></td>
	</tr>
	<tr>
		<th align="left" style="padding-left: 4em;">node-id</th>
		<td>integrated_circuit0</td>
    <td>State_DB</td>
		<td>CHASSIS_INFO</td>
		<td>256</td>
	</tr>
</table>


# 6 Unit Test cases
## 6.1 Functional Test Cases
Operations: 

gNMI - Set
1.	Verify that operations supported for gNMI Set works fine for node-id


gNMI - Get / Subscribe

1.	Verify that operations supported for gNMI works fine for chassis component
2.	Verify that operations supported for gNMI works fine for software components
3.	Verify that operations supported for gNMI works fine for fan component
4.	Verify that operations supported for gNMI works fine for transceiver component
5.	Verify that operations supported for gNMI works fine for integrated-circuit component
6.	Verify that operations supported for gNMI works fine for sensors component
7.	Verify that operations supported for gNMI works fine for storage component
8.	Verify that operations supported for gNMI works fine for pcie devices component


## 6.2 Negative Test Cases
1. Verify that any operation on unsupported paths give the correct error.
2. Verify that GET on components with non-existing component name give the correct error.
