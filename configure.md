- [Docker](#docker)
- [Configuring InfluxDB](#configuring-influxdb)
- [Configuring Telegraf](#configuring-telegraf)
- * [For previous users](#for-previous-users)
  * [Install the plugin and configure options](#install-the-plugin-and-configure-options)
  * [Add telegraf to sudoers](#add-telegraf-to-sudoers)
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
  * [Add the necessary files](#add-the-necessary-files)
  * [Restart Suricata and Telegraf](#restart-suricata-and-telegraf)
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

### For previous users

If you previously used the pkg install version of telegraf, follow these instructions.

Run `sudo pkg remove telegraf` to remove telegraf.

Delete the line that starts with telegraf in /usr/local/etc/sudoers.

Once those are done you can continue with the new configuration.

### Install the plugin and configure options
Install the Telegraf plugin on OPNsense, to do so, navigate to System -> Firmware -> Plugins -> Search for telegraf, and click the plus icon to install.

![](https://i.imgur.com/vowGSSx.png)

Navigate to Services -> Telegraf -> Input

Enable Network and PF Inputs.

![](https://i.imgur.com/WskfVlS.png)

Then click Save.

Now navigate to Services -> Telegraf -> Output

Enable Influx v2 Output and fill in the following:

Influx v2 Token: Your InfluxDB Token
Influx v2 URL: Your InfluxDB URL, this will be the IP address or hostname of your system that is running InfluxDB. E.g http://192.168.1.10:8086
Influx v2 Organization: Your InfluxDB Organization
Influx v2 Bucket: Your InfluxDB Bucket

![](https://i.imgur.com/VS4FKU7.png)

Then click Save.

### Add telegraf to sudoers

After that, we need to add telegraf to sudoers and use nopasswd to restrict telegraf to only what it needs to run as root.

```
printf 'telegraf ALL=(root) NOPASSWD: /usr/local/bin/telegraf_pfifgw.php' >> /usr/local/etc/sudoers
```

Add the  [custom.conf](./config/custom.conf) telegraf config to /usr/local/etc/telegraf.d

```
curl https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/config/custom.conf -o /usr/local/etc/telegraf.d/custom.conf
```

### Telegraf Plugins

**Plugins must be copied to your OPNsense system**

Place [telegraf_pfifgw.php](https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/plugins/telegraf_pfifgw.php) and [telegraf_temperature.sh](https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/plugins/telegraf_temperature.sh) in /usr/local/bin and chmod them to 755.

```
cd /usr/local/bin
curl "https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/plugins/telegraf_pfifgw.php" -o telegraf_pfifgw.php
curl "https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/plugins/telegraf_temperature.sh" -o telegraf_temperature.sh
chmod 755 telegraf_temperature.sh telegraf_pfifgw.php
```

Test these out before starting the telegraf service by executing them

`sudo ./telegraf_pfifgw.php`

`./telegraf_temperature.sh`

The temperature plugin may not work on every system, if you receive `sysctl: unknown oid 'hw.acpi.thermal'` comment out or remove that line from the plugin.


After this is done, navigate to Services -> Telegraf -> General -> Enable Telegraf Agent.

Lastly, check if Telegraf is running

`sudo service telegraf status`

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

Query Language: Flux
URL: http://influxdb:8086
Organization: Your InfluxDB Organization
Token: Your Grafana InfluxDB API Token
Default Bucket: Your opnsense bucket. This will be the bucket that the panel queries will use.
![Grafana InfluxDB Configuration](https://www.bsmithio.com/post/opnsense-dashboard/influxdb.png)

For ElasticSearch, make the following configurations

URL: http://elasticsearch:9200
Time field name: timestamp
Version: 7.10+
![Grafana ElasticSearch Configuration](https://www.bsmithio.com/post/opnsense-dashboard/elasticsearch.png)

### Import Dashboard

To import the dashboard, copy the JSON from [OPNsense-Grafana-Dashboard.json](https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/OPNsense-Grafana-Dashboard.json) and navigate to Dashboards -> Browse -> Import and paste under Import via panel json.

### Configure Variables

Dashboard Settings -> Variables -> Click on a variable to edit

WAN - $WAN is the variable for our WAN interface panels. It is currently set to igb0 under Custom options. Use a comma-separated list for multiple WAN interfaces.

LAN - $LAN uses a regex to remove any interfaces you don't want to be grouped as LAN. The filtering happens in the "Regex" field. I use a negative lookahead regex to match the interfaces I want excluded.  It should be pretty easy to understand what you need to do here. I have excluded igb0 (WAN),igb2,igb3,ovpnc1, and wg0.

iface - $iface is the interface variable for the Firewall panels, I have it set to igb0 by default. You can either remove igb0 so you can select all interfaces, or use a comma separated list of interfaces.

dataSource - dataSource is the variable for our InfluxDB data source. It is the data source that will be used for all panels with InfluxDB queries. You should see a preview of values you can pick from. Under Instance name filter, use the name of the InfluxDB data source you created for OPNsense if you have multiple InfluxDB data sources.

ESdataSource - ESdataSource is the variable four our Elasticsearch data source. It is the data source that will be used for all panels with Elasticsearch queries. You should see a preview of values you can pick from. Under Instance name filter, use the name of the Elastisearch data source you created for OPNsense if you have multiple Elasticsearch data sources.

Lastly, I don't recommend setting the time range beyond 24 hours, due to how many data points that will return in Grafana.

## Configuration for the Suricata dashboard #Optional
This section assumes you have already configured Suricata.

### Add the necessary files

Add [suricata.conf](./config/suricata/suricata.conf) to /usr/local/etc/telegraf.d

```
curl 'https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/config/suricata/suricata.conf' -o /usr/local/etc/telegraf.d/suricata.conf
```

Add [custom.yaml](./config/suricata/custom.yaml) to /usr/local/opnsense/service/templates/OPNsense/IDS

```
curl 'https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/config/suricata/custom.yaml' -o /usr/local/opnsense/service/templates/OPNsense/IDS/custom.yaml
```

Create the log file and give telegraf permissions to read it

```
touch /tmp/eve.json
chown :telegraf /tmp/eve.json
chmod 640 /tmp/eve.json
```

### Restart Suricata and Telegraf

Restart Suricata from Services -> Intrusion Detection -> Administration

Uncheck Enabled and apply.
Check Enabled and apply.

`sudo service telegraf restart`


### Import the Suricata Dashboard

To import the dashboard, copy the JSON from [OPNsense-Grafana-Dashboard-Suricata.json](https://raw.githubusercontent.com/bsmithio/OPNsense-Dashboard/master/OPNsense-Grafana-Dashboard-Suricata.json) and navigate to Dashboards -> Browse -> Import and paste under Import via panel json.

## Troubleshooting

### Telegraf Plugins

- You can run most plugins from a shell/ssh session to verify the output. (the environment vars may be different when telegraf is executing the plugin)
- If you're copying from a windows system, make sure the [CRLF is correct](https://www.cyberciti.biz/faq/howto-unix-linux-convert-dos-newlines-cr-lf-unix-text-format/)
- The below command should display unix line endings (\n or LF) as $ and Windows line endings (\r\n or CRLF) as ^M$.

`# cat -e /usr/local/bin/telegraf_pfinterface.php`

### Telegraf Troubleshooting
If you get no good output from running the plugin directly, try the following command before moving to the below step.

    # telegraf --test --config /usr/local/etc/telegraf.conf

To troubleshoot plugins further, add the following lines to the agent block in /usr/local/etc/telegraf.conf and send a HUP to the telegraf pid. You're going to need to do this from a ssh shell. Once you update the config you are going to need to tell telegraf to read the new configs. If you restart telegraf from OPNsense, this will not work since it will overwrite your changes.

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
