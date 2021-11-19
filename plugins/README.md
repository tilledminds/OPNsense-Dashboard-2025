# Installing the Plugins
Place these plugins in "/usr/local/bin". The easiest way of doing this would be to SSH into your router, navigate to "/usr/local/bin" and use curl to download the files, like so:


`curl https://raw.githubusercontent.com/Bsmith101/OPNsense-Dashboard/master/plugins/telegraf_pfifgw.php -o telegraf_pfifgw.php`

`curl https://raw.githubusercontent.com/Bsmith101/OPNsense-Dashboard/master/plugins/telegraf_temperature.sh -o telegraf_temperature.sh`

Make sure to set the permissions to "755"

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
* GW Description
* Delay
* Stddev
* Loss (%)
* Status (Online/Offline/etc.)
