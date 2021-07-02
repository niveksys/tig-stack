# TIG Stack (Telegraf, InfluxDB and Grafana)

## Table of Contents <!-- omit in toc -->
- [1. Configure Mac mini as remote Docker Host](#1-configure-mac-mini-as-remote-docker-host)
- [2. Run `influxdb` & `grafana` containers on Mac mini](#2-run-influxdb--grafana-containers-on-mac-mini)
- [3. Configure `influxdb` container](#3-configure-influxdb-container)
- [4. Install & configure `telegraf` on Mac mini](#4-install--configure-telegraf-on-mac-mini)
- [5. Configure `grafana` container](#5-configure-grafana-container)
- [6. Backup & Restore Configurations](#6-backup--restore-configurations)
- [7. Creating Docker macvlan network (broken on Mac OSX...)](#7-creating-docker-macvlan-network-broken-on-mac-osx)

## 1. Configure Mac mini as remote Docker Host

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

## 2. Run `influxdb` & `grafana` containers on Mac mini

* Connect docker remote host with Context
    ```shell
    $ docker context create mac-mini --docker "host=ssh://user@mac-mini.local"
    $ docker context ls
    $ docker context use mac-mini
    $ docker context ls
    $ docker ps
    ```
* Start `influxdb` & `grafana` with docker compose on Mac mini
    ```shell
    $ cd ~/Workspaces/tig-stack
    $ docker context use mac-mini
    $ docker compose config
    $ docker compose up -d
    $ docker compose logs -f
    ```

## 3. Configure `influxdb` container

* Setup at: http://mac-mini.local:8086
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

## 4. Install & configure `telegraf` on Mac mini

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

## 5. Configure `grafana` container

* Setup at: http://mac-mini.local:3000
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
    - Default Bucket: telegraf
* Import Dashboard: Create > Import
    - Import via grafana.com: 14126
    - InfluxDB2.0: InfluxDB

## 6. Backup & Restore Configurations

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

## 7. Creating Docker macvlan network (broken on Mac OSX...)
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
