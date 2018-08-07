# Dashboards for use with Grafana, InfluxDB and telegraf

Fork from https://grafana.com/dashboards/928

Instead of 1 dashboard with datasource variable (which not work with downsampled data https://github.com/influxdata/influxdb/issues/7332), split the dashboard with fix datasource names


![](/images/dashboard.png)

Tested with telegraf 1.6.3 / Influxdb 1.5.2 / Grafana 5.1.3 on CentOS 7

## requirements

### telegraf

custom telegraf config file with the following content:

```
[[inputs.disk]]
  ignore_fs = ["tmpfs", "devtmpfs", "devfs"]

[[inputs.net]]
  fieldpass = ["bytes_recv", "bytes_sent", "tcp_estabresets", "tcp_outrsts", "tcp_activeopens", "tcp_passiveopens", "drop_in", "drop_out", "err_in", "err_out", "host"]

[[inputs.linux_sysctl_fs]]
  fieldpass = ["file-max", "file-nr", "host"]

[[inputs.netstat]]
  fieldpass = ["tcp_close", "tcp_close_wait", "tcp_closing", "tcp_established", "tcp_fin_wait1", "tcp_fin_wait2", "tcp_last_ack", "tcp_syn_recv", "tcp_syn_sent", "tcp_time_wait", "host"]
```

### influxdb

* DB: telegraf 
  * 30 days data retention policy
  * 10 sec data interval
  
* DB: telegraf_downsampled 
  * downsampled data (infinity)
  * 15 min data interval

```
CREATE DATABASE telegraf
CREATE DATABASE telegraf_downsampled
CREATE RETENTION POLICY "rp_short" ON "telegraf" DURATION 30d REPLICATION 1 DEFAULT
CREATE CONTINUOUS QUERY cq_all_measurement ON telegraf BEGIN SELECT mean(*) INTO telegraf_downsampled.autogen.:MEASUREMENT FROM telegraf.rp_short./.*/ GROUP BY time(15m), * END
```

### grafana

The following datasources are required:

* datasource 1:
  * Name: influxDB-telegraf
  * Database: telegraf

* datasource 2:
  * Name: influxDB-telegraf-downsampled
  * Database: telegraf_downsampled


## Dashboard for availability checks

### influxdb

* DB: telegraf_availability 
  * default

```
CREATE DATABASE telegraf_availability
```

### telegraf

Routing the http_response input to db telegraf_availability:

```
[[outputs.influxdb]]
urls = ["https://<fqdn>:8086"]
database = "telegraf"
username = "telegraf"
password = "<pw>"
[outputs.influxdb.tagdrop]
influxdb_database = "*"

[[outputs.influxdb]]
urls = ["https://<fqdn>:8086"]
database = "telegraf_availability"
username = "telegraf"
password = "<pw>"
tagexclude = ["influxdb_database"]
[outputs.influxdb.tagpass]
influxdb_database = ["telegraf_availability"]

[[inputs.http_response]]
address = "https://<url>/@@health-check"
response_string_match = '"status": "OK"'
interval = "30s"
response_timeout = "10s"
[inputs.http_response.tags]
influxdb_database = "telegraf_availability"
```

### grafana

The following datasources are required:

* datasource 3:
  * Name: influxDB-telegraf-availability
  * Database: telegraf_availability


The plugin piechart-panel is needed, install it on the server:
```
grafana-cli plugins install grafana-piechart-panel
```
