# mqtt-logger data archives

Information about the file format and notes about the message types.

Contact us for access to the data itself.
It is available pretty much just for the asking, but it would be great to get some feedback on its usage and interesting results people find.


## Background

If you are teaching or learning about data analytics, you frequently have a need for raw data.
Specifically, you want a set of stuff that is **not** _squeaky clean_ but like the data you would find in the "real world."
Such a <strike>dumpster fire</strike> valuable resouce can be surprisingly hard to come by.

> **You have come to the right place!**


This data begins on roughly 2019-12-01 and, as of 2020-12-20 contains more than 52 million time-stampped messages and is about 1.2 GiB uncompressed.
While this doesn't qualify as "big data," it _is_ large enough to make the cleaning, summarizing, and "feature extraction" work take non-trivial amounts of time on a typical desktop/laptop.
Just enough to feel the effects of your data processing pipeline's efficiency.
For example:

* A simple `zcat *.gz | wc` takes 48 seconds from a warm cache.
* Histogram of the topic strings: `zcat *.gz | cut -d\  -f 2 | sort | uniq -c | tee topics.txt` takes 150 seconds.
* While `zcat *.gz | awk '{a[$2]++} END {for (i in a) printf "%8i %s\n", a[i], i}' | sort -k2 | tee topics-awk.txt` only takes 34 seconds.

Despite the descriptions below, a data analyist will quickly run into many of the problems common to parsing real data.
Simple scripts will _mostly_ work...

We use MQTT as part of our data collection backend services for our sensor nodes and low-rate data collection.

