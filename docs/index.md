---
title: Muppet: Manta loadbalancer
markdown2extras: tables, code-friendly
apisections:
---
<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2014, Joyent, Inc.
-->

# tl;dr

muppet is a custom loadbalancing solution with a haproxy, stud (for SSL
termination) and a daemon that watches group membership changes in ZooKeeper.

# Overview

The loadbalancer can actually be used without the ZooKeeper tie-in, as the ZK
logic is self-contained in a separate process (muppet).  To use the loadbalancer
you'll want stud (available via pkgsrc) installed and configured, and then you
can create a loadbalancer configuration (see below) that fronts your backend
servers.

In manta at least, all configuration of the load balancer is automatically
managed by the muppet service, which watches for registrar changes in
ZooKeeper.  This simply updates the upstream server list of IP addresses
and restarts the loadbalancer service (not gracefully...).

# Configuration

## Stud

This configuration assumes that stud is going to listen on 443, and proxy to
`127.0.0.1:8443`; note that `sendproxy` must be turned on, as the loadbalancer
assumes the remote IP address is sent over by stud.

    #
    # stud(8), The Scalable TLS Unwrapping Daemon's configuration
    #

    # NOTE: all config file parameters can be overriden
    #       from command line!

    # Listening address. REQUIRED.
    #
    # type: string
    # syntax: [HOST]:PORT
    frontend = "[*]:443"

    # Upstream server address. REQUIRED.
    #
    # type: string
    # syntax: [HOST]:PORT.
    backend = "[127.0.0.1]:8443"

    # SSL x509 certificate file. REQUIRED.
    # List multiple certs to use SNI. Certs are used in the order they
    # are listed; the last cert listed will be used if none of the others match
    #
    # type: string
    pem-file = /opt/local/etc/server.pem

    # SSL protocol.
    #
    tls = on
    ssl = on

    # List of allowed SSL ciphers.
    #
    # Run openssl ciphers for list of available ciphers.
    # type: string
    ciphers = ""

    # Enforce server cipher list order
    #
    # type: boolean
    prefer-server-ciphers = on

    # Use specified SSL engine
    #
    # type: string
    ssl-engine = ""

    # Number of worker processes
    #
    # type: integer
    workers = 4

    # Listen backlog size
    #
    # type: integer
    backlog = 100

    # TCP socket keepalive interval in seconds
    #
    # type: integer
    keepalive = 3600

    # Chroot directory
    #
    # type: string
    chroot = ""

    # Set uid after binding a socket
    #
    # type: string
    user = "stud"

    # Set gid after binding a socket
    #
    # type: string
    group = "stud"

    # Quiet execution, report only error messages
    #
    # type: boolean
    quiet = off

    # Use syslog for logging
    #
    # type: boolean
    syslog = off

    # Syslog facility to use
    #
    # type: string
    syslog-facility = "daemon"

    # Run as daemon
    #
    # type: boolean
    daemon = on

    # Report client address by writing IP before sending data
    #
    # NOTE: This option is mutually exclusive with option write-proxy and proxy-proxy.
    #
    # type: boolean
    write-ip = off

    # Report client address using SENDPROXY protocol, see
    # http://haproxy.1wt.eu/download/1.5/doc/proxy-protocol.txt
    # for details.
    #
    # NOTE: This option is mutually exclusive with option write-ip and proxy-proxy.
    #
    # type: boolean
    write-proxy = on

    # Proxy an existing SENDPROXY protocol header through this request.
    #
    # NOTE: This option is mutually exclusive with option write-ip and write-proxy.
    #
    # type: boolean
    proxy-proxy = off


### Generate an OpenSSL Certificate

Run this command:

    $ openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
        -keyout /opt/local/etc/server.pem -out /opt/local/etc/server.pem \
        -subj "/C=US/ST=CA/O=Joyent/OU=manta/CN=localhost"

## Load Balancer

The loadbalancer used is just [HAProxy](http://haproxy.1wt.eu/), but with a
[patch from openwrt](https://dev.openwrt.org/) that lets us get the "real"
client IP address into the `x-forwarded-for` header.  There is an
`haproxy.cfg.in` file that is templated with a sparse number of `%s`; this
file is used to generate a new haproxy.cfg each time the topology of online
loadbalancers changes.

*Important:* Checked into this repo is a "blank" haproxy.cfg - *DO NOT EDIT
THIS FILE!*.  That file is used as a syntactically correct, but empty
haproxy.cfg file that we use to bootstrap haproxy _before_ muppet is running.
Any changes you want to see made to haproxy.cfg must be made in the template
file, as that's what you really care about.

## Muppet

*name* is really the important variable here, as that dicatates the path in
ZooKeeper to watch (note the DNS name is reversed and turned into a `/`
separated path); entries should have been written there by `registrar`.

    {
        "name": "manta.bh1-kvm1.joyent.us",
        "srvce": "_http",
        "proto": "_tcp",
        "port": 80,
        "zookeeper": {
            "servers": [ {
                "host": "10.2.201.66",
                "port": 2181
            } ],
            "timeout": 1000
        }
    }
