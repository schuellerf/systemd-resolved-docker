# systemd-resolved-docker

Provides systemd-resolved and docker DNS integration.

A DNS server is configured to listen on each docker interface's IP address. This is used to:
1. allows containers to be referenced by hostname by adding the created DNS servers to the docker interface using
   the systemd-resolved D-Bus API.
1. expose the systemd-resolved DNS service (`127.0.0.53`) to docker containers by proxying DNS requests, which doesn't
   work by default due to the differing network namespaces.

## Features

### Container domain addresses

Based on the container's properties multiple domain names may be generated. For this the `default_domain`
(`DEFAULT_DOMAIN`) and _allowed domains_ (`ALLOWED_DOMAINS`) options are used. The list of _allowed domains_ specifies
which domains may be handled. An entry starting with `.` (example: `.docker`) allows all matching subdomains, otherwise
an exact match is required. If a generated domain address doesn't match the list of _allowed domains_, then the
`default_domain` is appended.

1. `<container_id>.<default_domain>`

   All containers will be reachable by their `container_id`:
   ```sh
   docker run --rm -it alpine                                        #  d6d51528ac46.docker
   docker ps
   CONTAINER ID        IMAGE                     COMMAND                  CREATED             STATUS              PORTS                    NAMES
   d6d51528ac46        alpine                    "/bin/sh"                8 seconds ago       Up 6 seconds                                 relaxed_cartwright
   ```

1. `<container_hostname>.<default_domain>`, `<container_hostname>.<container_domain>.<default_domain>`, `<container_hostname>.<container_domain>`

   If an explicit `--hostname` is provided then that may also be used:
   ```sh
   docker run --rm -it --hostname test      alpine                   # test.docker
   ```
   When the hostname is in the list of _allowed domains_ (`ALLOWED_DOMAINS=.docker,some-host`), then the `default_domain`
   will not be appended:
   ```sh
   docker run --rm -it --hostname some-host alpine                   # some-host
   ```
   If an explicit `--domainname` is provided then that may also be used:
   ```sh
   docker run --rm -it --hostname test --domainname mydomain alpine  # test.mydomain.docker
   ```
   When the domain name is in the list of _allowed domains_ (`ALLOWED_DOMAINS=.docker,.local`), then the `default_domain`
   will not be appended:
   ```sh
   docker run --rm -it --hostname test --domainname local    alpine  # test.local
   ```

1. `<container_name>.<container_network>.<default_domain>`, `<container_name>.<container_network>`

   If a non-default network is used (not `bridge` or `host`) then a name will be generated based on the network's name:
   ```sh
   docker run --rm -it           --network testnet alpine            # zealous_jones.testnet.docker
   docker run --rm -it --name db --network testnet alpine            # db.testnet.docker
   ```
   When the network's name is in the list of _allowed domains_ (`ALLOWED_DOMAINS=.docker,.somenet`), then the
   `default_domain` will not be appended:
   ```sh
   docker run --rm -it           --network somenet alpine            # zealous_jones.somenet
   docker run --rm -it --name db --network somenet alpine            # db.somenet.docker
   ```

If configured correctly then `resolvectl status` should show the configured link-specific DNS server:

    $ resolvectl status
    ...
    Link 7 (docker0)
    Current Scopes: DNS LLMNR/IPv4 LLMNR/IPv6
         Protocols: -DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
       DNS Servers: 172.17.0.1
        DNS Domain: ~docker
    ...

### Docker compose support

For containers started with docker-compose the service tries to generate a short domain name: `<compose-service-name>.<default_domain>`

This would result in a name like `my_app.docker`

Once you would use two compose files with the same service name or start to scale out the service then the short form `<compose-service-name>.<default_domain>` will be removed and only the unique names as described above (e.g. `<container_name>.<container_network>.<default_domain>`) will be available.

### 127.0.0.53 / systemd-resolved within containers

If docker is configured to use the provided DNS server then the container domain names may also be resolved within containers:

```
$ docker run --dns 1.1.1.1 --rm -it alpine
/ # apk add bind
/ # host test.docker
Host test.docker not found: 3(NXDOMAIN)
```

```
$ docker run --dns 172.17.0.1 --rm -it alpine
/ # apk add bind
/ # host test.docker
/ # host test.docker
test.docker has address 172.17.0.3
Host test.docker not found: 3(NXDOMAIN)
Host test.docker not found: 3(NXDOMAIN)
```

If there are link-local, VPN or other DNS servers configured then those will also work within containers.

## Install

### Fedora / COPR

For Fedora and RPM based systems [COPR](https://copr.fedorainfracloud.org/coprs/flaktack/systemd-resolved-docker/) contains pre-built packages.

1. Enabled the COPR repository
   
       dnf copr enable flaktack/systemd-resolved-docker

1.  Install the package
    
        dnf install systemd-resolved-docker
    
1. Start and optionally enable the service
   
       systemctl start  systemd-resolved-docker
       systemctl enable systemd-resolved-docker

1. Docker should be updated to use the DNS server provided by `systemd-docker-resolved`. This may be done
   globally by editing the docker daemon's configuration (`daemon.json`) or per-container using the `--dns`
   flag.

    ```js
    "dns": [
      "172.17.0.1" // docker0 interface's IP address
    ]
    ```

1. NetworkManager may reset the docker interface's configuration for systemd-resolved. If that happens than
   the interface needs to be unmanaged. This may be done by creating a `/etc/NetworkManager/conf.d/99-docker.conf`:

   ```ini
   [main]
   plugins=keyfile

   [keyfile]
   unmanaged-devices=interface-name:docker0
   ```

### Configuration

`systemd-resolved-docker` may be configured using environment variables. When installed using the RPM
`/etc/sysconfig/systemd-resolved-docker` may also be modified to update the environment variables.

| Name             | Description                                                                | Default Value                                          | Example                  |
|------------------|----------------------------------------------------------------------------|--------------------------------------------------------|--------------------------|
| DNS_SERVER       | DNS server to use when resolving queries from docker containers.           | `127.0.0.53` - systemd-resolved DNS server             | `127.0.0.53`             |
| DOCKER_INTERFACE | Docker interface name                                                      | The first docker network's interface                   | `docker0`                |
| LISTEN_ADDRESS   | IPs to listen on for queries from systemd-resolved and docker containers.  | _ip of the default docker bridge_, often `172.17.0.1`  | `172.17.0.1,127.0.0.153` |
| LISTEN_PORT      | Port to listen on for queries from systemd-resolved and docker containers. | `53`                                                   | `1053`                   |
| ALLOWED_DOMAINS  | Domain which will be handled by the DNS server. If a domain starts with `.` then all subdomains will also be allowed.  | `.docker`  | `.docker,.local`         |
| DEFAULT_DOMAIN   | Domain to append to hostnames which are not allowed by `ALLOWED_DOMAINS`.  | `docker`                                               | `docker`                 |

## Build

`setup.py` may be used to create a python package.

`tito` may be used to create RPMs.

## Links

Portions are based on [docker-auto-dnsmasq](https://github.com/metal3d/docker-auto-dnsmasq) and [dnslib](https://github.com/paulc/dnslib).
