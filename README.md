# pgpf

pgpf is a simple failover and proxy middleware that works between PostgreSQL servers and a PostgreSQL database client

[![Go Report Card](https://goreportcard.com/badge/github.com/balugcath/pgpf?style=flat-square)](https://goreportcard.com/report/github.com/balugcath/pgpf)
[![codecov](https://codecov.io/gh/balugcath/pgpf/branch/master/graph/badge.svg)](https://codecov.io/gh/balugcath/pgpf)

## Table of contents

- [How it work](#how-it-work)
- [Install](#install)
- [Start](#start)
- [Configure](#configure)
- [Prometheus metrics](#prometheus_metrics)
- [Benchmark](#benchmark)

## How it work

After start pgpf determines the available master server using 'select pg_is_in_recovery()'. If the master is found, all client connections are redirected through the TCP proxy to it. At the same time pgpf starts monitoring the availability of the master server using 'show server_version'. If the master server is unavailable or an error occurs after the specified time, pgpf determines the available hot-standby server using 'select pg_is_in_recovery()'. If the hot-standby server found, 'select pg_promote()' is used in the procedure for switching the master for postgresql versions above 11, otherwise pgpf executes the specified command for this hot-standby server. All clients are redirected to a new server. Pgpf does **NOT** provide connection pooling and load balancing service, so the usual use of pgpf between PostgreSQL server and PGBouncer.

## Install

Installation from source

```sh
$ git clone https://github.com/balugcath/pgpf $GOPATH/src/github.com/balugcath/pgpf
$ cd $GOPATH/src/github.com/balugcath/pgpf
$ go get -t ./...
$ go build ./cmd/pgpf/...
```

get docker image

```sh
$ docker pull balugcath/pgpf
```

## Start

pgpf start example

```sh
$ cat pgpf.yml
listen: :5432
shard_listen: :5433
failover_timeout: 1
pg_user: postgres
pg_passwd: "123"
prometheus_listen_port: :9092
servers:
  one:
    address: 192.168.1.25:5432
    use: true
  two:
    address: 192.168.1.26:5432
    use: true
$ pgpf -f pgpf.yaml
```

pgpf start example in docker environment

```sh
$ cat pgpf.yml
listen: :5432
shard_listen: :5433
failover_timeout: 1
pg_user: postgres
pg_passwd: "123"
prometheus_listen_port: :9092
servers:
  one:
    address: 192.168.1.25:5432
    use: true
  two:
    address: 192.168.1.26:5432
    use: true
$ ETCDCTL_API=3 etcdctl --endpoints http://192.168.1.16:2379 put pgpf < pgpf.yml
OK
$ docker run -d -p 5432:5432 -p 5433:5433 -p 9092:9092 balugcath/pgpf -e http://192.168.1.16:2379 -k pgpf
$
```

## Configure

The configuration file can be saved locally or in an etcd v3 cluster. The local configuration file must be writable. The runtime configuration is updated by sending a HUP signal to the pgpf process. Below is an example of a complete configuration file

```yaml
listen: :5432
shard_listen: :5433
failover_timeout: 2
pg_user: postgres
pg_passwd: "123"
prometheus_listen_port: :9091
servers:
  one:
    address: 192.168.1.25:5432
    use: true
  two:
    address: 192.168.1.26:5432
    command: ssh -i id_rsa -o StrictHostKeyChecking=no root@192.168.1.26 touch /var/lib/postgresql/12/data/failover_triggerr
    use: true
```

- listen: listen port for client access to the master server, required

- shard_listen: listen port for client access to hot-standby server, optional

- failover_timeout: timeout for switching to hot-standby after detecting access errors to the master server, optional, seconds

- pg_user: username for server access, required

- pg_passwd: password for server access, required

- prometheus_listen_port: listen port for access to prometheus metrics, optional

- servers: array of definitions of available servers, required

  - two: server name, required

  - address: server address, required

  - command: the command to switch hot-standby to master, is required for versions below 12, for versions above it will use 'select pg_promote()'

  - use: server availability flag, required, changes during operation

## Prometheus metrics

pgpf exports the prometheus metrics listed below

- pgpf_client_connections: how many client connected, partition by host
- pgpf_transfer_bytes: how many bytes transferred, partition by host
- pgpf_status_hosts: status hosts, with values:

    0: host is not accessible

    1: host role is hot-standby

    2: host role is master

## Benchmark

without pgpf:

```sh
$ pgbench -c 20 -j 8 -t 100 -h 192.168.1.31 -U postgres test1
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 8
number of transactions per client: 100
number of transactions actually processed: 2000/2000
latency average = 218.650 ms
tps = 91.470342 (including connections establishing)
tps = 91.581986 (excluding connections establishing)
```

with pgpf:

```sh
$ pgbench -c 20 -j 8 -t 100 -h 192.168.1.32 -U postgres test1
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 8
number of transactions per client: 100
number of transactions actually processed: 2000/2000
latency average = 284.515 ms
tps = 70.294974 (including connections establishing)
tps = 70.380981 (excluding connections establishing)
```
