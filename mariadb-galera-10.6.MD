# Config file 

```
datadir="/var/lib/proxysql"

admin_variables=
{
    admin_credentials="admin:admin;radmin:radmin"
    mysql_ifaces="0.0.0.0:6032"
    refresh_interval=2000
    web_enabled=true
    web_port=6080
    stats_credentials="stats:stats"
}

mysql_variables=
{
    threads=4
    max_connections=2048
    interfaces="0.0.0.0:6033"
    default_query_delay=0
    default_query_timeout=36000000
    have_compress=true
    poll_timeout=2000
    sessions_sort=true
    connect_timeout_server=3000
    monitor_username="user01"
    monitor_password="user01-password"
    monitor_history=600000
    monitor_connect_interval=60000
    monitor_ping_interval=10000
    monitor_read_only_interval=1500
    monitor_read_only_timeout=500
    ping_interval_server_msec=120000
    ping_timeout_server=500
    commands_stats=true
    query_digests=true
    query_digests_lowercase=true
}

# Define hostgroups: 0=writer, 1=reader
mysql_servers=
(
    # Writer group (hostgroup 0) - all nodes can write in Galera
    { address="mariadb-galera-node1", port=3306, hostgroup=0, weight=900, max_connections=200, max_replication_lag=0 },
    { address="mariadb-galera-node2", port=3306, hostgroup=0, weight=900, max_connections=200, max_replication_lag=0 },
    { address="mariadb-galera-node3", port=3306, hostgroup=0, weight=900, max_connections=200, max_replication_lag=0 },
    
    # Reader group (hostgroup 1) - same nodes for read load balancing
    { address="mariadb-galera-node1", port=3306, hostgroup=1, weight=900, max_connections=200, max_replication_lag=0 },
    { address="mariadb-galera-node2", port=3306, hostgroup=1, weight=900, max_connections=200, max_replication_lag=0 },
    { address="mariadb-galera-node3", port=3306, hostgroup=1, weight=900, max_connections=200, max_replication_lag=0 }
)

mysql_users=
(
    { username="user01", password="user01-password", default_hostgroup=0, max_connections=500, active=1 },
    { username="root", password="root-password", default_hostgroup=0, max_connections=100, active=1 }
)

# Optimized query routing rules for ERPNext
mysql_query_rules=
(
    # Route SELECT queries with FOR UPDATE/LOCK to writers (critical for ERPNext transactions)
    {
        rule_id=1
        active=1
        match_pattern="^SELECT.*FOR UPDATE.*"
        destination_hostgroup=0
        apply=1
    },
    
    {
        rule_id=2
        active=1
        match_pattern="^SELECT.*LOCK IN SHARE MODE.*"
        destination_hostgroup=0
        apply=1
    },
    
    # Route transaction control statements to writers
    {
        rule_id=3
        active=1
        match_pattern="^BEGIN.*"
        destination_hostgroup=0
        apply=1
    },
    
    {
        rule_id=4
        active=1
        match_pattern="^START TRANSACTION.*"
        destination_hostgroup=0
        apply=1
    },
    
    {
        rule_id=5
        active=1
        match_pattern="^COMMIT.*"
        destination_hostgroup=0
        apply=1
    },
    
    {
        rule_id=6
        active=1
        match_pattern="^ROLLBACK.*"
        destination_hostgroup=0
        apply=1
    },
    
    # Route write operations to writers
    {
        rule_id=7
        active=1
        match_pattern="^INSERT.*"
        destination_hostgroup=0
        apply=1
    },
    
    {
        rule_id=8
        active=1
        match_pattern="^UPDATE.*"
        destination_hostgroup=0
        apply=1
    },
    
    {
        rule_id=9
        active=1
        match_pattern="^DELETE.*"
        destination_hostgroup=0
        apply=1
    },
    
    {
        rule_id=10
        active=1
        match_pattern="^REPLACE.*"
        destination_hostgroup=0
        apply=1
    },
    
    # Route DDL statements to writers
    {
        rule_id=11
        active=1
        match_pattern="^CREATE.*"
        destination_hostgroup=0
        apply=1
    },
    
    {
        rule_id=12
        active=1
        match_pattern="^ALTER.*"
        destination_hostgroup=0
        apply=1
    },
    
    {
        rule_id=13
        active=1
        match_pattern="^DROP.*"
        destination_hostgroup=0
        apply=1
    },
    
    {
        rule_id=14
        active=1
        match_pattern="^TRUNCATE.*"
        destination_hostgroup=0
        apply=1
    },
    
    # Route session/connection management to writers
    {
        rule_id=15
        active=1
        match_pattern="^SET.*"
        destination_hostgroup=0
        apply=1
    },
    
    # Route SHOW commands that might be session-specific to writers
    {
        rule_id=16
        active=1
        match_pattern="^SHOW (VARIABLES|STATUS|PROCESSLIST).*"
        destination_hostgroup=0
        apply=1
    },
    
    # Route all other SELECT queries to readers for load balancing
    {
        rule_id=17
        active=1
        match_pattern="^SELECT.*"
        destination_hostgroup=1
        apply=1
    }
)

```


