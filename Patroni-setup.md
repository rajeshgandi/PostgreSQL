## Patroni-Setup

```
192.168.110.189	p1
192.168.110.190	p2
192.168.110.191	p3
192.168.110.192	ha-vip
```
```
sudo hostnamectl set-hostname p1
reboot
root@p1 ~]#
```
```
useradd postgres
passwd postgres
```
```
vi /etc/hosts
192.168.110.189	p1
192.168.110.190	p2
192.168.110.191	p3
192.168.110.192	ha-vip
```

```
vi /etc/selinux/config
SELINUX=disabled
reboot
```

```
firewall-cmd --zone=public --add-port=5432/tcp --permanent
firewall-cmd --zone=public --add-port=6432/tcp --permanent
firewall-cmd --zone=public --add-port=8008/tcp --permanent
firewall-cmd --zone=public --add-port=2379/tcp --permanent
firewall-cmd --zone=public --add-port=2380/tcp --permanent
firewall-cmd --permanent --zone=public --add-service=http
firewall-cmd --zone=public --add-port=5000/tcp --permanent
firewall-cmd --zone=public --add-port=5001/tcp --permanent
firewall-cmd --zone=public --add-port=7000/tcp --permanent
firewall-cmd --zone=public --add-port=112/tcp --permanent
firewall-cmd --zone=public --add-port=5405/tcp --permanent
firewall-cmd --add-rich-rule='rule protocol value="vrrp" accept' --permanent
firewall-cmd --reload

```

```
cd /etc/pki/rpm-gpg
[root@p1 rpm-gpg]# wget https://apt.postgresql.org/pub/repos/yum/RPM-GPG-KEY-PGDG

```

```
vi /etc/yum.repos.d/etcd.repo

[etcd]
name=PostgreSQL common RPMs for RHEL / Rocky $releasever - $basearch
baseurl=http://ftp.postgresql.org/pub/repos/yum/common/pgdg-rhel8-extras/redhat/rhel-$releasever-$basearch
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-PGDG
repo_gpgcheck = 1
```

```
dnf -y install etcd
```
```
mv /etc/etcd/etcd.conf /etc/etcd/etcd.conf.orig
vi /etc/etcd/etcd.conf
```

