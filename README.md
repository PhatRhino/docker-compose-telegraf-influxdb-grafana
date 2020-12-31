# docker-compose-telegraf-influxdb-grafana

Multi-container Docker app built from the following services:

* [Telegraf](https://github.com/influxdata/telegraf) - agent for collecting, processing, aggregating, and writing metrics.
* [InfluxDB](https://github.com/influxdata/influxdb) - time series database
* [Chronograf](https://github.com/influxdata/chronograf) - admin UI for InfluxDB
* [Grafana](https://github.com/grafana/grafana) - visualization UI for InfluxDB

Useful for quickly setting up an ephemeral Model Driven Telemetry stack.  
This project has been forked from [Jeff Kehres repository](https://github.com/jkehres/docker-compose-influxdb-grafana) to add missing pieces.

## Quick Start

To start the app:

1. Install [docker-compose](https://docs.docker.com/compose/install/) on the docker host.
1. Clone this repo on the docker host.
1. Optionally, change default credentials or Grafana provisioning.
1. Run the following command from the root of the cloned repo:
```
docker-compose up -d
```

To stop the app:

1. Run the following command from the root of the cloned repo:
```
docker-compose down
```

## Ports

The services in the app run on the following ports:

| Host Port | Service |
| - | - |
| 3000 | Grafana |
| 8086 | InfluxDB |
| 57100, 57500 | Telegraf |
| 127.0.0.1:8888 | Chronograf |

Note that Chronograf does not support username/password authentication. Anyone who can connect to the service has full admin access. Consequently, the service is not publically exposed and can only be access via the loopback interface on the same machine that runs docker.

If docker is running on a remote machine that supports SSH, use the following command to setup an SSH tunnel to securely access Chronograf by forwarding port 8888 on the remote machine to port 8888 on the local machine:

```
ssh [options] <user>@<docker-host> -L 8888:localhost:8888 -N
```

## Volumes

No volume is created or used in this ephemeral stack. Data is lost on purpose when the app is stopped. If you want to retain data (metrics, dashboards, etc.)  you can check how to implement volumes on [Jeff Kehres initial project](https://github.com/jkehres/docker-compose-influxdb-grafana).


## Users

The app creates two admin users - one for InfluxDB and one for Grafana. By default, the username and password of both accounts is `admin`. To override the default credentials, set the following environment variables before starting the app:

* `INFLUXDB_USERNAME`
* `INFLUXDB_PASSWORD`
* `GRAFANA_USERNAME`
* `GRAFANA_PASSWORD`

## Database

The app creates a default InfluxDB database called `telemetry`. No retention policy has been configured as metrics disappear as soon as the container is killed.

## Data Sources

The app creates a Grafana data source called `InfluxDB` that's connected to the default IndfluxDB database (e.g. `telemetry`).

To provision additional data sources, see the Grafana [documentation](http://docs.grafana.org/administration/provisioning/#datasources) and add a config file to `./grafana-provisioning/datasources/` before starting the app.

## Sample router configuration

Model Driven Telemetry must be configured on your router to export counters. This is done in 3 steps:

1. Configure a sensor-group to list metrics to collect
1. Configure a destination-group to export metrics (IP of the server hosting the application, protocol is TCP or gRPC)
1. Configure a subscription-group to bind sensors to collectors and specify export frequency.

```
telemetry model-driven
 destination-group COLLECTOR
  address-family ipv4 10.60.11.193 port 57100
   encoding self-describing-gpb
   protocol tcp
   
 sensor-group POWER-COUNTERS
  sensor-path Cisco-IOS-XR-sysadmin-envmon-ui:environment/oper
  sensor-path Cisco-IOS-XR-sysadmin-fretta-envmon-ui:environment/oper/power
  
 subscription SUBSCRIPTION
  sensor-group-id POWER-COUNTERS sample-interval 30000
  destination-id COLLECTOR
```

You can check router exports counters:

```
RP/0/RP0/CPU0:ROUTER#sh telemetry model-driven destination COLLECTOR
Fri Dec 18 16:53:04.763 CET
  Destination Group:  COLLECTOR
  -----------------
    Destination IP:       10.209.198.53
    Destination Port:     57100
    Subscription:         Su8
    State:                Active
    Encoding:             self-describing-gpb
    Transport:            tcp
    No TLS
    Total bytes sent:     31666910
    Total packets sent:   1096
    Last Sent time:       2020-12-18 16:52:47.429043227 +0100

    Collection Groups:
    ------------------
      Id: 31549
      Sample Interval:      30000 ms
    Encoding:             self-describing-gpb
      Num of collection:    157
      Collection time:      Min:     6 ms Max:    14 ms
      Total time:           Min:     6 ms Max:    17 ms Avg:    10 ms
      Total Deferred:       0
      Total Send Errors:    0
      Total Send Drops:     0
      Total Other Errors:   0
    No data Instances:    157
      Last Collection Start:2020-12-18 16:52:45.427333227 +0100
      Last Collection End:  2020-12-18 16:52:45.427343227 +0100
      Sensor Path:          Cisco-IOS-XR-sysadmin-envmon-ui:environment/oper
```

## Dashboards

A sample dashboard is located at `./grafana-provisioning/dashboards/power.json`. This dashboard represents power consumption of several IOS-XR based device using Model Driven Telemetry with TCP and gRPC transports. It's been used to write [this blog article on xrdocs.io](https://xrdocs.io/telemetry/tutorials/ios-xr-telemetry-power-consumption-docker-compose/)

To provision additional dashboards, see the Grafana [documentation](http://docs.grafana.org/administration/provisioning/#dashboards) and add a config file to `./grafana-provisioning/dashboards/` before starting the app.