# Compose file 

```
version: "3.8"

services:
  # MariaDB Galera Cluster Node 1
  mariadb-galera-node1:
    image: docker.io/bitnami/mariadb-galera:10.6
    hostname: mariadb-galera-node1
    ports:
      - "3306:3306"
    volumes:
      - mariadb_galera_data_node1:/bitnami/mariadb
    environment:
      - MARIADB_GALERA_FORCE_SAFETOBOOTSTRAP=yes
      - MARIADB_GALERA_CLUSTER_BOOTSTRAP=yes
      - MARIADB_ROOT_PASSWORD=root-password
      - MARIADB_GALERA_MARIABACKUP_PASSWORD=backup-password
      - MARIADB_USER=user01
      - MARIADB_PASSWORD=user01-password
      - MARIADB_DATABASE=my_database
      - MARIADB_GALERA_CLUSTER_NAME=galera_cluster
      - MARIADB_GALERA_CLUSTER_ADDRESS=gcomm://mariadb-galera-node1,mariadb-galera-node2,mariadb-galera-node3
      - MARIADB_EXTRA_FLAGS=--character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --skip-character-set-client-handshake --skip-innodb-read-only-compressed --innodb-buffer-pool-size=1G --innodb-log-file-size=256M --max-connections=500
    networks:
      galera-network:
        aliases:
          - mariadb-galera-node1
    healthcheck:
      test: ["CMD", "/opt/bitnami/scripts/mariadb-galera/healthcheck.sh"]
      interval: 15s
      timeout: 5s
      retries: 6

  # MariaDB Galera Cluster Node 2
  mariadb-galera-node2:
    image: docker.io/bitnami/mariadb-galera:10.6
    hostname: mariadb-galera-node2
    ports:
      - "3307:3306"
    volumes:
      - mariadb_galera_data_node2:/bitnami/mariadb
    environment:
      - MARIADB_GALERA_CLUSTER_BOOTSTRAP=no
      - MARIADB_ROOT_PASSWORD=root-password
      - MARIADB_GALERA_MARIABACKUP_PASSWORD=backup-password
      - MARIADB_USER=user01
      - MARIADB_PASSWORD=user01-password
      - MARIADB_DATABASE=my_database
      - MARIADB_GALERA_CLUSTER_NAME=galera_cluster
      - MARIADB_GALERA_CLUSTER_ADDRESS=gcomm://mariadb-galera-node1,mariadb-galera-node2,mariadb-galera-node3
      - MARIADB_EXTRA_FLAGS=--character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --skip-character-set-client-handshake --skip-innodb-read-only-compressed --innodb-buffer-pool-size=1G --innodb-log-file-size=256M --max-connections=500
    networks:
      galera-network:
        aliases:
          - mariadb-galera-node2
    depends_on:
      - mariadb-galera-node1
    healthcheck:
      test: ["CMD", "/opt/bitnami/scripts/mariadb-galera/healthcheck.sh"]
      interval: 15s
      timeout: 5s
      retries: 6

  # MariaDB Galera Cluster Node 3
  mariadb-galera-node3:
    image: docker.io/bitnami/mariadb-galera:10.6
    hostname: mariadb-galera-node3
    ports:
      - "3308:3306"
    volumes:
      - mariadb_galera_data_node3:/bitnami/mariadb
    environment:
      - MARIADB_GALERA_CLUSTER_BOOTSTRAP=no
      - MARIADB_ROOT_PASSWORD=root-password
      - MARIADB_GALERA_MARIABACKUP_PASSWORD=backup-password
      - MARIADB_USER=user01
      - MARIADB_PASSWORD=user01-password
      - MARIADB_DATABASE=my_database
      - MARIADB_GALERA_CLUSTER_NAME=galera_cluster
      - MARIADB_GALERA_CLUSTER_ADDRESS=gcomm://mariadb-galera-node1,mariadb-galera-node2,mariadb-galera-node3
      - MARIADB_EXTRA_FLAGS=--character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --skip-character-set-client-handshake --skip-innodb-read-only-compressed --innodb-buffer-pool-size=1G --innodb-log-file-size=256M --max-connections=500
    networks:
      galera-network:
        aliases:
          - mariadb-galera-node3
    depends_on:
      - mariadb-galera-node1
      - mariadb-galera-node2
    healthcheck:
      test: ["CMD", "/opt/bitnami/scripts/mariadb-galera/healthcheck.sh"]
      interval: 15s
      timeout: 5s
      retries: 6

  # ProxySQL Load Balancer - with inline config
  proxysql:
    image: proxysql/proxysql:2.5.5
    ports:
      - "6033:6033"  # MySQL client port
      - "6032:6032"  # Admin (MySQL protocol)
      # 6080 is not used by stock ProxySQL; remove unless you run a sidecar UI
    configs:
      - source: proxysql
        target: /etc/proxysql.cnf
    volumes:
      - proxysql_data:/var/lib/proxysql
    networks:
      galera-network:
        aliases:
          - proxysql
    depends_on:
      - mariadb-galera-node1
      - mariadb-galera-node2
      - mariadb-galera-node3
    healthcheck:
      test: ["CMD", "mysql", "-h", "127.0.0.1", "-P", "6033", "-u", "root", "-proot-password", "-e", "SELECT 1"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s
    restart: unless-stopped

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
    attachable: true

```   

