# What's Monitored
- Active Users
- Uptime
- CPU Load total
- Disk Utilization
- Memory Utilization
- CPU Utilization per core (Single Graph)
- Ram Utilization time graph
- Load Average
- Load Average Graph
- CPU and ACPI Temperature Sensors
- Gateway Response time - dpinger
- List of interfaces with IPv4, IPv6, Subnet, MAC, Status and pfSense labels thanks to [/u/trumee](https://www.reddit.com/r/PFSENSE/comments/fsss8r/additional_grafana_dashboard/fmal0t6/)
- WAN Statistics - Traffic & Throughput (Identified by dashboard variable)
- LAN Statistics - Traffic & Throughput (Identified by dashboard variable)
- Firewall Statistics - Blocked Ports, Protocols, Events, Blocked IP Locations, and Top Blocked IP
# Changelog

Updated for later versions of Docker, ElasticSearch, Grafana, Greylog, InfluxDB, OPNsense, and Telegraf.

![Main Dashboard](Grafana-OPNsense.png)

![Suricata Dashboard](Grafana-OPNsense-Suricata.png)

# Running on

    Grafana 12.2.0
    EleasticSearch 7.17.22
    InfluxDB 2.7.12
    Graylog 6.3.1
    Mongo 6.0


# Configuration
Configuration instructions can be found [here](./configure.md).
