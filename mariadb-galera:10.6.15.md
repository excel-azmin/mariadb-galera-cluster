
# Mariadb Galera 10.6.15

```
version: '3.8'

services:
  mariadb-galera-node1:
    image: docker.io/bitnami/mariadb-galera:10.6.15
    hostname: mariadb-galera-node1
    volumes:
      - mariadb_galera_data_node1:/bitnami/mariadb
    environment:
      - BITNAMI_DEBUG=yes
      # Galera
      - MARIADB_GALERA_CLUSTER_BOOTSTRAP=yes
      - MARIADB_GALERA_FORCE_SAFETOBOOTSTRAP=yes
      - MARIADB_GALERA_SST_METHOD=mariabackup
      - MARIADB_GALERA_CLUSTER_NAME=galera_cluster
      - MARIADB_GALERA_CLUSTER_ADDRESS=gcomm://mariadb-galera-node1,mariadb-galera-node2,mariadb-galera-node3
      - MARIADB_GALERA_MARIABACKUP_PASSWORD=backup-password
      # Root + initial DB ONLY on node1
      - MARIADB_ROOT_PASSWORD=root-password
      - MARIADB_USER=user01
      - MARIADB_PASSWORD=user01-password
      - MARIADB_DATABASE=my_database
      # ERPNext-friendly settings (no NO_AUTO_CREATE_USER)
      - MARIADB_EXTRA_FLAGS=--character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --skip-name-resolve=1 --sql-mode=NO_ENGINE_SUBSTITUTION --innodb-file-format=barracuda --innodb-file-per-table=1 --innodb_large_prefix=1
    networks: [galera-network]
    healthcheck:
      test: ['CMD', '/opt/bitnami/scripts/mariadb-galera/healthcheck.sh']
      interval: 15s
      timeout: 5s
      retries: 6

  mariadb-galera-node2:
    image: docker.io/bitnami/mariadb-galera:10.6.15
    hostname: mariadb-galera-node2
    volumes:
      - mariadb_galera_data_node2:/bitnami/mariadb
    environment:
      - BITNAMI_DEBUG=yes
      - MARIADB_GALERA_CLUSTER_BOOTSTRAP=no
      - MARIADB_GALERA_SST_METHOD=mariabackup
      - MARIADB_GALERA_CLUSTER_NAME=galera_cluster
      - MARIADB_GALERA_CLUSTER_ADDRESS=gcomm://mariadb-galera-node1,mariadb-galera-node2,mariadb-galera-node3
      - MARIADB_GALERA_MARIABACKUP_PASSWORD=backup-password
      - MARIADB_ROOT_PASSWORD=root-password
      - MARIADB_EXTRA_FLAGS=--character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --skip-name-resolve=1 --sql-mode=NO_ENGINE_SUBSTITUTION --innodb-file-format=barracuda --innodb-file-per-table=1 --innodb_large_prefix=1
    networks: [galera-network]
    depends_on: [mariadb-galera-node1]
    healthcheck:
      test: ['CMD', '/opt/bitnami/scripts/mariadb-galera/healthcheck.sh']
      interval: 15s
      timeout: 5s
      retries: 6

  mariadb-galera-node3:
    image: docker.io/bitnami/mariadb-galera:10.6.15
    hostname: mariadb-galera-node3
    volumes:
      - mariadb_galera_data_node3:/bitnami/mariadb
    environment:
      - BITNAMI_DEBUG=yes
      - MARIADB_GALERA_CLUSTER_BOOTSTRAP=no
      - MARIADB_GALERA_SST_METHOD=mariabackup
      - MARIADB_GALERA_CLUSTER_NAME=galera_cluster
      - MARIADB_GALERA_CLUSTER_ADDRESS=gcomm://mariadb-galera-node1,mariadb-galera-node2,mariadb-galera-node3
      - MARIADB_GALERA_MARIABACKUP_PASSWORD=backup-password
      - MARIADB_ROOT_PASSWORD=root-password
      - MARIADB_EXTRA_FLAGS=--character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --skip-name-resolve=1 --sql-mode=NO_ENGINE_SUBSTITUTION --innodb-file-format=barracuda --innodb-file-per-table=1 --innodb_large_prefix=1
    networks: [galera-network]
    depends_on: [mariadb-galera-node1, mariadb-galera-node2]
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
      - "6033:6033"
      - "6032:6032"
      - "6080:6080"
    volumes:
      - proxysql_data:/var/lib/proxysql
    configs:
      - source: proxysql
        target: /etc/proxysql.cnf:ro
    networks: [galera-network]
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
