# Introduction
The following descriptions address:
1. Basic settings for your configuration
2. Advanced settings for other frameworks (e.g. Tasmota, Home Assistant, Homie)
3. Integration examples

# Basic settings
The basic settings will be adapted in three files:   
 - [support.h](src/support.h) for general settings related to WiFi, MQTT, OTA and the external temperature sensor DS18x20
 - [MHI-AC-Ctrl.h](src/MHI-AC-Ctrl.h) for input / output settings (i.e. topic / payload text)
 - [MHI-AC-Ctrl-core.h](src/MHI-AC-Ctrl-core.h) for settings related to the behaviour of MHI-AC-Ctrl (e.g. selection of operating data)

## WiFi ([support.h](src/support.h))
WiFi STA mode is supported.

### WiFi Settings (SSID, Password, hostname)
Adapt the SSID and the password. Changing the hostname is usually not required.
```
#define WIFI_SSID "your SSID"
#define WIFI_PASSWORD "your WiFi password"
#define HOSTNAME "MHI-AC-Ctrl"
```
 Changing the hostname is required when multiple ACs should be supported. E.g. replace "MHI-AC-Ctrl" by "Living-Room-MHI-AC-Ctrl"

Per default ESP8266 uses the first WiFi access point with matching SSID. This behaviour can be changed.
```
#define UseStrongestAP true             // when false then the first WiFi access point with matching SSID found is used.
                                        // when true then the strongest WiFi access point with matching SSID found is used,
                                        // Implemented, but not completely tested: The network is rescanned every 12 minutes for the strongest AP.
```
 
