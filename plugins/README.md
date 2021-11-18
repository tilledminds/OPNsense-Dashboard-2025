# Installing the Plugins
Perhaps the easiest method of installing these plugins is by utlizing the "Filer" plugin in pfSense. Simply type the entire file name (i.e. "/usr/local/bin/plugin_name") and paste the code into the window. Make sure to set the permissions to be "0755", as they default to "0644" which is NOT executable!

# telegraf_pfifgw.php

This single script collects information for Interfaces and gateways.

**Interfaces:**
* Interface name
* IP4 address
* IP4 subnet
* IP6 address
* IP6 subnet
* MAC address
* Friendly name
* Status (Online/Offline/Etc.)

**Gateways:**
* Interface name
* Monitor IP
* Source IP
* Default gw (True/False)
* GW Description
* Delay
* Stddev
* Loss (%)
* Status (Online/Offline/etc.)
* Substatus (None/Packetloss/Latency/Etc.)
