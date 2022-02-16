- [Docker](#docker)
- [Configuring InfluxDB](#configuring-influxdb)
- [Configuring Telegraf](#configuring-telegraf)
  * [Telegraf Plugins](#telegraf-plugins)
- [Configuring Graylog](#configuring-graylog)
  * [Add GeoIP to Graylog](#add-geoip-to-graylog)
  * [Configure Additional Settings](#configure-additional-settings)
  * [Add Graylog server as syslog target on OPNsense](#add-graylog-server-as-syslog-target-on-opnsense)
- [Configuring Grafana](#configuring-grafana)
  * [Add InfluxDB and ElasticSearch data sources](#add-influxdb-and-elasticsearch-data-sources)
  * [Import Dashboard](#import-dashboard)
  * [Configure Variables](#configure-variables)
- [Configuration for the Suricata dashboard #Optional](#configuration-for-the-suricata-dashboard-optional)
  * [Update Telegraf.conf](#update-telegrafconf)
  * [Configure logging to UNIX socket](#configure-logging-to-unix-socket)
  * [Import the Suricata Dashboard](#import-the-suricata-dashboard)
- [Troubleshooting](#troubleshooting)
  * [Telegraf Plugins](#telegraf-plugins-1)
  * [Telegraf Troubleshooting](#telegraf-troubleshooting)
  * [Telegraf Config (Paste in to [agent] section)](#telegraf-config-paste-in-to-agent-section)
  * [Restarting Telegraf](#restarting-telegraf)
  * [InfluxDB](#influxdb)
  * [View measurements](#view-measurements)
  * [View field values](#view-field-values)
  * [How to drop an InfluxDB v2 measurement](#how-to-drop-an-influxdb-v2-measurement)
  * [Learn more about Flux queries](#learn-more-about-flux-queries)

## Docker

To simplify everything, we'll use a docker compose file.

You'll need to have Docker and Docker-Compose installed on the server that will host the dashboard.


[Install Docker](https://docs.docker.com/engine/install/)

[Install Docker Compose](https://docs.docker.com/compose/install/)

After you've installed Docker and Docker Compose, copy this code block to a file named docker-compose.yaml.
It's important that you change the TZ environment variable to your timezone for everything to work properly.
I also recommend you change the passwords in this compose file as well.
After you've made the necessary changes, run `docker-compose up -d` in the same directory as your docker-compose.yaml.

```
version: '3'
services:
  mongodb:
    container_name: mongodb
    image: mongo:4.4.10
    volumes:
      - mongodb_data:/data/db
    restart: "unless-stopped"
    environment:
      # Change this to your time zone, valid time zones can be found here: https://www.joda.org/joda-time/timezones.html
      - TZ=CST6CDT
  elasticsearch:
    container_name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch-oss:7.10.2
    volumes:
      - es_data:/usr/share/elasticsearch/data
    environment:
      # Change this to your time zone, valid time zones can be found here: https://www.joda.org/joda-time/timezones.html
      - TZ=CST6CDT
      - http.host=0.0.0.0
      - transport.host=localhost
      - network.host=0.0.0.0
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    mem_limit: 1g
    restart: "unless-stopped"
  graylog:
    container_name: graylog
    image: graylog/graylog:4.2.4
    volumes:
      - graylog_data:/usr/share/graylog/data
    environment:
      # Change this to your time zone, valid time zones can be found here: https://www.joda.org/joda-time/timezones.html
      - TZ=CST6CDT
      # CHANGE ME (must be at least 16 characters)!
      - GRAYLOG_PASSWORD_SECRET=ZDcwMzQ3NTE4ZTIwM
      # Username is "admin"
      # Password is "admin", change this to your own hashed password. 'echo -n "password" | sha256sum'
      - GRAYLOG_ROOT_PASSWORD_SHA2=8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
      - GRAYLOG_HTTP_EXTERNAL_URI=http://127.0.0.1:9000/
    entrypoint: /usr/bin/tini -- wait-for-it elasticsearch:9200 --  /docker-entrypoint.sh
    links:
      - mongodb:mongo
      - elasticsearch
    depends_on:
      - mongodb
      - elasticsearch
    ports:
      # Graylog web interface and REST API
      - 9000:9000
      # Syslog UDP
      - 1514:1514/udp
      # Syslog TCP
      - 1514:1514
    restart: "unless-stopped"
  influxdb:
    container_name: influxdb
    image: influxdb:2.1.1
    ports:
      - '8086:8086'
    volumes:
      - influxdb2_data:/var/lib/influxdb2
    environment:
      # Change this to your time zone, valid time zones can be found here: https://www.joda.org/joda-time/timezones.html
      - TZ=CST6CDT
    restart: "unless-stopped"
  grafana:
    container_name: grafana
    image: grafana/grafana:8.3.3
    ports:
      - '3000:3000'
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - influxdb
    environment:
      # Change this to your time zone, valid time zones can be found here: https://www.joda.org/joda-time/timezones.html
      - TZ=CST6CDT
      # Change these
      - GF_SECURITY_ADMIN_USER=opnsense
      - GF_SECURITY_ADMIN_PASSWORD=opnsense
      - GF_INSTALL_PLUGINS=grafana-worldmap-panel
    restart: "unless-stopped"
volumes:
  grafana_data:
  influxdb2_data:
  graylog_data:
  es_data:
  mongodb_data:
```


Once you have your docker containers running, follow the steps below.

## Configuring InfluxDB
After InfluxDB is started, go to http://(ip of docker server):8086, you will need to setup your username, password, bucket and organization here. Once that is done navigate to the Data tab, click on Telegraf, and create a configuration for a system. Name it, and copy your API token, you will need this for your telegraf configuration. I recommend generating another API token for Grafana. Click on API tokens -> Generate API Token -> Read/Write Access -> Click on your bucket under Read -> and Save. Copy this somewhere as well, you'll need it for Grafana.

## Configuring Telegraf

You must manually install Telegraf on OPNsense, as the OPNsense Telegraf plugin does not currently support custom telegraf configuration.  To do so, SSH into your OPNsense router and use the command

`sudo pkg install telegraf`

After that, use these commands. The first, enables Telegraf to start on boot, and the second, adds Telegraf to sudoers and restricts nopasswd to only what Telegraf needs to run as root.

```
printf 'telegraf_enable="YES"' > /etc/rc.conf.d/telegraf

printf 'telegraf ALL=(root) NOPASSWD: /sbin/pfctl,/usr/local/bin/telegraf_pfifgw.php' >> /usr/local/etc/sudoers
```
You will need this [telegraf config](https://github.com/bsmithio/OPNsense-Dashboard/blob/master/config/telegraf.conf) file. 

You must edit this file and type in your InfluxDB URL, InfluxDB Telegraf API token, organization, and bucket under [[outputs.influxdb_v2]].

You will need to place this config in /usr/local/etc on your OPNsense system.

```
cd /usr/local/etc
curl https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/config/telegraf.conf
# Make the necessary changes with vi, if you wish to edit with nano you'll need to install nano with pkg install nano
vi telegraf.conf
```

### Telegraf Plugins

**Plugins must be copied to your OPNsense system**

Place [telegraf_pfifgw.php](https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/plugins/telegraf_pfifgw.php) and [telegraf_temperature.sh](https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/plugins/telegraf_temperature.sh) in /usr/local/bin and chmod them to 755.

```
cd /usr/local/bin
curl https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/plugins/telegraf_pfifgw.php
curl https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/plugins/telegraf_temperature.sh
chmod 755 telegraf_temperature.sh telegraf_pfifgw.php
```

Test these out before starting the telegraf service by executing them

`./telegraf_pfifgw.php`

`./telegraf_temperature.sh`

The temperature plugin may not work on every system, if you receive `sysctl: unknown oid 'hw.acpi.thermal'` comment out or remove that line from the plugin.


After this is done, use `sudo service telegraf restart` to start telegraf with the new configuration.

## Configuring Graylog

### Add GeoIP to Graylog

To make the map work on Grafana, you must create a MaxMind account here https://www.maxmind.com/en/geolite2/signup. Then generate a license key by going to Account -> Manage License Keys -> Generate New License Key. Copy this key somewhere because you'll need it again soon.

You'll need to download the GeoIP database file to your Graylog container. Access your Graylog container's shell from your Docker host like so

`sudo docker exec -it graylog /bin/bash`

Then download the database file, replace `YOUR_LICENSE_KEY` with the key you generated above.

```
curl "https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-Country&license_key=YOUR_LICENSE_KEY&suffix=tar.gz" -o GeoLite2-Country.tar.gz \
&& tar -xzvf GeoLite2-Country.tar.gz \
&& mv GeoLite2-Country_*/GeoLite2-Country.mmdb /usr/share/graylog/data/data/
```

### Configure Additional Settings

In a browser navigate to http://(ip of docker server):9000 and login.

For Graylog, it's recommended to create an index set. To do so, navigate to System -> Indices. Create an index set with the name "OPNsense / filterlog" and set the index prefix to `opnsense_filterlog`.

![Graylog Index Set](https://www.bsmithio.com/post/opnsense-dashboard/indexset.png)

Once that's done, download the [content pack](https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/config/OPNsense-pack.json) and install it on Graylog by navigating to System -> Content Packs -> Upload, choose the pack, Upload, then Install.

Now, add your index set from earlier to the "OPNsense / filterlog" stream. Navigate to Streams -> More Actions -> Edit Stream -> select your index set and save.

![Graylog Stream Index Set](https://www.bsmithio.com/post/opnsense-dashboard/streamindex.png)

There's one more step we need to do here, navigate to System -> Configurations -> click on Update under Message Processors, and reorder like so:

![Graylog Message Processors](https://www.bsmithio.com/post/opnsense-dashboard/processors.png)

Ensure that all of these are enabled, and click save.

### Add Graylog server as syslog target on OPNsense

Once that is all done, login to your OPNsense router and navigate to System -> Settings -> Logging / targets. Add a new target with the following options: 

![OPNsense Syslog Target](https://www.bsmithio.com/post/opnsense-dashboard/opnsensesyslog.png)

Add a description if you'd like, then click save.

## Configuring Grafana

### Add InfluxDB and ElasticSearch data sources

You will need to add the data sources on Grafana. Navigate to http://(ip of docker server):3000, login and click on the cog wheel and Add a Data Source.

For InfluxDB, make the following configurations

![Grafana InfluxDB Configuration](https://www.bsmithio.com/post/opnsense-dashboard/influxdb.png)


For ElasticSearch, make the following configurations

![Grafana ElasticSearch Configuration](https://www.bsmithio.com/post/opnsense-dashboard/elasticsearch.png)

### Import Dashboard

To import the dashboard, copy the JSON from [OPNsense-Grafana-Dashboard.json](https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/OPNsense-Grafana-Dashboard.json) and navigate to Dashboards -> Browse -> Import and paste under Import via panel json.

### Configure Variables
Dashboard Settings -> Variables

WAN - $WAN is a static variable defined so that a separate dashboard panel can be created for WAN interfaces stats.  Use a comma-separated list for multiple WAN interfaces.

LAN - $LAN uses a regex to remove any interfaces you don't want to be grouped as LAN. The filtering happens in the "Regex" field. I use a negative lookahead regex to match the interfaces I want excluded.  It should be pretty easy to understand what you need to do here. I have excluded igb0 (WAN),igb2,igb3,ovpnc1, and ovpnc1.

Lastly, I don't recommend setting the time range beyond 24 hours, due to how many data points that will return in Grafana.

## Configuration for the Suricata dashboard #Optional
This section assumes you have already configured Suricata.

### Update Telegraf.conf
If you previously used the telegraf.conf in this repo you'll need to modify your telegraf.conf. Otherwise, you can skip this step.

Remove the following section.

```
[[inputs.tail]]
  data_format = "json"
  files = ["/var/log/suricata/eve.json"]
  name_override = "suricata"
  tag_keys = ["event_type","src_ip","src_port","dest_ip","dest_port"]
  json_string_fields = ["*"]
```

Replace with the section below.

```
[[inputs.suricata]]
  ## Data sink for Suricata stats log.
  # This is expected to be a filename of a
  # unix socket to be created for listening.
  source = "/tmp/suricata-stats.sock"

  # Delimiter for flattening field keys, e.g. subitem "alert" of "detect"
  # becomes "detect_alert" when delimiter is "_".
  delimiter = "_"

  # Detect alert logs
  alerts = false
```

Restart telegraf to load changes.

`service telegraf restart`

### Configure logging to UNIX socket

Now add the following section to /usr/local/etc/suricata/custom.yaml.

```
outputs:
  # Extensible Event Format (nicknamed EVE) to UNIX-socket
  - eve-log:
      enabled: yes
      filetype: unix_stream
      filename: /tmp/suricata-stats.sock
      types:
        - stats:
           threads: yes
        - alert:
             # packet: yes              # enable dumping of packet (without stream segments)
             # metadata: no             # enable inclusion of app layer metadata with alert. Default yes
             # http-body: yes           # Requires metadata; enable dumping of http body in Base64
             # http-body-printable: yes # Requires metadata; enable dumping of http body in printable format

             # Enable the logging of tagged packets for rules using the
             # "tag" keyword.
             tagged-packets: yes

             http: yes
             tls: yes
```

After that, navigate to the OPNsense IDS GUI at Services -> Intrusion Detection -> Administration and click Apply. 

### Import the Suricata Dashboard

To import the dashboard, copy the JSON from [OPNsense-Grafana-Dashboard-Suricata.json](https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/OPNsense-Grafana-Dashboard-Suricata.json) and navigate to Dashboards -> Browse -> Import and paste under Import via panel json.

Please let me know if you have any issues with this, as I've only tested this on my own OPNsense router.

## Troubleshooting

### Telegraf Plugins

- You can run most plugins from a shell/ssh session to verify the output. (the environment vars may be different when telegraf is executing the plugin)
- If you're copying from a windows system, make sure the [CRLF is correct](https://www.cyberciti.biz/faq/howto-unix-linux-convert-dos-newlines-cr-lf-unix-text-format/)
- The below command should display unix line endings (\n or LF) as $ and Windows line endings (\r\n or CRLF) as ^M$.

`# cat -e /usr/local/bin/telegraf_pfinterface.php`

### Telegraf Troubleshooting
If you get no good output from running the plugin directly, try the following command before moving to the below step.

    # telegraf --test --config /usr/local/etc/telegraf.conf

To troubleshoot plugins further, add the following lines to the agent block in /usr/local/etc/telegraf.conf and send a HUP to the telegraf pid. You're going to need to do this from a ssh shell. One you update the config you are going to need to tell telegraf to read the new configs. If you restart telegraf from pfSense, this will not work since it will overwrite your changes.

### Telegraf Config (Paste in to [agent] section)
    debug = true
    quiet = false
    logfile = "/var/log/telegraf/telegraf.log"

### Restarting Telegraf
    # ps aux | grep '[t]elegraf.conf'
    # kill -HUP <pid of telegraf proces>

Now go read /var/log/telegraf/telegraf.log
    
### InfluxDB
When in doubt, run a few queries to see if the data you are looking for is being populated.
I recommend doing this in Grafana's Explore tab.

### View measurements
    import "influxdata/influxdb/schema"

    schema.measurements(bucket: "opnsense")
    
### View field values

    from(bucket: "opnsense")
      |> range(start: -24h)
      |> filter(fn: (r) => r["_measurement"] == "system")
      |> limit(n:10)
    
### How to drop an InfluxDB v2 measurement

You must access your influx instance's shell to do this.
To do so run 
`sudo docker exec -it influxdb /bin/bash`
on your docker host.

Then use the following

    bash-4.4# influx delete --bucket "$YourBucket" --predicate '_measurement="$Example"' -o $organization --start "1970-01-01T00:00:00Z" --stop "2050-12-31T23:59:00Z" --token "$YourAPIToken"

### Learn more about Flux queries 

https://docs.influxdata.com/influxdb/cloud/query-data/flux/query-fields/

https://docs.influxdata.com/influxdb/cloud/query-data/flux/explore-schema/
