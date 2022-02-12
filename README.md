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


Converted InfluxQL queries to Flux.

Converted pFSense functions to OPNsense.

Added Firewall panels.

Added subnet info to Interface Summary panels

Added Suricata panels

![Screenshot](Grafana-OPNsense.png)

# Running on

    Grafana 8.2.4
    InfluxDB 2.1.1
    Graylog 4.2


# Configuration
Configuration instructions can be found [here](./configure.md).
