---
layout: guide
title: StorageOS Docs - Docker (legacy)
anchor: install
module: install/container
---

# Docker Application Container (Docker Engine 1.10 - 1.12)

Install StorageOS as an application container when Docker managed plugins aren't
available.

## Quick Start

```bash
$ sudo mkdir /var/lib/storageos
$ sudo wget -O /etc/docker/plugins/storageos.json https://docs.storageos.com/assets/storageos.json
$ sudo modprobe nbd nbds_max=1024
$ docker run -d --name storageos \
	-e HOSTNAME \
	--net=host \
	--pid=host \
	--privileged \
	--cap-add SYS_ADMIN \
	--device /dev/fuse \
	-v /var/lib/storageos:/var/lib/storageos:rshared \
	-v /run/docker/plugins:/run/docker/plugins \
	storageos/node server
```

## Overview

### Requirements

The container installation method requires Docker 1.10+.  For Docker 1.13+
most users should use the [managed plugin install](docker.html) method.

### KV Store

StorageOS relies on an external key-value store for configuration data and cluster
management.  See [Consul installation](consul.html) for more details.

### Routable IP Address

StorageOS nodes must be able to contact each other over the network.  By default,
the node's first non-loopback address will be configured as the `ADVERTISE_IP`.
In some cases (such as with Vagrant installations), this will not be appropriate
and it will need to be set manually.

Use `ip a` to list available ip addresses, and then configure StorageOS to use a
specific address by adding `-e ADVERTISE_IP=<ip>` to the StorageOS docker run
command.

TODO - change to container install
```
sudo docker plugin install storageos/plugin ADVERTISE_IP=123.123.123.123
```

## Installation

StorageOS shares volumes via the `/var/lib/storageos` directory.  This must be
present on each node where StorageOS runs.  Prior to installation, create it:

```bash
$ sudo mkdir /var/lib/storageos
```

Docker needs to be configured to use the StorageOS volume plugin.  This is done
by writing a configuration file in `/etc/docker/plugins/storageos.json` with
contents:

```json
{
	"Name": "storageos",
	"Addr": "unix:////run/docker/plugins/storageos/storageos.sock"
}
```

This file instructs Docker to use the volume plugin API listening on the
specified Unix domain socket.  Note that the socket is only accessible by the
root user, and it is only present when the StorageOS client container is running.

NBD (Network Block Device) is a default Linux kernel module that allows block
devices to be run in userspace. It is not a requirement for StorageOS to run,
but improves performance significantly. To enable the module and increase the
number of allowable devices, run:

```bash
$ sudo modprobe nbd nbds_max=1024
```

**To ensure the NBD module is loaded on reboot.**

1. Add the following line to `/etc/modules`
```
nbd
```

2. Add the following module configuration line in `/etc/modprobe.d/nbd.conf`
```
options nbd nbds_max=1024
```

If Consul is running locally, install the plugin using the defaults:

```bash
$ docker run -d --name storageos \
	-e HOSTNAME \
	--net=host \
	--pid=host \
	--privileged \
	--cap-add SYS_ADMIN \
	--device /dev/fuse \
	-v /var/lib/storageos:/var/lib/storageos:rshared \
	-v /run/docker/plugins:/run/docker/plugins \
	storageos/node server
```

If the KV store is not local, supply the IP address of the Consul service using
the `KV_ADDR` environment variable:

```bash
$ docker run -d --name storageos \
	-e HOSTNAME \
  -e KV_ADDR=127.0.0.1:8500 \
	--net=host \
	--pid=host \
	--privileged \
	--cap-add SYS_ADMIN \
	--device /dev/fuse \
	-v /var/lib/storageos:/var/lib/storageos:rshared \
	-v /run/docker/plugins:/run/docker/plugins \
	storageos/node server
$ docker plugin install storageos/plugin KV_ADDR=127.0.0.1:8500
```

Alternatively, to setup a single test StorageOS instance, you can use the
built-in BoltDB by setting `KV_BACKEND=boltdb`.  Note that each StorageOS node
will be isolated, so features such as replication and volume failover will not
be available.

## Environment variables

For most environments, the default settings should work. If Consul is not
running locally on the node, you will need to set `KV_ADDR`.

* `HOSTNAME`: Hostname of the Docker node, only if you wish to override it.
* `ADVERTISE_IP`: IP address of the Docker node, for incoming connections.  Defaults to first non-loopback address.
* `USERNAME`: Username to authenticate to the API with.  Defaults to `storageos`.
* `PASSWORD`: Password to authenticate to the API with.  Defaults to `storageos`.
* `KV_ADDR`: IP address/port of the Key/Vaue store.  Defaults to `127.0.0.1:8500`
* `KV_BACKEND`: Type of KV store to use.  Defaults to `consul`. `boltdb` can be used for single node testing.
* `API_PORT`: Port for the API to listen on.  Defaults to `5705` ([IANA Registered](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=5705)).
* `NATS_PORT`: Port for NATS messaging to listen on.  Defaults to `4222`.
* `NATS_CLUSTER_PORT`: Port for the NATS cluster service to listen on.  Defaults to `8222`.
* `SERF_PORT`: Port for the Serf protocol to listen on.  Defaults to `13700`.
* `DFS_PORT`: Port for DirectFS to listen on.  Defaults to `17100`.
* `LOG_LEVEL`: One of `debug`, `info`, `warning` or `error`.  Defaults to `info`.
* `LOG_FORMAT`: Logging output format, one of `text` or `json`.  Defaults to `json`.
