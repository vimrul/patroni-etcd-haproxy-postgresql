# patroni-etcd-haproxy-postgresql
Comprehensive guide to setting up a high-availability PostgreSQL cluster using Patroni, etcd, and HAProxy.

# Patroni PostgreSQL High-Availability Cluster Setup

This repository contains instructions to set up a high-availability PostgreSQL cluster using Patroni, etcd, and HAProxy across multiple nodes.

## Prerequisites

- Three nodes for PostgreSQL setup (`node1`, `node2`, `node3`).
- One node for etcd setup (`etcdnode`).
- One node for HAProxy setup (`haproxynode`).
- Ubuntu OS installed on all nodes.

---

## Step 1: Setup PostgreSQL Nodes (`node1`, `node2`, `node3`)

Run the following commands on each PostgreSQL node:

```bash
sudo apt update
sudo hostnamectl set-hostname nodeN  # Replace `nodeN` with node1, node2, or node3
sudo apt install -y net-tools gnupg2 wget nano
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg
sudo apt update
sudo apt install -y postgresql-16 postgresql-server-dev-16
sudo systemctl stop postgresql
sudo ln -s /usr/lib/postgresql/12/bin/* /usr/sbin/
sudo apt -y install python python3-pip python3-testresources
sudo pip3 install --upgrade setuptools psycopg2 patroni python-etcd
```

---

## Step 2: Setup etcd Node (`etcdnode`)

Run the following commands on the etcd node:

```bash
sudo apt update
sudo hostnamectl set-hostname etcdnode
sudo apt install -y net-tools etcd
```

---

## Step 3: Setup HAProxy Node (`haproxynode`)

Run the following commands on the HAProxy node:

```bash
sudo apt update
sudo hostnamectl set-hostname haproxynode
sudo apt install -y net-tools haproxy
```

---

## Step 4: Configure etcd on the `etcdnode`

Edit the etcd configuration and set the environment variables:

```bash
ETCD_NAME="etcd-nodeN"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="http://<etcd-node1_IP>:2380"
ETCD_LISTEN_CLIENT_URLS="http://<etcd-node1_IP>:2379,http://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://<etcd-node1_IP>:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://<etcd-node1_IP>:2379"
ETCD_INITIAL_CLUSTER="etcd-node1=http://<etcd-node1_IP>:2380,etcd-node2=http://<etcd-node2_IP>:2380,etcd-node3=http://<etcd-node3_IP>:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
```

---

## Step 5: Configure Patroni on PostgreSQL Nodes

1. Create and edit the Patroni configuration file:

    ```bash
    sudo nano /etc/patroni.yml
    ```

    Example `patroni.yml`:

    ```yaml
    scope: postgres
    namespace: /db/
    name: node1

    restapi:
        listen: <nodeN_ip>:8008
        connect_address: <nodeN_ip>:8008

    etcd:
        hosts: "<etcdnode1_ip>:2379"

    bootstrap:
      dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
          use_pg_rewind: true
          use_slots: true
          parameters: {}

      initdb:
      - encoding: UTF8
      - data-checksums

      pg_hba:
      - host replication replicator 127.0.0.1/32 md5
      - host replication replicator <node1_ip>/0 md5
      - host replication replicator <node2_ip>/0 md5
      - host replication replicator <node3_ip>/0 md5
      - host all all 0.0.0.0/0 md5

      users:
        admin:
          password: admin
          options:
            - createrole
            - createdb

    postgresql:
      listen: <nodeN_ip>:5432
      connect_address: <nodeN_ip>:5432
      data_dir: /data/patroni
      pgpass: /tmp/pgpass
      authentication:
        replication:
          username: replicator
          password: ************
        superuser:
          username: postgres
          password: ************
      parameters:
          unix_socket_directories: '.'

    tags:
        nofailover: false
        noloadbalance: false
        clonefrom: false
        nosync: false
    ```

2. Create the data directory:

    ```bash
    sudo mkdir -p /data/patroni
    sudo chown postgres:postgres /data/patroni
    sudo chmod 700 /data/patroni
    ```

3. Create a systemd service file for Patroni:

    ```bash
    sudo nano /etc/systemd/system/patroni.service
    ```

    Add the following content:

    ```ini
    [Unit]
    Description=High availability PostgreSQL Cluster
    After=syslog.target network.target

    [Service]
    Type=simple
    User=postgres
    Group=postgres
    ExecStart=/usr/local/bin/patroni /etc/patroni.yml
    KillMode=process
    TimeoutSec=30
    Restart=no

    [Install]
    WantedBy=multi-user.target
    ```

4. Start Patroni:

    ```bash
    sudo systemctl start patroni
    sudo systemctl status patroni
    ```

---

## Step 6: Configure HAProxy on the `haproxynode`

1. Edit the HAProxy configuration file:

    ```bash
    sudo nano /etc/haproxy/haproxy.cfg
    ```

    Replace its content with:

    ```
    global
        maxconn 100
        log     127.0.0.1 local2

    defaults
        log global
        mode tcp
        retries 2
        timeout client 30m
        timeout connect 4s
        timeout server 30m
        timeout check 5s

    listen stats
        mode http
        bind *:7000
        stats enable
        stats uri /

    listen postgres
        bind *:5000
        option httpchk
        http-check expect status 200
        default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
        server node1 <node1_ip>:5432 maxconn 100 check port 8008
        server node2 <node2_ip>:5432 maxconn 100 check port 8008
        server node3 <node3_ip>:5432 maxconn 100 check port 8008
    ```

2. Restart HAProxy:

    ```bash
    sudo systemctl restart haproxy
    sudo systemctl status haproxy
    ```

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---

## Contributions

Feel free to open issues or create pull requests to improve this setup.