* Telemetry from Raspberry Pi and other single-board computers.
* Bluetooth LE sniffing
* Low-power wireless network experiments, where a gateway connects between the experimental network and the Internet via MQTT.
* Collection of packets from ISM band wireless sensors via [`rtl_433`](https://github.com/merbanan/rtl_433) receivers.   From deployed environmental temperature + humidity sensors, for example.


Examples of what you can do this data set:

* When, exactly, did Facilities Management change the building temperature setpoint after the initial COVID-19 campus shutdown?
* Plot the temperature and humidity of four of the ECE classroom / laboratories.
* Find the likely BLE addresses of devices carried by people.
* When were the batteries changed in the building temperature sensors?
* (bonus!) What were/are the Bluetooth LE MAC addresses of the [TrackR Pixel](https://www.thetrackr.com/) units attached to Prof. White or Prof. Grossman's keychans?




## File format

Each line of the file represents one message from a sensor with 3 fields separated by a _single space_ (`0x20`):

```
datetimestamp1 topic1 message1
datetimestamp2 topic2 message2
datetimestamp3 topic3 message3
```

* `datetimestamp` of the server time in UTC, [RFC3339 format](https://tools.ietf.org/html/rfc3339#section-5.8) with microsecond time resolution \
    (`YYYY-MM-DDTHH:MM:SS.ssssssZ`).
* `topic` string with path/tree separators.  (look up MQTT topics)
* `message` payload data


### `datetimestamp`

Timestamps are created with the highest available precision on the originating device.

**Note:** date-time parsing tools sometimes have problems parsing fractional seconds.
Some tools accept milliseconds only, some microseconds only, some ???.
If you do not need this full precision, it may be convenient to truncate the seconds string to the number of digits of precision that your parsing tool prefers.


### `topic`

Topic strings represent a hierarchical identifier for the source or meaning of the message contents.
The topic levels are separated by forward slashes `/`.

See [MQTT Topics & Best Practices - MQTT Essentials: Part 5](https://www.hivemq.com/blog/mqtt-essentials-part-5-mqtt-topics-best-practices/)
for a good discussion of topics in MQTT.

Topic levels of the form `xxxxxxxxxxxx`, where x is `0-9, a-f, A-F` are generally hex strings that are the MAC addresses of a particular data source.
Specifically, the MAC address of the device transmitting the data via MQTT via the Internet -- some of these devices are _gateways_ and not the original source of the information.


### `message`

The message payload that was received is _everything_ after the space following the topic string to the end of the line.
With the MQTT protocol, this message is allowed to be arbitrary binary data in octets and has no implied meaning or encoding.
This also means that it is _possible_ (but not observed, yet) for the payload to contain a newline character (`\n` or `0x0a`) somewhere inside of the message.
Good luck with that :)

By [!WIRED Lab](https://github.com/wiredlab) convention the message payload is encoded as a [JSON object](json.org).
Note that the data contains messages that **do not** conform to the convention.
Those non-conforming messages are generally not "production data," instead coming from demonstrations and experiments.


## Common topics and data sources

### `esp32-sniffer/mac-address`

Data published by ESP32-based Bluetooth LE sniffers running:
[wiredlab / esp32-ble-scan-mqtt](https://github.com/wiredlab/esp32-ble-scan-mqtt)

The sniffers scan for BLE advertising packets and report the packets' access address (`mac`) and Packet Data Unit (PDU, `payload`).


These topics have three available sub-topic levels:
(`esp32-sniffer/mac-address/ble`)

`/status`
: `{"state":"disconnnected"}`, or
: `{"state":"connected","ssid":"wifi-SSID","ip":"10.1.140.22"}`

`/control`
: `0` or `1` (bare character, no JSON) to control an LED

`/ble`
: `{"time":"...","mac":"...","payload":"...","rssi":-73}`


The `/ble` message JSON has three required fields:

* `time`: Timestamp of the ESP32's clock when the BLE.
* `mac`: Access address of BLE advertising transmitter.
* `payload`: Packet Data Unit (PDU) bytes in ascii-hex.  The PDU can be decoded later, see (https://jimmywongiot.com/2019/08/13/advertising-payload-format-on-ble/).
* `rssi`: (optional) received signal strength.


#### BLE Temperature and Humidity sensors
This topic also collects data from Bluetooth LE Xiaomi LYWSD03MMC temperature and humidity sensors running custom firmware (https://github.com/atc1441/ATC_MiThermometer).
These may be identified by their MAC address prefix of `a4:c1:38:xx:xx:xx` (`"mac":"a4c138______"`).
See the firmware function `set_adv_data()` for information helpful to decode the payload data: https://github.com/atc1441/ATC_MiThermometer/blob/master/ATC_Thermometer/ble.c#L204




### `valpo/rtl433/NAME` previously `rtl433/NAME` 

Collection of packets from ISM band wireless sensors via [`rtl_433`](https://github.com/merbanan/rtl_433) receivers.

`NAME`
: is (currently) the hostname of the machine running the receiver.

The devices publishing to this topic prefix now all use the scripts and systemd service from:

[wiredlab / rtl433-mqtt](https://github.com/wiredlab/rtl433-mqtt)


* Topic `rtl433/gem163` was changed to `valpo/rtl433/wiredlab` and is the same machine and antenna location.

The message is JSON formatted according to the output of `rtl_433 -F json -M time:utc:iso:usec -M protocol -M level`.
Time in UTC in [proper format](https://xkcd.com/1179/) and a received signal/noise power estimate.

`{"time" : "2020-11-21T19:42:52.350410", "protocol" : 20, "model" : "Ambientweather-F007TH", "id" : 247, "channel" : 2, "battery_ok" : 1, "temperature_F" : 70.600, "humidity" : 27, "mic" : "CRC", "mod" : "ASK", "freq" : 433.928, "rssi" : -0.131, "snr" : 14.055, "noise" : -14.185}`





### `valpo/esp8266-rf24-mqtt/mac-address`

(earlier devices used the topic prefix `esp8266-rf24-mqtt`)

Data received by gateway nodes which relay packets from an [RF24Network](https://github.com/nRF24/RF24Network)-based wireless sensor network.
The low-power nodes use `NRF24L01+` 2.4 GHz packet radios typically controlled with Arduino Nano style microcontrollers.
The [Keywish RF-Nano](https://github.com/keywish/keywish-nano-plus/tree/master/RF-Nano) boards are especially convenient.


[wiredlab / rf24-mqtt_master](https://github.com/wiredlab/rf24-mqtt_master)

`/mac-address/` is the WiFi MAC address of the gateway.

`/msg` subtopic example:

`{"channel":90,"time":"2020-04-12T06:25:27Z","from_addr":"2","to_addr":"0","type":0,"payload":"02001f07ae0bdd116cf5b2a1"}`


* `time`: Timestamp the packet was received by the gateway node.
* `channel`: RF24 channel number (frequency).
* `payload`: ascii-hex encoded payload bytes.



### `wiredlab/HOSTNAME/telemetry`

Where `HOSTNAME` is the data source's host name.

Examples:

`wiredlab/satnogs-1/telemetry {"timestamp":"2020-08-22T06:25:41Z", "temp":49.9, "fan":null, "tmpfs_used":0, "tmpfs_percent":0}`

`wiredlab/piaware/telemetry {"timestamp":"2020-12-12T06:19:30Z", "temp":39.3, "fan":1 ,"piaware":{"start":1607753853.5, "modes0":6567, "modes1":469}}`

The `piaware` machine runs [PiAware](https://flightaware.com/adsb/piaware/).
It is a Raspberry Pi 3B+ with the official [Raspberry Pi PoE HAT](https://www.raspberrypi.org/products/poe-hat/) and is mounted on the roof inside a small weather-resistant enclosure.
Data key `modes0` is Mode S packets decoded in the last 1 minute (and only therefore changes each minute) while `modes1` is the Mode S packets which have one error.




<!--
    vim: tw=0
-->