![image](https://github.com/rajeshgandi/PostgreSQL/assets/136494079/763586c1-3a76-49c3-8b6d-7b17e5375801)

```ruby

# This is an example configuration file for an etcd cluster with 3 nodes

# specify the name of an etcd member

ETCD_NAME=p1

# Configure the data directory
ETCD_DATA_DIR="/var/lib/etcd/p1"

# Configure the listen URLs
ETCD_LISTEN_PEER_URLS="http://192.168.110.189:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.110.189:2379"

# Configure the advertise URLs
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.110.189:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.110.189:2379"

# Configure the initial cluster
ETCD_INITIAL_CLUSTER="p1=http://192.168.110.189:2380,p2=http://192.168.110.190:2380,p3=http://192.168.110.191:2380"

# Configure the initial cluster state
ETCD_INITIAL_CLUSTER_STATE="new"

# Configure the initial cluster token
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"

# Configure v2 API
ETCD_ENABLE_V2="true"

```

```
systemctl start etcd
systemctl enable etcd
systemctl status etcd

```

```
vi .bash_profile
p1=192.168.110.131
p2=192.168.110.132
p3=192.168.110.133
ENDPOINTS=$p1:2379,$p2:2379,$p3:2379
```

```
. .bash_profile
etcdctl endpoint status --write-out=table --endpoints=$ENDPOINTS

```
```
dnf -y install keepalived
```

```
vi  /etc/sysctl.conf
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
```
```
sudo sysctl --system
sudo sysctl -p
```
```
ifconfig
ens33
```

```
mv /etc/keepalived/keepalived.conf  /etc/keepalived/keepalived_bkp.conf
vi /etc/keepalived/keepalived.conf
```

![image](https://github.com/rajeshgandi/PostgreSQL/assets/136494079/591a2565-0f91-4b54-a54e-860576d30a83)      ![image](https://github.com/rajeshgandi/PostgreSQL/assets/136494079/6b73177d-f56b-477b-bf06-4d9818936b86)

``` 
p1:
vrrp_script check_haproxy {
  script "pkill -0 haproxy"
  interval 2
  weight 2
}
vrrp_instance VI_1 {
  state MASTER
  interface ens33
  virtual_router_id 51
  priority 101
  advert_int 1
  virtual_ipaddress {
    192.168.110.192
  }
  track_script {
    check_haproxy
  }
}

```

```
p2:
vrrp_script check_haproxy {
   script "pkill -0 haproxy"
   interval 2
   weight 2
}
vrrp_instance VI_1 {
  state BACKUP
  interface ens33
  virtual_router_id 51
  priority 100
  advert_int 1
  virtual_ipaddress {
    192.168.110.192
  }
  track_script {
  check_haproxy
  }
}
```

```
systemctl start keepalived
systemctl enable keepalived
systemctl status keepalived
```

```
ip addr show ens33
```
vip should appear in any of those 3.

```
dnf -y install haproxy
```

```
mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.orig
vi /etc/haproxy/haproxy.cfg
```


![image](https://github.com/rajeshgandi/PostgreSQL/assets/136494079/1369ae5e-0de4-4c2c-9007-3af054133e7c)


```ruby
global
    maxconn     1000
	
defaults
    mode                    tcp
    log                     global
    option                  tcplog
    retries                 3 
    timeout queue           1m
    timeout connect         4s
    timeout client          60m
    timeout server          60m
    timeout check           5s
    maxconn                 900

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen primary
    bind 192.168.110.192:5000
    option httpchk OPTIONS /master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server p4 192.168.110.189:5432 maxconn 100 check port 8008
    server p5 192.168.110.190:5432 maxconn 100 check port 8008
    server p6 192.168.110.191:5432 maxconn 100 check port 8008

listen standby
    bind 192.168.110.192:5001
    balance roundrobin
    option httpchk OPTIONS /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server p4 192.168.110.189:5432 maxconn 100 check port 8008
    server p5 192.168.110.190:5432 maxconn 100 check port 8008
    server p6 192.168.110.191:5432 maxconn 100 check port 8008

```

Copy same in p2 and p3

```
systemctl start haproxy
systemctl enable haproxy
systemctl status haproxy
```
```
dnf install epel-release -y
dnf --enablerepo=powertools install moreutils -y
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf -qy module disable postgresql
dnf install -y postgresql15-server
dnf install -y postgresql15-contrib
dnf install -y postgresql15-devel
```
```
yum install -y python3 python3-devel gcc
sudo ln -s /usr/local/bin/pip /bin/pip
pip3 install --upgrade pip
pip install psycopg2-binary
pip install python-etcd
python3 -m pip install patroni[etcd]
```
```
dnf -y install patroni 
dnf -y install patroni-etcd 
dnf -y install watchdog

sudo ln -s /usr/local/bin/patronictl /bin/patronictl
/usr/local/bin/patronictl –help
patronictl –help
```
```
mkdir -p /etc/patroni
vi /etc/patroni/patroni.yml
```

![image](https://github.com/rajeshgandi/PostgreSQL/assets/136494079/b4479188-684e-4660-8ee3-04277be6511a)

```
scope: postgres
namespace: /db/
name: p1 

restapi:
    listen: 192.168.110.189:8008
    connect_address: 192.168.110.189:8008

etcd3:
    hosts: 192.168.110.131:2379,192.168.110.190:2379,192.168.110.191:2379

bootstrap:
    dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
        use_pg_rewind: true
    pg_hba:
    - host replication replicator 127.0.0.1/32 md5
    - host replication replicator 192.168.110.189/0 md5
    - host replication replicator 192.168.110.190/0 md5
    - host replication replicator 192.168.110.191/0 md5
    - host all all 0.0.0.0/0 md5

postgresql:
    listen: 192.168.110.189:5432 
    connect_address: 192.168.110.189:5432
    data_dir: /u01/pgsql/15
    bin_dir: /usr/pgsql-15/bin
    authentication:
        replication:
            username: replicator
            password: replicator
        superuser:
            username: postgres
            password: postgres
    parameters:
        unix_socket_directories: '.'

watchdog:
  mode: required
  device: /dev/watchdog
  safety_margin: 5

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```
```
vi /etc/watchdog.conf
Uncomment
watchdog-device = /dev/watchdog
```
```
mknod /dev/watchdog c 10 130
modprobe softdog
chown postgres /dev/watchdog
```
```
mkdir -p /u01
chown -R postgres:postgres /u01
```
```
systemctl start patroni
```
```
tail -10f /var/log/messages
p1 patroni[8469] INFO: no action. I am (p1), the leader with the lock
p2 patroni[8964] INFO: no action. I am (p2), a secondary, and following a leader (p1)
p2 patroni[8964] INFO: no action. I am (p3), a secondary, and following a leader (p1)
```
```
patronictl -c /etc/patroni/patroni.yml list
```
```
psql -h 192.168.110.192 -p 5000
create table emp(id int, sal int);
```

```
patronictl -c /etc/patroni/patroni.yml switchover
```

vi HATest.py
```
import psycopg2
import argparse
import socket
import time

# Define command line arguments
parser = argparse.ArgumentParser()
parser.add_argument("--port", type=int, help="PostgreSQL port number")
args = parser.parse_args()

# PostgreSQL connection parameters
DB_HOST = '192.168.110.192'
DB_PORT = args.port
DB_NAME = 'postgres'
DB_USER = 'postgres'
DB_PASSWORD = 'postgres'

# Number of times to attempt connection
MAX_ATTEMPTS = 10

def connect():
    """Attempts to connect to the PostgreSQL database"""
    attempts = 0
    while attempts < MAX_ATTEMPTS:
        try:
            conn = psycopg2.connect(
                host=DB_HOST,
                port=DB_PORT,
                dbname=DB_NAME,
                user=DB_USER,
                password=DB_PASSWORD
            )
            # Print the output of SELECT pg_is_in_recovery(),inet_server_addr() after successfully connecting
            cursor = conn.cursor()
            cursor.execute("SELECT pg_is_in_recovery(),inet_server_addr()")
            result = cursor.fetchone()
            print(f"Connected to: {result[1]}")
            return conn
        except psycopg2.OperationalError:
            attempts += 1
            print(f"Connection attempt {attempts} failed. Retrying in 1 seconds...")
            time.sleep(1)
    raise Exception("Failed to connect to the PostgreSQL database after multiple attempts.")

def perform_query():
    """Performs a query based on the recovery mode of the PostgreSQL server"""
    conn = connect()
    cursor = conn.cursor()

    # Execute the appropriate SQL statement based on the recovery mode of the PostgreSQL server
    cursor.execute("SELECT pg_is_in_recovery()")
    is_in_recovery = cursor.fetchone()[0]
    if is_in_recovery:
        print("PostgreSQL server is in recovery mode. Performing SELECT query...")
        cursor.execute("SELECT COUNT(1) FROM emp")
        result = cursor.fetchone()[0]
        print(f"Count: {result}")
    else:
        print("PostgreSQL server is not in recovery mode. Performing INSERT query...")
        cursor.execute("INSERT INTO emp (id, sal) VALUES (%s, %s)", ('1', '1'))
        conn.commit()
        print("Data inserted successfully")
        cursor.execute("SELECT COUNT(1) FROM emp")
        result = cursor.fetchone()[0]
        print(f"Count: {result}")


    # Close the database connection
    conn.close()

# Loop indefinitely and perform a query based on the recovery mode of the PostgreSQL server
while True:
    try:
        perform_query()
        print("Press Ctrl+Z to terminating the job...")
        time.sleep(1)
        print()
    except:
        print("Error performing query. Retrying in 1 second...")
        time.sleep(1)
		

```

```
python3 HATest.py --port 5000
python3 HATest.py --port 5001
```
