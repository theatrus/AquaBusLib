## Table of Contents
  
- [Neptune Apex Data Transfer](#neptune-apex-data-transfer)
  * [Apex Modbus configuration](#apex-modbus-configuration)
  * [Apex Modbus Frame structure](#apex-modbus-frame-structure)
  * [Known Apex Function Codes](#known-apex-function-codes)
- [Apex Device Communication](#apex-device-communication)
  * [Introducing new device to Apex](#introducing-new-device-to-apex)
    + [Probe Request Stage 1](#probe-request-stage-1)
    + [Probe Request Stage 2](#probe-request-stage-2)
    + [Probe Request Stage 3](#probe-request-stage-3)
    + [Probe Request Stage 5](#probe-request-stage-5)
  * [Reconnecting existing module to Apex](#reconnecting-existing-module-to-apex)
- [Device Specific Apex Communication](#device-specific-apex-communication)
  * [Energy Bar - 8 outlets (EB8)](#energy-bar---8-outlets--eb8-)
    + [Setting outlets and getting current reading](#setting-outlets-and-getting-current-reading)
    + [Calibrating EB8 current sensor](#calibrating-eb8-current-sensor)
  * [Probe Module 2 (PM2) - Salinity Module](#probe-module-2--pm2----salinity-module)
    + [Retrieving module and probe specific configuration](#retrieving-module-and-probe-specific-configuration)
    + [Calibrating PM2 module](#calibrating-pm2-module)
    + [Polling for probe and switch data](#polling-for-probe-and-switch-data)
    + [Emulating PM2 module for Switches functionality only](#emulating-pm2-module-for-switches-functionality-only)
  * [Probe Module 1 (PM1) - pH/ORP Module](#probe-module-1--pm1----ph-orp-module)
    + [Retrieving module and probe specific configuration](#retrieving-module-and-probe-specific-configuration-1)
    + [Calibrating PM1 module](#calibrating-pm1-module)
    + [Polling for probe and switch data](#polling-for-probe-and-switch-data-1)
    + [Emulating PM1 module for Switches functionality only](#emulating-pm1-module-for-switches-functionality-only)
  * [Probe Module 3 (PM3) - Dissolved Oxygen Module](#probe-module-3--pm3----dissolved-oxygen-module)
    + [Retrieving module and probe specific configuration](#retrieving-module-and-probe-specific-configuration-2)
    + [Calibrating PM3 module](#calibrating-pm3-module)
    + [Polling for probe and switch data](#polling-for-probe-and-switch-data-2)
    + [Emulating PM3 module for Switches functionality only](#emulating-pm3-module-for-switches-functionality-only)
		
## Neptune Apex Data Transfer

Although, Apex uses CAN transceiver at the physical layer, it does not implement CAN for data transfers. Instead, Apex uses [Modbus
RTU](https://en.wikipedia.org/wiki/Modbus) to communicate with addon modules and for arbitration. Specifically, it relies heavily
on [freemodbus](https://www.freemodbus.org) library for Modbus data transfer implementation.

### Apex Modbus configuration
Baudrate: 19200  
Data Bits: 8  
Parity: Even  
Stop bits: 1  

### Apex Modbus Frame structure
```
<------------------------ MODBUS SERIAL LINE PDU (1) ------------------->
             <----------- MODBUS PDU (1') ---------------->
 +-----------+---------------+----------------------------+-------------+
 | Address   | Function Code | Data                       | CRC/LRC     |
 +-----------+---------------+----------------------------+-------------+
 ```
 
 Where:
- Address - one byte modbus address of the destination device 
- Function Code - one byte function code tells destination device what to do
- Data - rest of the packet
- CRC/LRC - Modbus CRC16 over the entire frame, starting at "Address" until the end of the packet.

### Known Apex Function Codes

This is the list of known Function Codes supported by Apex:


| Function Code | Description                                                                        |
|---------------|------------------------------------------------------------------------------------|
| 0x01          | Device Probe Request. Used to attach new and existing devices to Apex              |
| 0x07          | Device firmware update Request. Used by Apex to push firmware updates to  connected devices  |
| 0x08          | Device EEPROM Request. Used by Apex to set and get EEPROM of attached device       |
| 0x11          | Apex Display Request. Used to communicate with Apex Display                        |
| 0x20          | Device Communication Request. Apex uses this function code to send commands to  already connected devices |

Table 1: Known Apex Function Codes

## Apex Device Communication

As Apex uses Modbus for device communication, the head unit always acts as the Modbus master. All attached devices are configured as Modbus slaves. Each device is given a unique address. The head unit always initiates a communication. Slave devices respond to commands from the head unit.

### Introducing new device to Apex

Apex head unit sends a probe request to all devices connected to the network. If a new device is attached, it response to the probe request, which initiates a series of requests that eventually result in the device being attached to Apex.

```
struct AB_PROBE_REQUEST_PACKET
{
  byte FunctionCode;
  byte ProbeStage;
  byte Address;
  unsigned short ApexSerialNumber;
  byte unknown[3];
}
```

Apex sends this request to the broadcast address 0x00 for all devices to pick up. FunctionCode is 0x01 for all Probe stages. Here's the list of probe stages:

| Probe Stage   | Description                                                                        |
|---------------|------------------------------------------------------------------------------------|
| 0x01          | Initial stage of probe request. Apex sends this to broadcast looking for newly attached devices | 
| 0x02          | Second state of probe request. Apex sends this if the response from initial stage came from device not previously registered with Apex                               
| 0x03          | Third stage of probe request. Apex agreed to install the new device and is in final configuration stage 
| 0x05          | Final Stage of probe request. Also called the Attach stage. This is the request sent to new devices telling them that they are now attached and ready to go. This is also the request sent to known devices on reattach.                        

Table 2: Probe Request Stages

#### Probe Request Stage 1
In the first probe stage, Apex sends two important pieces of information: Next available AquaBus address and Apex's serial number. This way the device being probed can check if it has already been previously attached to this Apex and on what address. Or if this is a new device, it can take the next available AquaBus address on registration.
The first two AquaBus addresses (0x01 and 0x02) are always reserved for up to two Displays. Typically, the next available address (0x03) gets taken by the Power Bar (Energy Bar), but this is not strictly reserved for it.
Here's an example of a complete initial probe request:
```
00 01 01 03 34 12 00 00 00 6F 73
```
Where:
  0x00 - Modbus broadcast address  
  0x01 - Device Probe Request  
  0x01 - Initial Stage Probe Request  
  0x03 - Next available Modbus (AquaBus) address  
  0x1234 - Apex Serial Number  
  0x000000 - unknown. Appears to be reserved and unused  
  0x736F - Modbus CRC16 checksum  
  
Since this is a new device, it takes Address and ApexSerialNumber values from the request, verifies internally that it has no record of previous connections to this Apex head unit, and send the response back. The structure of the response is as follows:
```
struct AB_PROBE_RESPONSE_PACKET
{
	byte FunctionCode;
	byte ProbeStage;
	byte hwId;
	byte hwRevision;
	byte swRevision;
	byte ABAddress;
	unsigned short ApexSerial;
	byte unknown[3];
}
```
In the response, the device keeps the original Function Code and stage Probe Stage provided in the request. It also claims the available AB address in ABAddress field and acknowledges Apex Serial number received. In addition, it also provides its hardware ID, hardware revision number and software (firmware) revision number. Apex uses this information to validate that the device can be supported by the version of Apex that it is trying to attach to. 

As of Apex firmware update 4.52_5A17, the following list of devices is known:

|Module           | HW_ID |HW_Rev|SW_Rev_Min|SW_Rev_Max|
|-----------------|-------|------|----------|----------|
|Display          |  0x1  |   1  |  10      |    11    |
|PM1              |  0x11 |   1  |   4      |     7    |
|PM2              |  0x12 |   1  |   2      |     3    |
|PM3              |  0x13 |   1  |   3      |     7    |
|ALD              |  0x14 |   1  |   7      |     7    |
|ASM              |  0x15 |   1  |   7      |     7    |
|FMM              |  0x16 |   1  |   5      |     5    |
|EB8              |  0x20 |   1  |   9      |    12    |
|WXM              |  0x21 |   1  |  10      |    11    |
|EB4              |  0x22 |   1  |   9      |    12    |
|VDM              |  0x23 |   1  |  13      |    13    |
|LSM              |  0x24 |   1  |  13      |    13    |
|EB6              |  0x25 |   1  |  11      |    12    |
|AWM              |  0x26 |   1  |   7      |     7    |
|AFS              |  0x27 |   1  |   2      |     2    |
|DOS              |  0x28 |   1  |   7      |     7    |
|WAV              |  0x29 |   3  |  16      |    16    |
|1Link            |  0x2A |   1  |   4      |     4    |

Table 3: List of Apex Modules

From this table, Apex checks hwID from the response to match one in the table. This allows Apex to choose how to handle the new device. swRevision from the response must be within SW_Rev_min and SW_Rev_max in the table. Otherwise, Apex will either attempt to update the firmware or refuse attaching the device.

#### Probe Request Stage 2

Apex may issues Probe Request Stage 2 if the device being attached has never previously registered with Apex. The Stage 2 request packet is identical to Stage 1, except for the updated ProbeStage field set to 2. The Address field is set to the same address as before. It is also sent to modbus broadcast address (0x00). The device response is also identical to the response to Stage 1 except for the ProbeStage Field.

#### Probe Request Stage 3
This is the SET stage of the new device probe. In this stage, Apex agrees to install the new device and updates it's own internal structures with information about the device. Apex sends the Stage 3 request to let the device know that it is set and allow the device to update it's internal stage as well with information about the Apex head unit it is now connected to, such as Apex serial number and AB address. Otherwise, information in Stage 3 request and response is identical to the previous two stages. The Address field is set to the same address as before. It is also sent to modbus broadcast address (0x00). After this stage, the device will be recognized by the Apex even after power loss or loss of communication.

#### Probe Request Stage 5
This is the Attach stage of the probe cycle. Apex sends this request only to previously set devices. The format of the request and the expected response repeat previous probe stages. The Address field is set to the same address as before. It is also sent to modbus broadcast address (0x00). This request tells the device that it's now attached to Apex and should expect regular communication from this point on. The response tells Apex that the device acknowledges attachment and is ready for communication.

### Reconnecting existing module to Apex
Once the device is initialized with Apex, meaning that it has gone through probe request stages 1-3, it can then be reattached to Apex with a single "Attach" request. This request assumes that the device already knows its AB address and the Apex serial number. If this is the case, the device first must respond to the initial Probe Stage 1 request, but instead of choosing the next available AB address from the request packet, it responds with its own AB address previously assigned and Apex's serial number. Upon receiving the response, Apex verifies that it already has a record of the device, and sends Probe Request Stage 5 "Attach" immediately. The request is still sent to modbus broadcast address (0x00). This request tells the device that it's now attached to Apex and should expect regular communication from this point on. The response tells Apex that the device acknowledges attachment and is ready for communication.

## Device Specific Apex Communication

This section covers communication protocols and considerations specific to particular Apex modules. For the complete list of Apex modules refer to Table 3.

### Energy Bar - 8 outlets (EB8)

For the purpose of this discussion, we will only consider two features of this device: 
- 8 controllable outlets
- Current sensing combined across all outlets

Once this module is connected to Apex and completes the initial probe sequence. It is ready to receive EB8 specific requests from Apex. EB8 supports two types of requests:
- Set outlets and get current reading
- Calibrate current sensor

| Request Type  | Description                                                                        |
|---------------|------------------------------------------------------------------------------------|
| 0x01          | Set EB8 outlets states and get current reading                                     |
| 0x03          | Calibrate (zero out) EB8 current sensor                                            |

Table 4: Available EB8 Request Types

#### Setting outlets and getting current reading
EB8 and most other Apex modules use the same function code to communicate with connected modules. The format of EB8 request is as follows:
```
struct AB_EB8_REQUEST_PACKET
{
  byte FunctionCode;
  byte RequestType;
  byte OutletStateBitmap;
  byte unknown;
}
```
In this packet:  
FunctionCode - 0x20  
RequestType - 0x01  
OutletStateBitmap - Bitmap of outlets that should be ON  
unknown - The purpose of this is not immediately clear, in some instances it seems to repeat OutletStateBitmap.  

The OutletStateBitmap byte tells EB8 which outlets should be in ON or OFF position. For example:
```
01000111
```
Sets outlets 1,2,3 and 7 to ON and outlets 4,5,6 and 8 to OFF position.

EB8 module responds to this request as follows:
```
struct AB_EB8_RESPONSE_PACKET
{
  byte FunctionCode;
  byte RequestType;
  byte OutletStateCurrent;
  byte unknown;
  unsigned short legacyCurrent;
  unsigned short frequency;
  unsigned long rawCurrent;
}
```
FunctionCode and RequestType fields repeat what was sent in the request.  
OutletStateCurrent returns the bitmap of outlets in their current state.  
unknown - the purpose of this is not currently clear  
legacyCurrent - reports current reading in legacy format  
frequency - Frequency constant to calculate Amperage. EB8 reports this as 40 (decimal) 
rawCurrent - reports current reading in full format  

It appears that EB8 uses RMS method to report total current. The formula used by apex to arrive at the final amperage number is:
```
AmpCurrent = ((sqrt(divsi3(RawCurrent,Frequency))*0x6ce8)/2^16)/10
```

#### Calibrating EB8 current sensor

EB8 supports calibration of its current sensor. The way this works is with nothing plugged into EB8 to draw any current, a user can issue a telnet command
```
eb8zero
```
This would then send a message to EB8 module to adjust the delta it uses to calculate rawCurrent. The format of the eb8zero request is as follows:
```
struct AB_EB8ZERO_REQUEST_PACKET
{
  byte FunctionCode;
  byte RequestType;
  byte Reserved[2];
}
```
Where:  
FunctionCode - 0x20  
RequestType - 0x03  
Reserved - Not used, uninitialized  

EB8 response to this request follows the standard AB_EB8_RESPONSE_PACKET format.

### Probe Module 2 (PM2) - Salinity Module

The PM2 module serves three major purposes:
- Salinity monitoring (done by conductivity probe)
- Temperature compensation for salinity monitoring (done by temperature probe)
- Six additional switches for various sensors

In this section, we will cover AquaBus PM2 protocol that support these functionalities. Once the module is connected to Apex and completes the initial probe sequence. It is ready to receive PM2 specific requests from Apex. PM2 supports three types of requests:
- Getting Initial Module Information
- Module and Probe Calibration
- Polling Module for Probe/Switch Status Data

The module also supports four types of salinity probe ranges
- Low
- Medium
- High
- Salinity

These ranges are important, because PM2 supports different polling requests for different ranges. Internally, these ranges are defined as following:
```
#define PROBE_RANGE_LOW      0
#define PROBE_RANGE_MEDIUM   1
#define PROBE_RANGE_HIGH     2
#define PROBE_RANGE_SALINITY 3
```

These values will be used throughout the protocol to indicate various range specific configuration options. You can find more information about these ranges in the official Apex PM2 manual.

While the PM2 protocol supports three types of requests as listed above, the data polling request type provides different, range specific requests as follows:

| Request Type  | Description                                                                        |
|---------------|------------------------------------------------------------------------------------|
| 0x01          | Retrieve module and probe specific configuration information from PM2 (e.g. Enabled probes, offset and scale values. |
| 0x02          | Calibrate module using data from probe calibration sequence |
| 0x03          | Poll module for probe data in PROBE_RANGE_LOW configuration|
| 0x04          | Poll module for probe data in PROBE_RANGE_MEDIUM configuration|
| 0x05          | Poll module for probe data in PROBE_RANGE_HIGH or PROBE_RANGE_SALINITY configurations|

Table 5: Available PM2 Request Types

#### Retrieving module and probe specific configuration

PM2 module stores its configuration and calibration information internally in EEPROM. In part, to avoid probe re-calibration on power cycle. Apex needs to have this information so that it can do proper, module specific data conversion for user display purposes. The format of the request is as follows:
```
struct AB_PM2_INIT_REQUEST_PACKET
{
  byte FunctionCode;
  byte RequestType;
}
```
Where:  
  FunctionCode - 0x20  
  RequestType - 0x01  
  
The module responses with a much larger packet that contains all of the internal module configuration and calibration information. The format of the response is as follows:
```
struct AB_PM2_INIT_RESPONSE_PACKET
{
  byte FunctionCode;
  byte RequestType;
  byte ProbeConfig;
  unsigned short TempProbeOffset;
  unsigned short Unknown_1;
  unsigned short Unknown_2;
  unsigned short ConductivityProbeOffset;
  unsigned short TempProbeScale;
  unsigned short Unknown_3;
  unsigned short Unknown_4;
  unsigned short ConductivityProbeScale;
}
```
Where:  
FunctionCode - 0x20  
RequestType - 0x01  
ProbeConfig - Bitfield indicates the operating range and enabled probes.  
Available PM2 probes:  
```
  #define PROBE_TYPE_None 0  
  #define PROBE_TYPE_Temp 1  
  #define PROBE_TYPE_Cond 0x40  
```
For example, PM2 module with enabled conductivity probe operating in SALINITY range would have ProbeConfig - PROBE_TYPE_Cond | (PROBE_RANGE_SALINITY < 1) or ProbeConfig = 0x46. A module with enabled conductivity and temperature probes operating in SALINITY range would have ProbeConfig = PROBE_TYPE_Cond | PROBE_TYPE_Temp | (PROBE_RANGE_SALINITY < 1) or ProbeConfig = 0x47 and so on.  
  
TempProbeOffset - Temperature Probe Offset. As described in the module manual, the value is used to adjust the temperature probe reading. This is a signed 16 bit field. An offset of '-14' is represented as value 0xFFF2.  
  
Unknown_1 - Currently unknown. Example seen transmitted by PM2 is 0xFFF9, likely another negative value -7.  
  
Unknown_2 - Currently unknown. Example seen transmitted by PM2 is 0xFFF0, likely another negative value -16.  
  
ConductivityProbeOffset - Conductivity Probe Offset. As described in the module manual, the value is used to adjust the conductivity probe reading. The offset of 0x0238 is shown in Apex probe offset configuration option as 568.  
  
TempProbeScale - Temperature Probe Scale. Seems to always be set to 0x1000. The value shows in Apex Web interface under Probe configuration as "Scale: 1.000"  
  
Unknown_3 - Currently unknown. Example seen transmitted by PM2 is 0x0B55.  
  
Unknown_4 - Currently unknown. Example seen transmitted by PM2 is 0x0E60.  
  
ConductivityProbeScale - Conductivity Probe Scale. Example seen transmitted by PM2 is 0x1086. The value shows in Apex Web interface under Probe configuration as "Offset: 1.086"  

#### Calibrating PM2 module

Apex support automated and manual probe calibration via either the display or the web interface. The calibration request is as follows:

```
struct AB_PM2_CALIBRATE_REQUEST_PACKET
{
  byte FunctionCode;
  byte RequestType;
  byte ProbeConfig;
  unsigned short TempProbeOffset;
  unsigned short Unknown_1;
  unsigned short Unknown_2;
  unsigned short ConductivityProbeOffset;
  unsigned short TempProbeScale;
  unsigned short Unknown_3;
  unsigned short Unknown_4;
  unsigned short ConductivityProbeScale;
}
```

Where:  
FunctionCode - 0x20  
RequestType - 0x02  
ProbeConfig - Bitfield indicates the operating range and enabled probes.   
Available PM2 probes:  
```
  #define PROBE_TYPE_None 0  
  #define PROBE_TYPE_Temp 1  
  #define PROBE_TYPE_Cond 0x40  
```
For example, PM2 module with enabled conductivity probe operating in SALINITY range would have ProbeConfig - PROBE_TYPE_Cond | (PROBE_RANGE_SALINITY < 1) or ProbeConfig = 0x46. A module with enabled conductivity and temperature probes operating in SALINITY range would have ProbeConfig = PROBE_TYPE_Cond | PROBE_TYPE_Temp | (PROBE_RANGE_SALINITY < 1) or ProbeConfig = 0x47 and so on.  
  
TempProbeOffset - Temperature Probe Offset. As described in the module manual, the value is used to adjust the temperature probe reading. This is a signed 16 bit field. An offset of '-14' is represented as value 0xFFF2.  
  
Unknown_1 - Currently unknown. Example seen transmitted by PM2 is 0xFFF9, likely another negative value -7.  
  
Unknown_2 - Currently unknown. Example seen transmitted by PM2 is 0xFFF0, likely another negative value -16.  
  
ConductivityProbeOffset - Conductivity Probe Offset. As described in the module manual, the value is used to adjust the conductivity probe reading. The offset of 0x0238 is shown in Apex probe offset configuration option as 568.  
  
TempProbeScale - Temperature Probe Scale. Seems to always be set to 0x1000. The value shows in Apex Web interface under Probe configuration as "Scale: 1.000"  
  
Unknown_3 - Currently unknown. Example seen transmitted by PM2 is 0x0B55.  
  
Unknown_4 - Currently unknown. Example seen transmitted by PM2 is 0x0E60.  
  
ConductivityProbeScale - Conductivity Probe Scale. Example seen transmitted by PM2 is 0x1086. The value shows in Apex Web interface under Probe configuration as "Offset: 1.086"  

The response structure is identical to the AB_PM2_INIT_RESPONSE_PACKET:

```
struct AB_PM2_CALIBRATE_RESPONSE_PACKET
{
  byte FunctionCode;
  byte RequestType;
  byte ProbeConfig;
  unsigned short TempProbeOffset;
  unsigned short Unknown_1;
  unsigned short Unknown_2;
  unsigned short ConductivityProbeOffset;
  unsigned short TempProbeScale;
  unsigned short Unknown_3;
  unsigned short Unknown_4;
  unsigned short ConductivityProbeScale;
}
```
The content is similar to that of AB_PM2_INIT_RESPONSE_PACKET except RequestType, which is set to 2.

#### Polling for probe and switch data

The request structure is identical for all four types of probe ranges:
```
struct AB_PM2_DATA_REQUEST_PACKET
{
	byte FunctionCode;
	byte RequestType;
}
```
Where:  
  FunctionCode - 0x20  
  RequestType - 0x3 if operating in PROBE_RANGE_LOW mode  
                0x4 if operating in PROBE_RANGE_MEDIUM mode  
                0x5 if operating in PROBE_RANGE_HIGH or PROBE_RANGE_SALINITY mode  

The module response with the following:
```
struct AB_PM2_DATA_RESPONSE_PACKET
{
  byte FunctionCode;
  byte RequestType;
  byte ProbeConfig;
  unsigned short CondReading;
  unsigned short TempReading;
  unsigned short Unknown;
  unsigned short SwitchState;
}
```
Where:  
FunctionCode - 0x20  
RequestType - 0x3, 0x4 or 0x5  
ProbeConfig - Bitfield indicates the operating range and enabled probes.  
Available PM2 probes:  
```
#define PROBE_TYPE_None 0  
#define PROBE_TYPE_Temp 1  
#define PROBE_TYPE_Cond 0x40  
```
For example, PM2 module with enabled conductivity probe operating in SALINITY range would have ProbeConfig - PROBE_TYPE_Cond | (PROBE_RANGE_SALINITY < 1) or ProbeConfig = 0x46. A module with enabled conductivity and temperature probes operating in SALINITY range would have ProbeConfig = PROBE_TYPE_Cond | PROBE_TYPE_Temp | (PROBE_RANGE_SALINITY < 1) or ProbeConfig = 0x47 and so on.
  
CondReading - Conductivity reading. Example seen transmitted by PM2 is 0x348A. This corresponds to salinity reading of "48"  
  
TempReading - Temperature reading. Example seen transmitted by PM2 is 0x2244. This corresponds to temperature reading of "74.7f"  
  
Unknown - Unknown and appears to not be used  
  
SwitchState - bitfield of current state of the 6 switches. For example, 0x36 or binary 00110110 would indicate that switches 2,3,5 and 6 are ON and switches 1 and 4 are OFF.  

#### Emulating PM2 module for Switches functionality only

It is possible to emulate the PM2 module without any probes for the purpose of adding the 6 sensor switches to Apex. At the minimum you would have to handle AB_PM2_INIT_REQUEST_PACKET and AB_PM2_DATA_REQUEST_PACKET. In the AB_PM2_INIT_REQUEST_PACKET handler, the response should set ProbeCofig to zero to indicate that no probes are enabled and operating in Low Range mode. All other field can be set to zero. In the AB_PM2_DATA_REQUEST_PACKET handler, the response should repeat the value of ProbeConfig field and set the SwitchState field appropriately. All other fields can be set to zero.

### Probe Module 1 (PM1) - pH/ORP Module

The PM1 module serves three major purposes:
- Provides additional pH/ORP monitoring
- Temperature compensation
- Six additional switches for various sensors

In this section, we will cover AquaBus PM1 protocol that support these functionalities. Once the module is connected to Apex and completes the initial probe sequence. It is ready to receive PM1 specific requests from Apex. PM1 supports three types of requests:
- Getting Initial Module Information
- Module and Probe Calibration
- Polling Module for Probe/Switch Status Data

| Request Type  | Description                                                                        |
|---------------|------------------------------------------------------------------------------------|
| 0x01          | Retrieve module and probe specific configuration information from PM1 (e.g. Enabled probes, offset and scale values. |
| 0x02          | Calibrate module using data from probe calibration sequence |
| 0x03          | Poll module for probe data |

Table 6: Available PM1 Request Types

#### Retrieving module and probe specific configuration

PM1 module stores its configuration and calibration information internally in EEPROM. In part, to avoid probe re-calibration on power cycle. Apex needs to have this information so that it can do proper, module specific data conversion for user display purposes. The format of the request is as follows:
```
struct AB_PM1_INIT_REQUEST_PACKET
{
  byte FunctionCode;
  byte RequestType;
}
```
Where:  
  FunctionCode - 0x20  
  RequestType - 0x01  
  
The module responses with a much larger packet that contains all of the internal module configuration and calibration information. The format of the response is as follows:
```
struct AB_PM1_INIT_RESPONSE_PACKET
{
  byte FunctionCode;
  byte RequestType;
  byte ProbeConfig;
  unsigned short pH_ProbeOffset;
  unsigned short TempProbeOffset;
  unsigned short ORPProbeOffset;
  unsigned short pH_ProbeScale;
  unsigned short TempProbeScale;
  unsigned short ORPProbeScale;
  unsigned short Unknown_1;
  unsigned short Unknown_2;
}
```
Where:  
FunctionCode - 0x20  
RequestType - 0x01  
ProbeConfig - Bitfield indicates enabled probes.  
Available PM1 probes:  
```
  #define PROBE_TYPE_None 0  
  #define PROBE_TYPE_Temp 1  
  #define PROBE_TYPE_pH   2
  #define PROBE_TYPE_ORP  4 
```
For example, PM1 module with enabled pH probe and Temperature probe would have ProbeConfig - PROBE_TYPE_pH |  PROBE_TYPE_Temp or ProbeConfig = 0x3. A module with enabled ORP and temperature probes operating would have ProbeConfig = PROBE_TYPE_ORP | PROBE_TYPE_Temp | or ProbeConfig = 0x5 and so on.  

pH_ProbeOffset - pH Probe Offset. The value is used to adjust the pH probe reading. This is a signed 16 bit field.  

TempProbeOffset - Temperature Probe Offset. As described in the module manual, the value is used to adjust the temperature probe reading. This is a signed 16 bit field. An offset of '-14' is represented as value 0xFFF2.  

ORPProbeOffset - ORP Probe Offset. The value is used to adjust the ORP probe reading. This is a signed 16 bit field.  

pH_ProbeScale - pH Probe Scale.  

TempProbeScale - Temperature Probe Scale. Seems to always be set to 0x1000. The value shows in Apex Web interface under Probe configuration as "Scale: 1.000"  

ORPProbeScale - ORP Probe Scale.  

Unknown_1 - Currently unknown.  
  
Unknown_2 - Currently unknown.  

#### Calibrating PM1 module

Apex support automated and manual probe calibration via either the display or the web interface. The calibration request is as follows:

```
struct AB_PM1_CALIBRATE_REQUEST_PACKET
{
  byte FunctionCode;
  byte RequestType;
  byte ProbeConfig;
  unsigned short pH_ProbeOffset;
  unsigned short TempProbeOffset;
  unsigned short ORPProbeOffset;
  unsigned short pH_ProbeScale;
  unsigned short TempProbeScale;
  unsigned short ORPProbeScale;
  unsigned short Unknown_1;
  unsigned short Unknown_2;
}
```

Where:  
FunctionCode - 0x20  
RequestType - 0x02  
ProbeConfig - Bitfield indicates enabled probes.   
Available PM1 probes:  
```
  #define PROBE_TYPE_None 0  
  #define PROBE_TYPE_Temp 1  
  #define PROBE_TYPE_pH   2
  #define PROBE_TYPE_ORP  4 
```
For example, PM1 module with enabled pH probe and Temperature probe would have ProbeConfig - PROBE_TYPE_pH |  PROBE_TYPE_Temp or ProbeConfig = 0x3. A module with enabled ORP and temperature probes operating would have ProbeConfig = PROBE_TYPE_ORP | PROBE_TYPE_Temp | or ProbeConfig = 0x5 and so on.  

pH_ProbeOffset - pH Probe Offset. The value is used to adjust the pH probe reading. This is a signed 16 bit field.  

TempProbeOffset - Temperature Probe Offset. As described in the module manual, the value is used to adjust the temperature probe reading. This is a signed 16 bit field. An offset of '-14' is represented as value 0xFFF2.  

ORPProbeOffset - ORP Probe Offset. The value is used to adjust the ORP probe reading. This is a signed 16 bit field.  

pH_ProbeScale - pH Probe Scale.  

TempProbeScale - Temperature Probe Scale. Seems to always be set to 0x1000. The value shows in Apex Web interface under Probe configuration as "Scale: 1.000"  

ORPProbeScale - ORP Probe Scale.  

Unknown_1 - Currently unknown.  
  
Unknown_2 - Currently unknown.   

The response structure is identical to the AB_PM1_INIT_RESPONSE_PACKET:

```
struct AB_PM1_CALIBRATE_RESPONSE_PACKET
{
  byte FunctionCode;
  byte RequestType;
  byte ProbeConfig;
  unsigned short pH_ProbeOffset;
  unsigned short TempProbeOffset;
  unsigned short ORPProbeOffset;
  unsigned short pH_ProbeScale;
  unsigned short TempProbeScale;
  unsigned short ORPProbeScale;
  unsigned short Unknown_1;
  unsigned short Unknown_2;
}
```
The content is similar to that of AB_PM1_INIT_RESPONSE_PACKET except RequestType, which is set to 2.

#### Polling for probe and switch data

The request structure is for the data polling request:
```
struct AB_PM1_DATA_REQUEST_PACKET
{
	byte FunctionCode;
	byte RequestType;
}
```
Where:  
  FunctionCode - 0x20  
  RequestType - 0x3  

The module response with the following:
```
struct AB_PM1_DATA_RESPONSE_PACKET
{
  byte FunctionCode;
  byte RequestType;
  byte ProbeConfig;
  unsigned short pH_Reading;
  unsigned short TempReading;
  unsigned short ORPReading;
  unsigned short SwitchState;
}
```
Where:  
FunctionCode - 0x20  
RequestType - 0x3  
ProbeConfig - Bitfield indicates enabled probes.  
Available PM1 probes:  
```
#define PROBE_TYPE_None 0  
#define PROBE_TYPE_Temp 1  
#define PROBE_TYPE_pH   2
#define PROBE_TYPE_ORP  4 
```
For example, PM1 module with enabled pH probe and Temperature probe would have ProbeConfig - PROBE_TYPE_pH |  PROBE_TYPE_Temp or ProbeConfig = 0x3. A module with enabled ORP and temperature probes operating would have ProbeConfig = PROBE_TYPE_ORP | PROBE_TYPE_Temp | or ProbeConfig = 0x5 and so on.  
  
pH_Reading - pH probe reading. Example seen transmitted by PM1 is 0x348A. This corresponds to pH reading of "3.52" with offset of "-16" and scale "0.238"
  
TempReading - Temperature probe reading. Example seen transmitted by PM1 is 0x2244. This corresponds to temperature reading of "75.3f" with offset of "-7"
  
ORPReading - ORP probe reading. Example seen transmitted by PM1 is 0x4018. This corresponds to ORP reading of "83" with offset of "-16" and scale of "0.b55"  
  
SwitchState - bitfield of current state of the 6 switches. For example, 0x36 or binary 00110110 would indicate that switches 2,3,5 and 6 are ON and switches 1 and 4 are OFF.  

#### Emulating PM1 module for Switches functionality only

It is possible to emulate the PM1 module without any probes for the purpose of adding the 6 sensor switches to Apex. At the minimum you would have to handle AB_PM1_INIT_REQUEST_PACKET and AB_PM2_DATA_REQUEST_PACKET. In the AB_PM1_INIT_REQUEST_PACKET handler, the response should set ProbeCofig to zero to indicate that no probes are enabled and operating in Low Range mode. All other field can be set to zero. In the AB_PM1_DATA_REQUEST_PACKET handler, the response should repeat the value of ProbeConfig field and set the SwitchState field appropriately. All other fields can be set to zero.  

### Probe Module 3 (PM3) - Dissolved Oxygen Module

The PM3 module serves three major purposes:
- Provides Dissolved Oxygen monitoring
- Temperature compensation
- Six additional switches for various sensors

In this section, we will cover AquaBus PM3 protocol that support these functionalities. Once the module is connected to Apex and completes the initial probe sequence. It is ready to receive PM3 specific requests from Apex. PM1 supports three types of requests:
- Getting Initial Module Information
- Module and Probe Calibration
- Polling Module for Probe/Switch Status Data  

The module also supports four types of salinity probe ranges
- SAT
- PPM

Internally, these ranges are defined as following:
```
#define PROBE_RANGE_SAT      0
#define PROBE_RANGE_PPM      1
```

These values will be used throughout the protocol to indicate various range specific configuration options. You can find more information about these ranges in the official Apex PM3 manual.

| Request Type  | Description                                                                        |
|---------------|------------------------------------------------------------------------------------|
| 0x01          | Retrieve module and probe specific configuration information from PM3 (e.g. Enabled probes, offset and scale values. |
| 0x02          | Calibrate module using data from probe calibration sequence |
| 0x03          | Poll module for probe data |

Table 7: Available PM3 Request Types

#### Retrieving module and probe specific configuration

PM3 module stores its configuration and calibration information internally in EEPROM. In part, to avoid probe re-calibration on power cycle. Apex needs to have this information so that it can do proper, module specific data conversion for user display purposes. The format of the request is as follows:
```
struct AB_PM3_INIT_REQUEST_PACKET
{
  byte FunctionCode;
  byte RequestType;
}
```
Where:  
  FunctionCode - 0x20  
  RequestType - 0x01  
  
The module responses with a much larger packet that contains all of the internal module configuration and calibration information. The format of the response is as follows:
```
struct AB_PM3_INIT_RESPONSE_PACKET
{
  byte FunctionCode;
  byte RequestType;
  byte ProbeConfig;
  unsigned short DO_ProbeOffset;
  unsigned short TempProbeOffset;
  unsigned short Reserved_1;
  unsigned short DO_ProbeScale;
  unsigned short TempProbeScale;
  unsigned short Reserved_2;
  unsigned short Unknown_1;
  unsigned short Unknown_2;
}
```
Where:  
FunctionCode - 0x20  
RequestType - 0x01  
ProbeConfig - Bitfield indicates enabled probes.  
Available PM3 probes:  
```
  #define PROBE_TYPE_None 0  
  #define PROBE_TYPE_Temp 1  
  #define PROBE_TYPE_DO   8 
```
For example, PM3 module with enabled DO probe operating in PPM range would have ProbeConfig - PROBE_TYPE_DO | (PROBE_RANGE_PPM < 1) or ProbeConfig = 0x10. A module with enabled DO and temperature probes operating in PPM range would have ProbeConfig = PROBE_TYPE_DO | PROBE_TYPE_Temp | (PROBE_RANGE_PPM < 1) or ProbeConfig = 0x11 and so on.  

DO_ProbeOffset - DO Probe Offset. The value is used to adjust the DO probe reading. This is a signed 16 bit field.  

TempProbeOffset - Temperature Probe Offset. As described in the module manual, the value is used to adjust the temperature probe reading. This is a signed 16 bit field. An offset of '-14' is represented as value 0xFFF2.  

Reserved_1 - Not used.  

DO_ProbeScale - DO Probe Scale.  

TempProbeScale - Temperature Probe Scale. Seems to always be set to 0x1000. The value shows in Apex Web interface under Probe configuration as "Scale: 1.000"  

Reserved_2 - Not used.  

Unknown_1 - Currently unknown.  
  
Unknown_2 - Currently unknown.  

#### Calibrating PM3 module

Apex support automated and manual probe calibration via either the display or the web interface. The calibration request is as follows:

```
struct AB_PM3_CALIBRATE_REQUEST_PACKET
{
  byte FunctionCode;
  byte RequestType;
  byte ProbeConfig;
  unsigned short DO_ProbeOffset;
  unsigned short TempProbeOffset;
  unsigned short Reserved_1;
  unsigned short DO_ProbeScale;
  unsigned short TempProbeScale;
  unsigned short Reserved_2;
  unsigned short Unknown_1;
  unsigned short Unknown_2;
}
```

Where:  
FunctionCode - 0x20  
RequestType - 0x02  
ProbeConfig - Bitfield indicates enabled probes.   
Available PM2 probes:  
```
  #define PROBE_TYPE_None 0  
  #define PROBE_TYPE_Temp 1  
  #define PROBE_TYPE_DO   8 
```
For example, PM3 module with enabled DO probe operating in PPM range would have ProbeConfig - PROBE_TYPE_DO | (PROBE_RANGE_PPM < 1) or ProbeConfig = 0x10. A module with enabled DO and temperature probes operating in PPM range would have ProbeConfig = PROBE_TYPE_DO | PROBE_TYPE_Temp | (PROBE_RANGE_PPM < 1) or ProbeConfig = 0x11 and so on.  

DO_ProbeOffset - DO Probe Offset. The value is used to adjust the DO probe reading. This is a signed 16 bit field.  

TempProbeOffset - Temperature Probe Offset. As described in the module manual, the value is used to adjust the temperature probe reading. This is a signed 16 bit field. An offset of '-14' is represented as value 0xFFF2.  

Reserved_1 - Not used.  

DO_ProbeScale - DO Probe Scale.  

TempProbeScale - Temperature Probe Scale. Seems to always be set to 0x1000. The value shows in Apex Web interface under Probe configuration as "Scale: 1.000"  

Reserved_2 - Not used.  

Unknown_1 - Currently unknown.  
  
Unknown_2 - Currently unknown.  

The response structure is identical to the AB_PM2_INIT_RESPONSE_PACKET:

```
struct AB_PM2_CALIBRATE_RESPONSE_PACKET
{
  byte FunctionCode;
  byte RequestType;
  byte ProbeConfig;
  unsigned short DO_ProbeOffset;
  unsigned short TempProbeOffset;
  unsigned short Reserved_1;
  unsigned short DO_ProbeScale;
  unsigned short TempProbeScale;
  unsigned short Reserved_2;
  unsigned short Unknown_1;
  unsigned short Unknown_2;
}
```
The content is similar to that of AB_PM3_INIT_RESPONSE_PACKET except RequestType, which is set to 2.

#### Polling for probe and switch data

The request structure is for the data polling request:
```
struct AB_PM3_DATA_REQUEST_PACKET
{
	byte FunctionCode;
	byte RequestType;
}
```
Where:  
  FunctionCode - 0x20  
  RequestType - 0x3  

The module response with the following:
```
struct AB_PM3_DATA_RESPONSE_PACKET
{
  byte FunctionCode;
  byte RequestType;
  byte ProbeConfig;
  unsigned short DO_Reading;
  unsigned short TempReading;
  unsigned short Reserved;
  unsigned short SwitchState;
}
```
Where:  
FunctionCode - 0x20  
RequestType - 0x3  
ProbeConfig - Bitfield indicates enabled probes.  
Available PM3 probes:  
```
#define PROBE_TYPE_None 0  
#define PROBE_TYPE_Temp 1  
#define PROBE_TYPE_DO   8
```
For example, PM3 module with enabled DO probe operating in PPM range would have ProbeConfig - PROBE_TYPE_DO | (PROBE_RANGE_PPM < 1) or ProbeConfig = 0x10. A module with enabled DO and temperature probes operating in PPM range would have ProbeConfig = PROBE_TYPE_DO | PROBE_TYPE_Temp | (PROBE_RANGE_PPM < 1) or ProbeConfig = 0x11 and so on.  

DO_Reading - DO probe reading. Example seen transmitted by PM3 is 0x348A. This corresponds to DO reading of "87" with offset of "-16" and scale "0.238"
  
TempReading - Temperature probe reading. Example seen transmitted by PM3 is 0x2244. This corresponds to temperature reading of "75.4f" with offset of "-7"
  
Reserved - not used    
  
SwitchState - bitfield of current state of the 6 switches. For example, 0x36 or binary 00110110 would indicate that switches 2,3,5 and 6 are ON and switches 1 and 4 are OFF.  

#### Emulating PM3 module for Switches functionality only

It is possible to emulate the PM3 module without any probes for the purpose of adding the 6 sensor switches to Apex. At the minimum you would have to handle AB_PM3_INIT_REQUEST_PACKET and AB_PM3_DATA_REQUEST_PACKET. In the AB_PM3_INIT_REQUEST_PACKET handler, the response should set ProbeCofig to zero to indicate that no probes are enabled and operating in Low Range mode. All other field can be set to zero. In the AB_PM3_DATA_REQUEST_PACKET handler, the response should repeat the value of ProbeConfig field and set the SwitchState field appropriately. All other fields can be set to zero.