## MQTT ([support.h](src/support.h))
The program uses the [MQTT client library](https://github.com/knolleary/pubsubclient) from Nick O'Leary (knolleary).
If you are not familiar with MQTT you find on the Internet endless numbers of descriptions and tutorials. My favorites are [here](https://www.hivemq.com/blog/how-to-get-started-with-mqtt/) and [here](https://www.heise.de/developer/artikel/Kommunikation-ueber-MQTT-3238975.html).
I recommend [MQTT Explorer](http://mqtt-explorer.com/) a great all-round MQTT client that provides a structured topic overview for the first steps.

### MQTT General Settings (broker, port, account data)
Adapt the server (broker) name and the port if needed:

```
#define MQTT_SERVER "MQTT broker name"  // broker name or IP address of the broker
#define MQTT_PORT 1883                  // port number used by the broker
```

If you want to use MQTT authentication enter user name and password:
```
#define MQTT_USER ""          // if authentication is not used, leave it empty
#define MQTT_PASSWORD ""      // if authentication is not used, leave it empty
```
note: TLS/SSL is not supported

The following sections show the configuration for the MQTT pathes.
### MQTT status
The topic level for status information from the AC consists of the MQTT_PREFIX and the function name separated by a slash, e.g. *MHI-AC-Ctrl/Power*.   
For writes the topic level consists of the MQTT_PREFIX, the prefix for set commands (MQTT_SET_PREFIX) and the function name, each separated by a slash, e.g. *MHI-AC-Ctrl/set/Power*.  
You can change MQTT_PREFIX and MQTT_SET_PREFIX, default is
```
#define MQTT_PREFIX HOSTNAME "/"           // basic prefix used for publishing AC data
#define MQTT_SET_PREFIX MQTT_PREFIX "set/" // prefix for subscribing set commands
```
Please pay attention to the case sensitivity.

The following status data is available (prefix is not listed)
They are only published when there is a change of the message. The retained flag is 'true'.
When writing data, the retain flag shall be 'false'!

topic|r/w|value|comment
-----|---|-----|------
Power|r/w|"On", "Off"|
Mode|r/w|"Auto", "Dry", "Cool", "Fan", "Heat" and "Off"|"Off" is only supported when option [POWERON_WHEN_CHANGING_MODE](#behaviour-when-changing-ac-mode-supporth) is selected
Tsetpoint|r/w|18 ... 30|Target room temperature (integer) in °C
Fan|r/w|1 ... 4|Fan level <sup>1</sup>
Vanes|r/w|1,2,3,4,"Swing","?"|Vanes up/down position <sup>2</sup>
Troom|r/w|0 ... 35|Room temperature (float) in °C.  <sup>3</sup>
Tds1820|r|-40 .. 85|Temperature (float) by the additional DS18x20 sensor in °C  <sup>4</sup>
Errorcode|r|0 .. 255|error code (unsigned int)
ErrOpData|w||triggers the reading of last error operating data

<sup>1</sup> Fan auto is not supported via the SPI.   
<sup>2</sup> When the last command was received via the infrared remote control then the Vanes status is unknown and the "?" is published.   
<sup>3</sup> Please compare with section [Room temperature](#room-temperature) for writing.   
<sup>4</sup> Only available when a DS18x20 is connected, please see the description in [Hardware.md](Hardware.md) and in section [External Temperature Sensor Settings](#external-temperature-sensor-settings-supporth).

Additionally, the following program status topics are available:

topic    |r/w| value |comment
---------|---|---|---
cmd_received|r|"o.k.", "unknown command" or "invalid parameter"|feedback for last set command
connected|r  |0, 1|MQTT connection status to broker
fMISO    |r  |unsigned integer|frequency of the MISO pin in Hz during boot
fMOSI    |r  |unsigned integer|frequency of the MOSI pin in Hz during boot
fSCK     |r  |unsigned integer|frequency of the SCK pin in Hz during boot
reset|w|"reset"|resets the ESP8266
RSSI     |r  |integer         |WiFI RSSI / signal Strength in dBm after MQTT (re-)connect
Version  |r  |string          |Version number of MHI-AC-Ctrl
WIFI_LOST|r  |integer         |number of lost WiFi connections since last reset
MQTT_LOST|r  |integer         |number of lost MQTT connections since last reset

note: The topic and the payload text of the status data is adaptable by defines in [MHI-AC-Ctrl.h](src/MHI-AC-Ctrl.h)

### MQTT operating data
MHI-AC-Ctrl can provide operating data of the indoor and outdoor unit. This data is not needed for daily use, but might be interesting in specific use cases. Operating data is only published when there is a change of the content. The retained flag is 'true'.
The path to the operating data topic can be adapted.

```
#define MQTT_OP_PREFIX "OpData/"    // prefix for publishing operating data
```

Without changes of the path, subscribe to *MHI-AC-Ctrl/OpData/#* for receiving all operating data. Please see section [Operating data](#operating-data-mhi-ac-ctrl-coreh) to find all supported operating data.

note: The topic and the payload text is adaptable by defines in [MHI-AC-Ctrl.h](src/MHI-AC-Ctrl.h)


### MQTT last error operating data
When an error in the AC occurs, some operating data of this error are stored in the AC and can be read out.
The path to the operating data topic is defined in   
```
#define MQTT_OP_PREFIX "ErrOpData/"    // prefix for publishing operating data from last error
```
The readout of last error operating data is triggered by publishing *ErrOpData* to topic ErrOpData. Not all of the operating data from section [Operating data](#operating-data-mhi-ac-ctrl-coreh) might be available as last error operating data.

note: The topic and the payload text is adaptable by defines in [MHI-AC-Ctrl.h](src/MHI-AC-Ctrl.h)

## OTA Settings ([support.h](src/support.h))
OTA (Over the Air) update is the process of loading the firmware to ESP module using Wi-Fi connection rather than a serial port.
The OTA hostname can be adapted, per default it is the hostname used by WiFi.
```
#define OTA_HOSTNAME HOSTNAME     // default for the OTA_HOSTNAME is the HOSTNAME
#define OTA_PASSWORD ""           // Enter an OTA password if required
```
## External Temperature Sensor Settings ([support.h](src/support.h))
When an external temperature sensor is connected, you can configure the pin where DQ of the the DS18x20 is connected, default is Pin 4 (D2)
and how often the sensor should be read. To use reading of the external sensor you must adapt the TEMP_MEASURE_PERIOD.

```
#define TEMP_MEASURE_PERIOD 0   // period in seconds for temperature measurement with the external DS18x20 temperature sensor
                                // set to e.g. 30 to read the sensor every 30 seconds. 
#define ONE_WIRE_BUS 4          // D2, PIN for connecting temperature sensor DS18x20 DQ pin
```
note: The according libraries [OneWire](https://www.pjrc.com/teensy/td_libs_OneWire.html) and [DallasTemperature](https://github.com/milesburton/Arduino-Temperature-Control-Library) are only used if TEMP_MEASURE_PERIOD > 0.

If the DS18x20 should replace the room temperature sensor of the AC, you have to confiure it as described in the next clause.

## Room temperature
Usage of the room temperature sensor inside the AC is the default, but instead you can use the DS18x20 sensor on the MHI-AC-Ctrl board or the received temperature via the MQTT topic Troom.
```
//#define ROOM_TEMP_DS18X20           // use room temperature from DS18x20

//#define ROOM_TEMP_MQTT              // use room temperature from received MQTT topic
#define ROOM_TEMP_MQTT_TIMEOUT  20    // only considered if ROOM_TEMP_MQTT is defined
                                      // time in seconds, after this time w/o receiving a valid room temperature
                                      // via MQTT fallback to IU temperature sensor value
```
If the timeout occurs, and the system falls back to IU temperature, it will return to using the MQTT room temperature if the MQTT messages resume

## Behaviour when changing AC mode ([support.h](src/support.h))
Per default the power on/off state is not changed, when you change the AC mode (e.g. heat, dry, cold etc.).
But when you uncomment the following line, then the AC is switched on, once you change the AC mode and switched off if you publish "Off" to Mode (instead of Power). This beahviour is requested for use with [Home Assistant](https://www.home-assistant.io/).
```
//#define POWERON_WHEN_CHANGING_MODE true           // uncomment it to switch on the AC when the mode (heat, cool, dry etc.) is changed
```

# Advanced settings

## Topic and payload text ([MHI-AC-Ctrl.h](src/MHI-AC-Ctrl.h))
All topic and payload text is included in defines, e.g. change
```
#define PAYLOAD_POWER_ON "On"
```
to
```
#define PAYLOAD_POWER_ON "on"
```
if your framework prefers lower case. These topics and payloads are used for MQTT topics and payloads.

## Operating data ([MHI-AC-Ctrl-core.h](src/MHI-AC-Ctrl-core.h))
Currently the following operating data in double quotes are supported
```
  { 0xc0, 0x02},  //  1 "MODE"
  { 0xc0, 0x05},  //  2 "SET-TEMP" [°C]
  { 0xc0, 0x80},  //  3 "RETURN-AIR" [°C]
  { 0xc0, 0x81},  //  5 "THI-R1" [°C]
  { 0x40, 0x81},  //  6 "THI-R2" [°C]
  { 0xc0, 0x87},  //  7 "THI-R3" [°C]
  { 0xc0, 0x1f},  //  8 "IU-FANSPEED"
  { 0xc0, 0x1e},  // 12 "TOTAL-IU-RUN" [h]
  { 0x40, 0x80},  // 21 "OUTDOOR" [°C]
  { 0x40, 0x82},  // 22 "THO-R1" [°C]
  { 0x40, 0x11},  // 24 "COMP" [Hz]
  { 0x40, 0x85},  // 27 "TD" [A]
  { 0x40, 0x90},  // 29 "CT" [A]
  { 0x40, 0xb1},  // 32 "TDSH" [°C]
  { 0x40, 0x7c},  // 33 "PROTECTION-No"
  { 0x40, 0x1f},  // 34 "OU-FANSPEED"
  { 0x40, 0x0c},  // 36 "DEFROST"
  { 0x40, 0x1e},  // 37 "TOTAL-COMP-RUN" [h]
  { 0x40, 0x13},  // 38 "OU-EEV" [Puls]
  { 0x00, 0x00},  // dummy
```

note: If you are not interested in these operating modes (e.g. to reduce the MQTT load) you can comment out the according lines. But at least the dummy line must be available.
For THI-R2, THO-R1 and TDSH the formula for calculation is not yet known.
You can find some hints related to the meaning of the operating data [here](https://www.hrponline.co.uk/media/pdf/41/42/ed/Beijer-Ref-Service-Support-Handbook-19cWKESQUhzVIy5.pdf#page=7).    

hint: The error operating data is usually a sub-set of the operating data above. If user requests error operating data, all available error operating data is provided independent from the list above.


## Access Speed ([MHI-AC-Ctrl-core.h](src/MHI-AC-Ctrl-core.h ))
With the following parameter you can determine the number of frames used for writing to the AC. It is used
for writing status values and for reading (error) operating data.
```
#define NoFramesPerPacket 20  // number of frames/packet, must be an even number
```
A high number increases the probability that an access is successful also in the case of noisy SPI communication,
but it reduces the access speed and the frequency of operating data reads.
Since this parameter also influences the time available for the processing of AC commands, the parameter should not be chosen too small.

## MHI-AC-Ctrl partitioning
MHI-AC-Ctrl-core implements the core functions (SPI read/write, communication with the wrapper).
Wifi, MQTT, OTA and DS18x20 stuff is located in support.h and support.cpp.
MHI-AC-Ctrl.ino and MHI-AC-Ctrl.h contain the wrapper for MHI-AC-Ctrl-core.cpp and support.cpp.

### MHI-AC-Ctrl-core.h and MHI-AC-Ctrl-core.cpp
Usually it should be not touched, only configured via [MHI-AC-Ctrl-core.h](src/MHI-AC-Ctrl-core.h).
AC status information change will trigger the callback function cbiStatusFunction located in [MHI-AC-Ctrl.ino](src/MHI-AC-Ctrl.ino)
It is controlled via the functions:
```
void init();                          // initialization called once after boot
void reset_old_values();              // resets the 'old' variables ensuring that all status information are resend
int loop(uint max_time_ms);           // receive / transmit a frame of 20 bytes
void set_power(boolean power);        // power on/off the AC
void set_mode(ACMode mode);           // change AC mode (e.g. heat, dry, cool etc.)
void set_tsetpoint(uint tsetpoint);   // set the target temperature of the AC)
void set_fan(uint fan);               // set the requested fan speed
void set_vanes(uint vanes);           // set the vanes horizontal position (or swing)
void request_ErrOpData();             // request that the AC provides the error data
```
The following sections describe the usage of these functions.

#### void init()
Configures the input /output state of the SPI pins. Resets old values.

### reset_old_values()
This should be called if you want to ensure that the receiver of the status data has the latest data. E.g. in case of a MQTT broker disconnect it should be called.

### int loop(uint max_time_ms)
For receiving / transmitting a frame of 20 bytes.
The input parameter is the maximum time which should be consumed by the loop function. Use a value > T<sub>Frame</sub> + T<sub>FramePause</sub> to ensure there is sufficient time to receive a frame.

This is a blocking function which takes - dependent on the AC model - about 10 ... 50ms. Inside the loop function no delay() or yield() call is used.
The following return values are supported:

return value|meaning
------------|-------------
err_msg_valid_frame     |a valid frame was received in the given time
err_msg_invalid_signature | a frame with invalid signature bytes was received
err_msg_invalid_checksum | a frame with an invalid checksum was received
err_msg_timeout_SCK_low | the specified time max_time_ms has been exceeded because SCK is const low and not toggling
err_msg_timeout_SCK_high | the specified time max_time_ms has been exceeded because SCK is const high and not toggling

note: The input parameters and return values could be changed in future.

### set_*()
Controls the AC.

### request_ErrOpData()
The error operating data will be read upon a request via this function.

### support
Provides the interface between MHI-AC-Ctrl and the user interfaces. 
It contains helper functions for serving WiFi, MQTT, OTA and the external temperature sensor DS18x20. 

### MHI-AC-Ctrl.h and MHI-AC-Ctrl.ino
This is the wrapper for MHI-AC-Ctrl-core and support.
It provides beside the standard setup() and loop() functions the following two functions.

#### void cbiStatusFunction(ACStatus status, int value)
This is a member of the class StatusHandler : public CallbackInterface_Status. It is a callback function called by MHI-AC-Ctrl-core in case of AC status changes. 

#### void MQTT_subscribe_callback(char* topic, byte* payload, unsigned int length)
This function is called for incoming MQTT messages, the message is analyzed and translated to function calls.
From systematic point of view this function should actually be located in [support.h](src/support.h) but in order to keep it simple it resides in [MHI-AC-Ctrl.ino](src/MHI-AC-Ctrl.ino). 

# Integration examples
You find here some examples for integration of MHI-AC-Ctrl
- [Node-Red / Google Assistant](https://github.com/absalom-muc/MHI-AC-Ctrl/issues/60)
- [openHAB](https://community.openhab.org/t/control-mhi-aircon-by-mqtt/104972)
- [IoT MQTT Panel](https://github.com/absalom-muc/MHI-AC-Ctrl/issues/59)
- [Home Assistant](https://github.com/absalom-muc/MHI-AC-Ctrl/issues/58)
- [ESPHome](https://github.com/absalom-muc/MHI-AC-Ctrl/commit/4531327ff2b474eb047317975211df0430bdf199#commitcomment-51302883)
- [Tasmota](https://github.com/absalom-muc/MHI-AC-Ctrl/issues/13#issuecomment-630425714)
- [ioBroker](https://forum.iobroker.net/topic/17041/anfrage-airconwithme-intesishome-klimasteuerung-adapter/14)
- [FHEM](https://forum.fhem.de/index.php/topic,88841.0/all.html)
- [WiFi SSID, hostname and MQTT server dynamic](https://github.com/absalom-muc/MHI-AC-Ctrl/pull/69)
