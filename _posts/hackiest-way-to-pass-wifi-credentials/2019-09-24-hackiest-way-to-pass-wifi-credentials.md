---
layout: post
title: "The hackiest way I've seen to pass WiFi credentials to an IoT device"
description: "Picture this: you just bought your brand new IoT device. You take it out from the box and power it on. But... how do you pass your WiFi credentials to make it connect to your network? This solution literally blew my mind!"
date: 2019-09-24
keywords:
    - iot
    - smart config
    - ap mode
    - esp8266
    - tuya
    - internet of things
---

During the first startup of a new IoT device, users must be able to make it connect to them WiFi network. But, how it is possible to supply WiFi credentials to the device, if the device itself isn't attached to anything? 

Of course, the device can be shipped with some additional communication module (like Bluetooth or an ultrasound microphone) that will be used specifically to receive the network credentials. However, manufacturers may be interested in keeping production costs very low, so there's interest in doing everything with just the WiFi module.



To address this problem, the most commonly adopted solution is **AP mode**. 

Upon the startup, the device activates a software access point, along with a webserver and a DNS server. The user must disconnect from its home network and connect to the device AP. At this point, a setup webpage automatically appears, allowing the user to supply all the needed credentials. Once the credentials are sent, the device powers off its access point and connects itself to the home network. On the ESP8266, this can be implemented using the [WiFiManager](https://github.com/tzapu/WiFiManager) library.

Even if relatively simple, the user is forced to change its WiFi network back and forth, which may be annoying. For this reason, a second, **hacky and more interesting solution** has been designed.

**Smart config** does not even require the user to switch between the device AP and his home network, providing a seamless way to send WiFi credentials to IoT devices. 
This time, the device puts its network interface to <u>promiscous mode</u>, with the purpose of sniffing every packet transmitted nearby. On the other side, the user configuration app sends tons of UDP packets via broadcast in the home network, <u>encoding SSID and PSK</u> of the WiFi network <u>in the length field on each packet</u>. Indeed, while it is true that the content of those packets cannot be read from hosts that do not know which is the network passphrase (as our IoT device), the length field can be read with no hassle.

Tuya IoT devices implement this solution: we can take a look on the relative [API](https://github.com/TuyaAPI/link/blob/master/lib/link.js) to understand how does it work. 
The protocol starts with the transmission of a preamble sequence, i.e. a series of UDP packets that carry, in their length field, a recognisable pattern (1,3,6,10). This helps the IoT device to filter out which device is supposed to send useful data, by looking at the source MAC address. Then, the actual data is sent over multiple UDP packets, as the length field can hold up to 2 bytes of data. 

Here you can find some interesting material:

- [Documentation for the ESP8266](https://www.espressif.com/sites/default/files/30b-esp-touch_user_guide_en_v1.1_20160412_0.pdf)
- [Python script ](https://github.com/ct-Open-Source/tuya-convert/tree/master/scripts/smartconfig)