# ERPNext 

```
version: "3.8"

x-customizable-image: &customizable_image
  # image: registry.gitlab.com/arcapps/arc-container-registry/erpnext-v14:v2.0.6
  image: arcapps/erp:latest
  deploy:
    update_config:
      parallelism: 1
    rollback_config:
      parallelism: 1

x-backend-defaults: &backend_defaults
  <<: [*customizable_image]
  volumes:
    - sites:/home/frappe/frappe-bench/sites
  deploy:
    replicas: 1
    update_config:
      parallelism: 1
    rollback_config:
      parallelism: 1

services:

  redis-cache:
    image: redis:6.2-alpine
    volumes:
      - redis-cache-data:/data
    networks:
      - galera-network
    deploy:
      replicas: 1

  redis-queue:
    image: redis:6.2-alpine
    volumes:
      - redis-queue-data:/data
    networks:
      - galera-network
    deploy:
      replicas: 1

  configurator:
    <<: *backend_defaults
    entrypoint:
      - bash
      - -c
    command:
      - >
        ls -1 apps > sites/apps.txt;
        bench set-config -g db_host $$DB_HOST;
        bench set-config -gp db_port $$DB_PORT;
        bench set-config -g redis_cache "redis://$$REDIS_CACHE";
        bench set-config -g redis_queue "redis://$$REDIS_QUEUE";
        bench set-config -g redis_socketio "redis://$$REDIS_QUEUE";
        bench set-config -gp socketio_port $$SOCKETIO_PORT;
    environment:
      DB_HOST: proxysql
      DB_PORT: 6033
      REDIS_CACHE: redis-cache:6379
      REDIS_QUEUE: redis-queue:6379
      SOCKETIO_PORT: 9000
    networks:
      - galera-network
    deploy:
      replicas: 1
      restart_policy:
        condition: none

  backend:
    <<: *backend_defaults
    networks:
      - galera-network
    deploy:
      replicas: 1

  frontend:
    <<: *customizable_image
    command:
      - nginx-entrypoint.sh
    environment:
      BACKEND: backend:8000
      FRAPPE_SITE_NAME_HEADER: $$host
      SOCKETIO: websocket:9000
      UPSTREAM_REAL_IP_ADDRESS: ${UPSTREAM_REAL_IP_ADDRESS:-127.0.0.1}
      UPSTREAM_REAL_IP_HEADER: ${UPSTREAM_REAL_IP_HEADER:-X-Forwarded-For}
      UPSTREAM_REAL_IP_RECURSIVE: ${UPSTREAM_REAL_IP_RECURSIVE:-off}
      PROXY_READ_TIMEOUT: ${PROXY_READ_TIMEOUT:-120}
      CLIENT_MAX_BODY_SIZE: ${CLIENT_MAX_BODY_SIZE:-50m}
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:
      - galera-network
    ports:
      - "9630:8080"
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
        delay: 5s
      
    depends_on:
      - backend
      - websocket

  websocket:
    <<: [*customizable_image]
    command:
      - node
      - /home/frappe/frappe-bench/apps/frappe/socketio.js
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:
      - galera-network
    deploy:
      replicas: 1

  queue-short:
    <<: *backend_defaults
    command: bench worker --queue short,default
    networks:
      - galera-network
    deploy:
      replicas: 1

  queue-long:
    <<: *backend_defaults
    command: bench worker --queue long,default,short
    networks:
      - galera-network
    deploy:
      replicas: 1

  scheduler:
    <<: *backend_defaults
    command: bench schedule
    networks:
      - galera-network
    deploy:
      replicas: 1

volumes:
  sites:
  redis-cache-data:
  redis-queue-data:

networks:
  galera-network:
    attachable: true


```

