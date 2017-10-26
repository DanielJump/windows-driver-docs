---
title: MB Protocol Configuration Operations (PCO)
description: MB Protocol Configuration Operations (PCO)
ms.assetid: 682C507C-5B2C-45E3-99D2-EEC68F8FC715
keywords:
- MB PCO operations, Mobile Broadband PCO operations, MB Protocol Configuration Operations, Mobile Broadband Protocol Configuration Operations, WDK network drivers, MBB miniport drivers
ms.author: windowsdriverdev
ms.date: 08/10/2017
ms.topic: article
ms.prod: windows-hardware
ms.technology: windows-devices
---

# MB Protocol Configuration Operations (PCO)

## Overview

Traditionally, the Windows NDIS definitions for Protocol Configuration Operations (PCO) values have been generic, for potentially receiving full PCO values from the modem and network in the future. As of Windows 10, version 1709, however, some modems are only able to pass up operator specific PCO elements to the OS. This topic defines the behavior of the current operator specific-only PCO implementation.

There are three scenarios where the PCO value will be passed to the host:
1.	When a new PCO value has arrived on an activated connection
2.	When an app or service queries for the latest PCO value from the modem
3.	When a connection is bridged or activated for the first time and a PCO value already exists in the modem

For the first scenario, the modem should send an [NDIS_STATUS_WWAN_PCO_STATUS](ndis-status-wwan-pco-status.md) notification to the OS indicating a new PCO value change whenever a new PCO value is received from the network, with the appropriate NDIS port number to represent the corresponding PDN. To avoid draining the battery unnecessarily, the modem should avoid noisy notifications, as described in [Modem behavior with Selective Suspend and Connected Standby](#modem-behavior-with-selective-suspend-and-connected-standby).

For the second scenario, when an app or service queries for PCO value from the modem on an activated PDN connection, the host will send the modem an [OID_WWAN_PCO](oid-wwan-pco.md) query request to read the latest cached PCO value in the modem.

For the third scenario, when a connection is activated or bridged on the host, the modem should send an **NDIS_STATUS_WWAN_PCO_STATUS** notification when a PCO value already exists in the modem for the activated or bridged connection the host requested. The notification should be passed up from the corresponding NDIS port number of the PDN.

The following figure shows the scenario flow:

![MB PCO operations flow](images/mb_PCO_operations_flow.png "MB PCO operations flow")

## Modem behavior with Selective Suspend and Connected Standby

When Selective Suspend is enabled, the modem can notify the OS whenever it receives a PCO data structure from the network. However, the modem should avoid unnecessary device wakeup. Otherwise, noisy PCO notifications from the network will wake the device up frequently and drain the battery unnecessarily.

When Connected Standby is enabled, the modem shouldn’t notify the OS when it receives PCO data structures from the network because it will not only wake up the device, but it will also wake up the OS, which is not necessary. Instead, the modem should cache all the latest PCO elements from the data structure and notify the OS once the OS exits Connected Standby. For an MBIM modem, it should cache all PCO data structures and only send PCO notifications to the OS after the host has subscribed to it. This will be done using the MBIM_CID_DEVICE_SERVICE_SUBSCRIBE_LIST CID when system power has returned to full power after coming out of Connected Standby.

## Resetting the modem based on PCO values

Based on PCO values received from the network, the modem will be reset in the following scenarios:

1.	The user completed self-activation after receiving PCO = 5 from the network. A new PCO value (3, 0 or anything Mobile Operator App can recognize) will be sent to the OS and the OS will pass it to Mobile Operator App.
2.	The user added more credit to his or her account after receiving PCO = 3. A new PCO value (0, or anything Mobile Operator App can recognize) will be sent to the OS and the OS will pass it to Mobile Operator App.

The host is not aware of the modem being reset, so the activated connections from the host will not be deactivated and the modem should automatically re-establish connection with those PDN after resetting. Upon establishing connection and receiving a new incoming PCO value from the network, the modem will provide an unsolicited [NDIS_STATUS_WWAN_PCO_STATUS](ndis-status-wwan-pco-status.md) notification to the host.

The following diagram illustrates the modem’s reset flow when one of these scenarios occurs, with Verizon Wireless as the example MO:

![MB modem reset based on PCO values](images/mb_PCO_modem_reset.png "MB modem reset based on PCO values")

## NDIS interface to the modem

For querying the status and payload of a PCO value the modem received from the operator network, see [OID_WWAN_PCO](oid-wwan-pco.md). **OID_WWAN_PCO** uses the [**NDIS_WWAN_PCO_STATUS**](https://msdn.microsoft.com/library/windows/hardware/C71187C5-74B6-450A-8461-BB9FDF60DB8D) structure, which in turn contains a [**WWAN_PCO_VALUE**](https://msdn.microsoft.com/library/windows/hardware/45A499CE-2C9A-4070-BEF8-880E7673FA8E)  structure representing the PCO information payload from the network.

For the status notification sent by a modem miniport driver to inform the OS of the current PCO state in the modem, see [NDIS_STATUS_WWAN_PCO_STATUS](ndis-status-wwan-pco-status.md).

## MB CID to the modem

Service = **MBB_UUID_BASIC_CONNECT_EXT_CONSTANT**

Service UUID = **3d01dcc5-fef5-4d05-9d3a-bef7058e9aaf**

The following CIDs are defined for PCO:

| CID | Minimum OS Version |
| --- | --- |
| MBIM_CID_PCO | Windows 10, version 1709 |

### MBIM_CID_PCO

This command is used to query the PCO data cached in modem from the mobile operator network.

#### Query

The InformationBuffer contains an **MBIM_PCO_VALUE** in which the only relevant field is *SessionId*. *SessionId* is reserved for future use and will always be 0 in Windows 10, version 1709. The *SessionId* in a query indicates which IP data stream’s PCO value is to be returned by the function. 

#### Set

Not applicable.

#### Unsolicited Event

Unsolicited events contain an MBIM_PCO_VALUE and are sent when a new PCO value has arrived on an activated connection.

#### Parameters

|  | Set | Query | Notification |
| --- | --- | --- | --- |
| Command | Not applicable | MBIM_PCO_VALUE | Not applicable |
| Response | Not applicable | MBIM_PCO_VALUE | MBIM_PCO_VALUE |

#### Data Structures

##### MBIM_PCO_TYPE

| Type | Value | Description |
| --- | --- | --- |
| MBIMPcoTypeComplete | 0 | Specifies that the complete PCO structure will be passed up as received from the network and the header realistically reflects the protocol in octet 3 of the PCO structure, defined in the 3GPP TS24.008 spec. |
| MBIMPcoTypePartial | 1 | Specifies that the modem will only be passing up a subset of PCO structures that it received from the network. The header matches the PCO structure defined in the 3GPP TS24.008 spec, but the “Configuration protocol” of octet 3 may not be valid. |

##### MBIM_PCO_VALUE

| Offset | Size | Field | Type | Description |
| --- | --- | --- | --- | --- |
| 0 | 4 | SessionId | UINT32 | The SessionId in a query indicates which IP data stream’s PCO value is to be returned by the function. |
| 4 | 4 | PcoDataSize | UINT32 | The length of PcoData, from 0 to 256. This value will be 0 in a query. |
| 8 | 4 | PcoDataType | UINT32 | The PCO data type. For more info, see [MBIM_PCO_TYPE](#mbimpcotype). |
| 12 | | PcoDataBuffer | DATABUFFER | The PCO structure from the 3GPP TS24.008 spec. |

#### Status Codes

This CID only uses Generic Status Codes.

[Send comments about this topic to Microsoft](mailto:wsddocfb@microsoft.com?subject=Documentation%20feedback%20%5Bprint\print%5D:%20Slicer%20settings%20%20RELEASE:%20%289/2/2016%29&body=%0A%0APRIVACY%20STATEMENT%0A%0AWe%20use%20your%20feedback%20to%20improve%20the%20documentation.%20We%20don't%20use%20your%20email%20address%20for%20any%20other%20purpose,%20and%20we'll%20remove%20your%20email%20address%20from%20our%20system%20after%20the%20issue%20that%20you're%20reporting%20is%20fixed.%20While%20we're%20working%20to%20fix%20this%20issue,%20we%20might%20send%20you%20an%20email%20message%20to%20ask%20for%20more%20info.%20Later,%20we%20might%20also%20send%20you%20an%20email%20message%20to%20let%20you%20know%20that%20we've%20addressed%20your%20feedback.%0A%0AFor%20more%20info%20about%20Microsoft's%20privacy%20policy,%20see%20http://privacy.microsoft.com/default.aspx. "Send comments about this topic to Microsoft")