# Prometheus SQL Exporter [![Go](https://github.com/burningalchemist/sql_exporter/workflows/Go/badge.svg)](https://github.com/burningalchemist/sql_exporter/actions?query=workflow%3AGo) [![Go Report Card](https://goreportcard.com/badge/github.com/burningalchemist/sql_exporter)](https://goreportcard.com/report/github.com/burningalchemist/sql_exporter) [![Docker Pulls](https://img.shields.io/docker/pulls/burningalchemist/sql_exporter)](https://hub.docker.com/r/burningalchemist/sql_exporter) ![Downloads](https://img.shields.io/github/downloads/burningalchemist/sql_exporter/total)

This is a fork of Database agnostic SQL exporter for [Prometheus](https://prometheus.io), created by [@free](https://github.com/free/sql_exporter). The main goal is to bring some maintenance to the project until the original maintainer is back.

## Overview

SQL Exporter is a configuration driven exporter that exposes metrics gathered from DBMSs, for use by the Prometheus
monitoring system. Out of the box, it provides support for MySQL, PostgreSQL, Microsoft SQL Server, Clickhouse,
Snowflake, and Vertica, but any DBMS for which a Go driver is available may be monitored after rebuilding the binary
with the DBMS driver included.

The collected metrics and the queries that produce them are entirely configuration defined. SQL queries are grouped into
collectors -- logical groups of queries, e.g. *query stats* or *I/O stats*, mapped to the metrics they populate.
Collectors may be DBMS-specific (e.g. *MySQL InnoDB stats*) or custom, deployment specific (e.g. *pricing data
freshness*). This means you can quickly and easily set up custom collectors to measure data quality, whatever that might
mean in your specific case.

Per the Prometheus philosophy, scrapes are synchronous (metrics are collected on every `/metrics` poll) but, in order to
keep load at reasonable levels, minimum collection intervals may optionally be set per collector, producing cached
metrics when queried more frequently than the configured interval.

## Usage

Get Prometheus SQL Exporter, either as a [packaged release](https://github.com/burningalchemist/sql_exporter/releases/latest), as a [Docker image](https://hub.docker.com/r/burningalchemist/sql_exporter) or
build it yourself:

```shell
$ go install github.com/burningalchemist/sql_exporter/cmd/sql_exporter
```

then run it from the command line:

```shell
$ sql_exporter
```

Use the `-help` flag to get help information.

```shell
$ ./sql_exporter -help
Usage of ./sql_exporter:
  -config.file string
      SQL Exporter configuration file name. (default "sql_exporter.yml")
  -web.listen-address string
      Address to listen on for web interface and telemetry. (default ":9399")
  -web.metrics-path string
      Path under which to expose metrics. (default "/metrics")
  [...]
```

## Run as a Windows service

If you run SQL Exporter from Windows, it might come in handy to register it as a service to avoid interactive sessions. It is **important** to define `-config.file` parameter to load the configuration file. The other settings can be added as well. The registration itself is performed with Powershell or CMD (make sure you run them as Administrator):

Powershell:

```powershell
New-Service -name "SqlExporterSvc" `
-BinaryPathName "%SQL_EXPORTER_PATH%\sql_exporter.exe -config.file %SQL_EXPORTER_PATH%\sql_exporter.yml" `
-StartupType Automatic `
-DisplayName "Prometheus SQL Exporter"
```

CMD:

```shell
sc.exe create SqlExporterSvc binPath= "%SQL_EXPORTER_PATH%\sql_exporter.exe -config.file %SQL_EXPORTER_PATH%\sql_exporter.yml" start= auto
```

`%SQL_EXPORTER_PATH%` is a path to the SQL Exporter binary executable. This document assumes that configuration files are in the same location.

## Configuration

SQL Exporter is deployed alongside the DB server it collects metrics from. If both the exporter and the DB
server are on the same host, they will share the same failure domain: they will usually be either both up and running
or both down. When the database is unreachable, `/metrics` responds with HTTP code 500 Internal Server Error, causing
Prometheus to record `up=0` for that scrape. Only metrics defined by collectors are exported on the `/metrics` endpoint.
SQL Exporter process metrics are exported at `/sql_exporter_metrics`.

The configuration examples listed here only cover the core elements. For a comprehensive and comprehensively documented
configuration file check out 
[`documentation/sql_exporter.yml`](https://github.com/burningalchemist/sql_exporter/tree/master/documentation/sql_exporter.yml).
You will find ready to use "standard" DBMS-specific collector definitions in the
[`examples`](https://github.com/burningalchemist/sql_exporter/tree/master/examples) directory. You may contribute your own collector
definitions and metric additions if you think they could be more widely useful, even if they are merely different takes
on already covered DBMSs.

**`./sql_exporter.yml`**

```yaml
# Global settings and defaults.
global:
  # Subtracted from Prometheus' scrape_timeout to give us some headroom and prevent Prometheus from
  # timing out first.
  scrape_timeout_offset: 500ms
  # Minimum interval between collector runs: by default (0s) collectors are executed on every scrape.
  min_interval: 0s
  # Maximum number of open connections to any one target. Metric queries will run concurrently on
  # multiple connections.
  max_connections: 3
  # Maximum number of idle connections to any one target.
  max_idle_connections: 3
  # Maximum amount of time a connection may be reused to any one target. Infinite by default.
  max_connection_lifetime: 10m

# The target to monitor and the list of collectors to execute on it.
target:
  # Data source name always has a URI schema that matches the driver name. In some cases (e.g. MySQL)
  # the schema gets dropped or replaced to match the driver expected DSN format.
  data_source_name: 'sqlserver://prom_user:prom_password@dbserver1.example.com:1433'

  # Collectors (referenced by name) to execute on the target.
  collectors: [pricing_data_freshness]

# Collector definition files.
collector_files:
  - "*.collector.yml"
```

### Collectors

Collectors may be defined inline, in the exporter configuration file, under `collectors`, or they may be defined in
separate files and referenced in the exporter configuration by name, making them easy to share and reuse.

The collector definition below generates gauge metrics of the form `pricing_update_time{market="US"}`.

**`./pricing_data_freshness.collector.yml`**

```yaml
# This collector will be referenced in the exporter configuration as `pricing_data_freshness`.
collector_name: pricing_data_freshness

# A Prometheus metric with (optional) additional labels, value and labels populated from one query.
metrics:
  - metric_name: pricing_update_time
    type: gauge
    help: 'Time when prices for a market were last updated.'
    key_labels:
      # Populated from the `market` column of each row.
      - Market
    static_labels:
      # Arbitrary key/value pair
      portfolio: income
    values: [LastUpdateTime]
    query: |
      SELECT Market, max(UpdateTime) AS LastUpdateTime
      FROM MarketPrices
      GROUP BY Market
```

### Data Source Names

To keep things simple and yet allow fully configurable database connections to be set up, SQL Exporter uses DSNs (like
`sqlserver://prom_user:prom_password@dbserver1.example.com:1433`) to refer to database instances. However, because the
Go `sql` library does not allow for automatic driver selection based on the DSN (i.e. an explicit driver name must be
specified) SQL Exporter uses the schema part of the DSN (the part before the `://`) to determine which driver to use.

Unfortunately, while this works out of the box with the [MS SQL Server](https://github.com/denisenkom/go-mssqldb) and
[PostgreSQL](github.com/lib/pq) drivers, the [MySQL driver](github.com/go-sql-driver/mysql) DSNs format does not include
a schema and the [Clickhouse](github.com/kshvakov/clickhouse) one uses `tcp://`. So SQL Exporter does a bit of massaging
of DSNs for the latter two drivers in order for this to work:

DB | SQL Exporter expected DSN | Driver sees
:---|:---|:---
MySQL | `mysql://user:passw@protocol(host:port)/dbname` | `user:passw@protocol(host:port)/dbname`
PostgreSQL (libpq) | `postgres://user:passw@host:port/dbname` | *unchanged*
PostgreSQL (pgx) | `pgx://user:passw@host:port/dbname` | `postgres://user:passw@host:port/dbname`
SQL Server | `sqlserver://user:passw@host:port/instance` | *unchanged*
Clickhouse | `clickhouse://host:port?username=user&password=passw&database=dbname` | `tcp://host:port?username=user&password=passw&database=dbname`
Snowflake  | ` snowflake://user:pass@account/dbname?role=rolename&warehouse=warehousename` | `user:pass@account/dbname?role=rolename&warehouse=warehousename`
Vertica | `vertica://user:passw@host:port/dbname` | *unchanged*


## TLS and basic authentication

SQL Exporter supports TLS and basic authentication. This enables better control of the various HTTP endpoints.

To use TLS and/or basic authentication, you need to pass a configuration file using the `--web.config` parameter. The format of the file is described
[in the exporter-toolkit repository](https://github.com/prometheus/exporter-toolkit/blob/master/docs/web-configuration.md).

## Why It Exists

SQL Exporter started off as an exporter for Microsoft SQL Server, for which no reliable exporters exist. But what is
the point of a configuration driven SQL exporter, if you're going to use it along with 2 more exporters with wholly
different world views and configurations, because you also have MySQL and PostgreSQL instances to monitor?

A couple of alternative database agnostic exporters are available -- https://github.com/justwatchcom/sql_exporter and
https://github.com/chop-dbhi/prometheus-sql -- but they both do the collection at fixed intervals, independent of
Prometheus scrapes. This is partly a philosophical issue, but practical issues are not all that difficult to imagine:
jitter; duplicate data points; or collected but not scraped data points. The control they provide over which labels get
applied is limited, and the base label set spammy. And finally, configurations are not easily reused without
copy-pasting and editing across jobs and instances.
