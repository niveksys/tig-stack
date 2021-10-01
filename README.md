# TIG Stack (Telegraf, InfluxDB and Grafana)

## Table of Contents <!-- omit in toc -->
- [1. Configure Synology NAS as remote Docker Host](#1-configure-synology-nas-as-remote-docker-host)
- [2. Run `influxdb` & `grafana` containers on Synology NAS](#2-run-influxdb--grafana-containers-on-synology-nas)
- [3. Configure `influxdb` container](#3-configure-influxdb-container)
- [4. Configure `grafana` container](#4-configure-grafana-container)
- [5. Backup & Restore Data Volumes](#5-backup--restore-data-volumes)
- [6. Import Grafana Dashboard from JSON file](#6-import-grafana-dashboard-from-json-file)
- [7. Query and Delete with InfluxDB CLI](#7-query-and-delete-with-influxdb-cli)
- [8. Configure Mac mini as remote Docker Host](#8-configure-mac-mini-as-remote-docker-host)
- [9. Install & configure `telegraf` on Mac mini](#9-install--configure-telegraf-on-mac-mini)
- [10. Backup & Restore `telegraf` Configurations](#10-backup--restore-telegraf-configurations)
- [11. Creating Docker macvlan network (broken on Mac OSX...)](#11-creating-docker-macvlan-network-broken-on-mac-osx)

## 1. Configure Synology NAS as remote Docker Host

* Synology DSM > Control Panel > Terminal & SNMP > Enable SSH service
* Generate key pairs and copy public key to remote host
    ```shell
    $ cd ~/.ssh
    $ ssh-keygen -t ed25519 -f id_ed25519_nas
    # ssh-keygen -t rsa -b 4096 -f id_rsa_nas
    $ ssh-add -l
    $ ssh-add -D
    $ ssh-add -K ~/.ssh/id_ed25519_nas
    $ ssh-copy-id -i ~/.ssh/id_ed25519_nas user@nas.local
    $ ssh user@nas.local
    ```
* Add the docker group, with the administrator user as a member:
    ```shell
    $ ssh user@nas.local
    $ grep -i docker /etc/group
    $ sudo synogroup --add docker $(whoami)
    ```
* Change the group owner of the Docker socket:
    ```shell
    $ ls -l /var/run/ | grep docker
    $ sudo chown root:docker /var/run/docker.sock
    ```
* Configure SSH environment at remote host
    - Update `/etc/ssh/sshd_config`, change `#PermitUserEnvironment no` to `PermitUserEnvironment yes`
    - Restart SSH by restarting the NAS
    - create a new file `~/.ssh/environment`
        ```shell
        $ env | grep PATH | tee .ssh/environment
        ```
* Connect docker remote host with SSH connection
    ```shell
    $ docker -H ssh://user@nas.local ps
    ```
* Connect docker remote host with Context
    ```shell
    $ docker context create nas --docker "host=ssh://user@nas.local"
    $ docker context ls
    $ docker context use nas
    $ docker context ls
    $ docker ps
    ```

## 2. Run `influxdb` & `grafana` containers on Synology NAS

* Start `influxdb` & `grafana` with docker compose on Synology NAS
    ```shell
    $ cd ~/Workspaces/tig-stack
    $ docker context use nas
    $ docker compose config
    $ docker compose up -d
    $ docker compose logs -f
    ```

## 3. Configure `influxdb` container

* Setup at: http://nas.local:8086
    - Username: admin
    - Password
    - Inital Organization Name: Homelab
    - Initial Bucket Name: telegraf
* Date > Buckets > Create Bucket
    - Name: sensors
* Date > Tokens > Generate Token > Read / Write Token
    - Description: telegraf's Token
    - Read > Scoped > Buckets: sensors, telegraf
    - Write > Scoped > Buckets: sensors, telegraf
* Date > Tokens > Generate Token > Read / Write Token
    - Description: grafana's Token
    - Read > Scoped > Buckets: sensors, grafana
    - Write > Scoped > Buckets: sensors, grafana 

## 4. Configure `grafana` container

* Setup at: http://nas.local:3000
    - Default username: admin
    - Default password: admin
* Add a Data Source: Configuration > Data sources > Add data source > InfluxDB
* Configure InfluxDB Data Source:
    - Name: InfluxDB
    - Query Language: Flux
    - URL: http://influxdb:8086
    - Basic auth: disable
    - Organization: Homelab
    - Token
    - Default Bucket: sensors

## 5. Backup & Restore Data Volumes

* Backup data volumes:
    ```shell
    $ cd ~/Workspaces/tig-stack/backup
    $ docker ps
    $ docker compose stop
    $ docker run --rm -it --volumes-from influxdb -v $(pwd):/backup busybox sh -c "tar cvf /backup/influxdb-data-vol.tar -C / var/lib/influxdb2 && tar cvf /backup/influxdb-config-vol.tar -C / etc/influxdb2"
    $ docker run --rm -it --volumes-from grafana -v $(pwd):/backup busybox tar cvf /backup/grafana-vol.tar -C / var/lib/grafana
    ```

* Restore data volumes:
    ```shell
    $ docker volume create influxdb-data-vol
    $ docker volume create influxdb-config-vol
    $ docker volume create grafana-vol
    $ docker volume ls
    $ docker run --rm -it -v influxdb-data-vol:/var/lib/influxdb2 -v influxdb-config-vol:/etc/influxdb2 -v /volume2/docker/backup:/backup busybox sh -c "tar xvf /backup/influxdb-data-vol.tar -C /var/lib/influxdb2 --strip 3 && tar xvf /backup/influxdb-config-vol.tar -C /etc/influxdb2 --strip 2"
    $ docker run --rm -it -v grafana-vol:/var/lib/grafana -v /volume2/docker/backup:/backup busybox tar xvf /backup/grafana-vol.tar -C /var/lib/grafana --strip 3
    $ docker run --rm -it -v influxdb-data-vol:/var/lib/influxdb2 -v influxdb-config-vol:/etc/influxdb2 busybox sh
    $ docker run --rm -it -v grafana-vol:/var/lib/grafana busybox sh

    $ docker run --rm -it -v grafana-vol:/var/lib/grafana busybox sh
    $ docker exec -it grafana sh
    ```

## 6. Import Grafana Dashboard from JSON file

* Dashboard > Manage > Import > Upload JSON file

* [Dashboard not found error on import](https://github.com/hagen1778/grafana-import-export/issues/10)
    ```shell
    $ cd ~/Workspaces/tig-stack/backup
    $ brew install gnu-sed
    $ gsed -i '0,/"id": .*/{s/"id": .*/"id": null,/}' dashboard.json
    ```

## 7. Query and Delete with InfluxDB CLI

* Run an interactive `bash` shell in `influxdb` container:
    ```shell
    $ docker exec -it influxdb /bin/bash
    ```
* Create an influx CLI config and set it as active:
    ```shell
    $ influx config create --config-name local-config \
        --host-url http://localhost:8086 \
        --org Homelab \
        --token <your-auth-token> \
        --active
    ```
* Query with Flux:
    ```shell
    $ influx query 'from(bucket:"sensors") |> range(start: -1mo) |> filter(fn: (r) => r["topic"] =~ /sensors\/openaq/)'
    ```
* Delete measurments on specific topic with InfluxDB CLI:
    ```shell
    $ influx delete --org Homelab --bucket sensors \
        --start '1970-01-01T00:00:00Z' \
        --stop $(date +"%Y-%m-%dT%H:%M:%SZ") \
        --predicate 'topic="sensors/openaq/aqi"'
    ```

## 8. Configure Mac mini as remote Docker Host

* Generate key pairs and copy public key to remote host
    ```shell
    $ cd ~/.ssh
    $ ssh-keygen -t ed25519 -f id_ed25519_mac-mini
    # ssh-keygen -t rsa -b 4096 -f id_rsa_mac-mini
    $ ssh-add -l
    $ ssh-add -D
    $ ssh-add -K ~/.ssh/id_ed25519_mac-mini
    $ ssh-copy-id -i ~/.ssh/id_ed25519_mac-mini user@mac-mini.local
    $ ssh user@mac-mini.local
    ```
* For without `ssh-copy-id` command, setup authorized_keys at remote host
    ```shell
    @mac-mini
    $ ssh user@mac-mini.local
    $ cd ~
    $ mkdir .ssh
    $ chmod 700 .ssh
    $ touch .ssh/authorized_keys
    $ chmod 640 .ssh/authorized_keys

    @localhost
    $ cat ~/.ssh/id_rsa.pub | ssh user@mac-mini 'cat >> .ssh/authorized_keys'
    ```
* Configure SSH environment at remote host
    - Update `/etc/ssh/sshd_config`, change `#PermitUserEnvironment no` to `PermitUserEnvironment yes`
    - System Preferences > Sharing > Remote Login: restart SSH by unchecking and checking the checkbox
    - create a new file `~/.ssh/environment`
        ```shell
        $ echo "PATH=$PATH:/usr/local/bin" >> ~/.ssh/environment
        ```
* Connect docker remote host with SSH connection
    ```shell
    $ docker -H ssh://user@mac-mini.local ps
    ```
* Connect docker remote host with Context
    ```shell
    $ docker context create mac-mini --docker "host=ssh://user@mac-mini.local"
    $ docker context ls
    $ docker context use mac-mini
    $ docker context ls
    $ docker ps
    ```

## 9. Install & configure `telegraf` on Mac mini

* Install via Homebrew:
    ```shell
    $ ssh user@mac-mini.local
    $ brew install telegraf
    ```
* Edit `telegraf.conf`:
    ```shell
    $ vi /usr/local/etc/telegraf.conf
    ```
* Update as follow:
    ```conf
    ...
    # Configuration for sending metrics to InfluxDB
    # [[outputs.influxdb]]
    ...
    # Configuration for sending metrics to InfluxDB
    [[outputs.influxdb_v2]]
      ## The URLs of the InfluxDB cluster nodes.
      urls = ["http://127.0.0.1:8086"]

      ## Token for authentication.
      token = "$TOKEN"

      ## Organization is the name of the organization you wish to write to; must exist.
      organization = "Homelab"

      ## Destination bucket to write into.
      bucket = "telegraf"
    ...
    ```
* Run `telegraf` as service:
    ```shell
    $ brew services restart telegraf
    ```

## 10. Backup & Restore `telegraf` Configurations

* Backup configuration files:
    ```shell
    $ cd ~/Workspaces/tig-stack/backup
    $ scp mac-mini.local:/usr/local/etc/telegraf.conf .
    ```
* Restore configuration files:
    ```shell
    $ cd ~/Workspaces/tig-stack/backup
    $ scp telegraf.conf mac-mini.local:/usr/local/etc
    ```

## 11. Creating Docker macvlan network (broken on Mac OSX...)

* [Set up a PiHole using Docker MacVlan Networks](https://blog.ivansmirnov.name/set-up-pihole-using-docker-macvlan-network/)
* [Using Docker macvlan Networks](https://blog.oddbit.com/post/2018-03-12-using-docker-macvlan-networks/)
* [macvlan driver doesn't work in MacOS](https://github.com/docker/for-mac/issues/3926)
* [macvlan / ipvlan parent interface (e.g. en0 instead of eth0) broken on Mac OSX](https://github.com/moby/libnetwork/issues/2614)
```shell
$ docker network create -d macvlan -o parent=en0 \
    --subnet 192.168.1.0/24 \
    --gateway 192.168.1.1 \
    --ip-range 192.168.1.10/28 \
    --aux-address 'host=192.168.1.11' \
    macvlan0
```
