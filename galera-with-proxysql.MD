
# Config File

```
datadir="/var/lib/proxysql"

admin_variables =
{
    admin_credentials="admin:admin;radmin:radmin"
    mysql_ifaces="0.0.0.0:6032"
    refresh_interval=2000
    web_enabled=true
    web_port=6080
}

mysql_variables =
{
    threads=4
    max_connections=2048
    interfaces="0.0.0.0:6033"
    default_query_delay=0
    default_query_timeout=36000000
    have_compress=true
    poll_timeout=2000
}

mysql_servers =
(
    { address="mariadb-galera-node1", port=3306, hostgroup=0, max_connections=100, max_replication_lag=0 },
    { address="mariadb-galera-node2", port=3306, hostgroup=0, max_connections=100, max_replication_lag=0 },
    { address="mariadb-galera-node3", port=3306, hostgroup=0, max_connections=100, max_replication_lag=0 }
)

mysql_users =
(
    { username="user01", password="user01-password", default_hostgroup=0, active=1 }
)

mysql_query_rules =
(
    {
        rule_id=1
        active=1
        match_pattern="^SELECT"
        destination_hostgroup=0
        apply=1
    }
)

```

# Compose File 

```
version: '3.8'

services:
  mariadb-galera-node1:
    image: docker.io/bitnami/mariadb-galera:11.8
    hostname: mariadb-galera-node1
    ports:
      - '3306:3306'
    volumes:
      - 'mariadb_galera_data_node1:/bitnami/mariadb'
    environment:
      - MARIADB_GALERA_CLUSTER_BOOTSTRAP=yes
      - MARIADB_ROOT_PASSWORD=root-password
      - MARIADB_GALERA_MARIABACKUP_PASSWORD=backup-password
      - MARIADB_USER=user01
      - MARIADB_PASSWORD=user01-password 
      - MARIADB_DATABASE=my_database
      - MARIADB_GALERA_CLUSTER_NAME=galera_cluster
      - MARIADB_GALERA_CLUSTER_ADDRESS=gcomm://mariadb-galera-node1,mariadb-galera-node2,mariadb-galera-node3
    networks:
      - galera-network
    healthcheck:
      test: ['CMD', '/opt/bitnami/scripts/mariadb-galera/healthcheck.sh']
      interval: 15s
      timeout: 5s
      retries: 6

  mariadb-galera-node2:
    image: docker.io/bitnami/mariadb-galera:11.8
    hostname: mariadb-galera-node2
    ports:
      - '3307:3306'
    volumes:
      - 'mariadb_galera_data_node2:/bitnami/mariadb'
    environment:
      - MARIADB_GALERA_CLUSTER_BOOTSTRAP=no
      - MARIADB_ROOT_PASSWORD=root-password
      - MARIADB_GALERA_MARIABACKUP_PASSWORD=backup-password
      - MARIADB_USER=user01
      - MARIADB_PASSWORD=user01-password 
      - MARIADB_DATABASE=my_database
      - MARIADB_GALERA_CLUSTER_NAME=galera_cluster
      - MARIADB_GALERA_CLUSTER_ADDRESS=gcomm://mariadb-galera-node1,mariadb-galera-node2,mariadb-galera-node3
    networks:
      - galera-network
    depends_on:
      - mariadb-galera-node1
    healthcheck:
      test: ['CMD', '/opt/bitnami/scripts/mariadb-galera/healthcheck.sh']
      interval: 15s
      timeout: 5s
      retries: 6

  mariadb-galera-node3:
    image: docker.io/bitnami/mariadb-galera:11.8
    hostname: mariadb-galera-node3
    ports:
      - '3308:3306'
    volumes:
      - 'mariadb_galera_data_node3:/bitnami/mariadb'
    environment:
      - MARIADB_GALERA_CLUSTER_BOOTSTRAP=no
      - MARIADB_ROOT_PASSWORD=root-password
      - MARIADB_GALERA_MARIABACKUP_PASSWORD=backup-password
      - MARIADB_USER=user01
      - MARIADB_PASSWORD=user01-password 
      - MARIADB_DATABASE=my_database
      - MARIADB_GALERA_CLUSTER_NAME=galera_cluster
      - MARIADB_GALERA_CLUSTER_ADDRESS=gcomm://mariadb-galera-node1,mariadb-galera-node2,mariadb-galera-node3
    networks:
      - galera-network
    depends_on:
      - mariadb-galera-node1
      - mariadb-galera-node2
    healthcheck:
      test: ['CMD', '/opt/bitnami/scripts/mariadb-galera/healthcheck.sh']
      interval: 15s
      timeout: 5s
      retries: 6

  proxysql:
    image: proxysql/proxysql:2.5.5
    container_name: proxysql
    restart: always
    ports:
      - "6033:6033"  # MySQL Client Port
      - "6032:6032"  # Admin Web GUI/API Port
      - "6080:6080"  # Web interface
    volumes:
      - proxysql_data:/var/lib/proxysql
    configs:
      - source: proxysql
        target: /etc/proxysql.cnf:ro
    networks:
      - galera-network
    depends_on:
      - mariadb-galera-node1
      - mariadb-galera-node2
      - mariadb-galera-node3

configs:
  proxysql:
    external: true

volumes:
  mariadb_galera_data_node1:
  mariadb_galera_data_node2:
  mariadb_galera_data_node3:
  proxysql_data:

networks:
  galera-network:
    name: galera-network
    attachable: true

```
