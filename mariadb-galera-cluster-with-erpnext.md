# Config Compose:

```
datadir="/var/lib/proxysql"

admin_variables =
{
    admin_credentials="admin:admin"
    mysql_ifaces="0.0.0.0:6032"
}

mysql_variables =
{
    threads=4
    max_connections=2048
}

mysql_servers =
(
    { address = "mariadb-galera-node1", port = 3306, hostgroup = 0 },
    { address = "mariadb-galera-node2", port = 3306, hostgroup = 0 },
    { address = "mariadb-galera-node3", port = 3306, hostgroup = 0 }
)

mysql_users =
(
    { username = "my_user", password = "oHY8kNc2qKuuNJTKPjpk", default_hostgroup = 0, active = 1 }
)

mysql_query_rules =
(
    {
        rule_id=1
        active=1
        match_pattern="^SELECT"
        destination_hostgroup=0
        apply=1
    },
    {
        rule_id=2
        active=1
        match_pattern="."
        destination_hostgroup=0
        apply=1
    }
)

```


# Stack Compose : 

```
version: "3.7"

services:
  mariadb-galera-node1:
    image: docker.io/bitnami/mariadb-galera:11.8
    hostname: mariadb-galera-node1
    deploy:
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: '2.0'
          memory: 8G
        reservations:
          cpus: '1.0'
          memory: 4G
    volumes:
      - 'mariadb_galera_data_node1:/bitnami/mariadb'
    environment:
      - MARIADB_GALERA_CLUSTER_BOOTSTRAP=yes
      - MARIADB_GALERA_FORCE_SAFETOBOOTSTRAP=yes
      - MARIADB_ROOT_PASSWORD=oHY8kNc2qKuuNJTKPjpk
      - MARIADB_GALERA_MARIABACKUP_PASSWORD=oHY8kNc2qKuuNJTKPjpk
      - MARIADB_USER=my_user
      - MARIADB_PASSWORD=oHY8kNc2qKuuNJTKPjpk
      - MARIADB_GALERA_CLUSTER_NAME=galera_cluster
      - MARIADB_GALERA_CLUSTER_ADDRESS=gcomm://mariadb-galera-node1,mariadb-galera-node2,mariadb-galera-node3
      - MARIADB_GALERA_NODE_ADDRESS=mariadb-galera-node1
      - MARIADB_CHARACTER_SET=utf8mb4
      - MARIADB_COLLATE=utf8mb4_unicode_ci
      - MARIADB_EXTRA_FLAGS=--skip-character-set-client-handshake
    networks:
      - erp-db-network
    healthcheck:
      test: ['CMD', '/opt/bitnami/scripts/mariadb-galera/healthcheck.sh']
      interval: 15s
      timeout: 5s
      retries: 6

  mariadb-galera-node2:
    image: docker.io/bitnami/mariadb-galera:11.8
    hostname: mariadb-galera-node2
    deploy:
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: '2.0'
          memory: 8G
        reservations:
          cpus: '1.0'
          memory: 4G
    volumes:
      - 'mariadb_galera_data_node2:/bitnami/mariadb'
    environment:
      - MARIADB_GALERA_CLUSTER_BOOTSTRAP=no
      - MARIADB_GALERA_SST_RETRY=1
      - MARIADB_ROOT_PASSWORD=oHY8kNc2qKuuNJTKPjpk
      - MARIADB_GALERA_MARIABACKUP_PASSWORD=oHY8kNc2qKuuNJTKPjpk
      - MARIADB_USER=my_user
      - MARIADB_PASSWORD=oHY8kNc2qKuuNJTKPjpk
      - MARIADB_GALERA_CLUSTER_NAME=galera_cluster
      - MARIADB_GALERA_CLUSTER_ADDRESS=gcomm://mariadb-galera-node1,mariadb-galera-node2,mariadb-galera-node3
      - MARIADB_GALERA_NODE_ADDRESS=mariadb-galera-node2
      - MARIADB_CHARACTER_SET=utf8mb4
      - MARIADB_COLLATE=utf8mb4_unicode_ci
      - MARIADB_EXTRA_FLAGS=--skip-character-set-client-handshake
    networks:
      - erp-db-network
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
    deploy:
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: '2.0'
          memory: 8G
        reservations:
          cpus: '1.0'
          memory: 4G
    volumes:
      - 'mariadb_galera_data_node3:/bitnami/mariadb'
    environment:
      - MARIADB_GALERA_CLUSTER_BOOTSTRAP=no
      - MARIADB_GALERA_SST_RETRY=1
      - MARIADB_ROOT_PASSWORD=oHY8kNc2qKuuNJTKPjpk
      - MARIADB_GALERA_MARIABACKUP_PASSWORD=oHY8kNc2qKuuNJTKPjpk
      - MARIADB_USER=my_user
      - MARIADB_PASSWORD=oHY8kNc2qKuuNJTKPjpk
      - MARIADB_GALERA_CLUSTER_NAME=galera_cluster
      - MARIADB_GALERA_CLUSTER_ADDRESS=gcomm://mariadb-galera-node1,mariadb-galera-node2,mariadb-galera-node3
      - MARIADB_GALERA_NODE_ADDRESS=mariadb-galera-node3
      - MARIADB_CHARACTER_SET=utf8mb4
      - MARIADB_COLLATE=utf8mb4_unicode_ci
      - MARIADB_EXTRA_FLAGS=--skip-character-set-client-handshake
    networks:
      - erp-db-network
    depends_on:
      - mariadb-galera-node1
      - mariadb-galera-node2
    healthcheck:
      test: ['CMD', '/opt/bitnami/scripts/mariadb-galera/healthcheck.sh']
      interval: 15s
      timeout: 5s
      retries: 6

  # ProxySQL Load Balancer
  proxysql:
    image: proxysql/proxysql:2.5.5
    deploy:
      restart_policy:
        condition: always
    ports:
      - "6033:6033"  # MySQL Client Port
      - "6032:6032"  # Admin Web GUI/API Port
    volumes:
      - proxysql_data:/var/lib/proxysql
    configs:
      - source: proxysql
        target: /etc/proxysql.cnf:ro
    networks:
      - erp-db-network
    depends_on:
      - mariadb-galera-node1
      - mariadb-galera-node2
      - mariadb-galera-node3
  
  backend:
    image: registry.gitlab.com/arcapps/arc-container-registry/erpnext-v14:v2.0.5
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
      replicas: 3
      update_config:
        parallelism: 1
      rollback_config:
        parallelism: 1
      restart_policy:
        delay: 10s
        max_attempts: 3
    environment:
      GUNICORN_CMD_ARGS: "--workers 9 --max-requests 5000 --max-requests-jitter 1000"
      DB_HOST: proxysql  # Changed from mariadb-master to proxysql
      DB_PORT: "6033"    # Changed to ProxySQL port
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:  
      - erp-db-network
      - erp-bench-network
    depends_on:
      - proxysql

  configurator:
    image: registry.gitlab.com/arcapps/arc-container-registry/erpnext-v14:v2.0.5
    deploy:
      restart_policy:
        condition: none  
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
        bench set-config -g redis_socketio "redis://$$REDIS_SOCKETIO";
        bench set-config -gp socketio_port $$SOCKETIO_PORT;
    environment:
      DB_HOST: proxysql    # Changed from mariadb-master to proxysql
      DB_PORT: "6033"      # Changed to ProxySQL port
      REDIS_CACHE: redis-cache:6379
      REDIS_QUEUE: redis-queue:6379
      REDIS_SOCKETIO: redis-socketio:6379
      SOCKETIO_PORT: "9000"
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:
      - erp-db-network
    depends_on:
      - proxysql
  
  frontend:
    image: registry.gitlab.com/arcapps/arc-container-registry/erpnext-v14:v2.0.5
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure  
      labels:
        traefik.docker.network: traefik-public
        traefik.enable: "true"
        traefik.constraint-label: traefik-public
        # Change router name prefix from erpnext- to the name of stack in case of multi bench setup
        traefik.http.routers.erpnext-testing--http.rule: Host(${SITES:?No sites set})
        traefik.http.routers.erpnext-testing--http.entrypoints: http
        # Remove following lines in case of local setup
        traefik.http.routers.erpnext-testing--http.middlewares: https-redirect
        traefik.http.routers.erpnext-testing--https.rule: Host(${SITES})
        traefik.http.routers.erpnext-testing--https.entrypoints: https
        traefik.http.routers.erpnext-testing--https.tls: "true"
        traefik.http.routers.erpnext-testing--https.tls.certresolver: le
        # Remove above lines in case of local setup
        traefik.http.services.erpnext-testing-.loadbalancer.server.port: "8080"    

    command:
      - nginx-entrypoint.sh
    environment:
      BACKEND: backend:8000
      FRAPPE_SITE_NAME_HEADER: $$host
      SOCKETIO: websocket:9000
      UPSTREAM_REAL_IP_ADDRESS: 127.0.0.1
      UPSTREAM_REAL_IP_HEADER: X-Forwarded-For
      UPSTREAM_REAL_IP_RECURSIVE: "off"
      PROXY_READ_TIMEOUT: 120
      CLIENT_MAX_BODY_SIZE: 50m
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:
      - erp-db-network
      - erp-bench-network
      - traefik-public 

  queue-default:
    image: registry.gitlab.com/arcapps/arc-container-registry/erpnext-v14:v2.0.5
    deploy:
      replicas: 4
      resources:
        limits:
          cpus: '1.0'
          memory: 2G
      restart_policy:
        condition: on-failure  
    command:
      - bench
      - worker
      - --queue
      - default
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:  
      - erp-db-network
      - erp-bench-network   

  queue-long:
    image: registry.gitlab.com/arcapps/arc-container-registry/erpnext-v14:v2.0.5
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure  
    command:
      - bench
      - worker
      - --queue
      - long
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:  
      - erp-db-network
      - erp-bench-network   

  queue-short:
    image: registry.gitlab.com/arcapps/arc-container-registry/erpnext-v14:v2.0.5
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure  
    command:
      - bench
      - worker
      - --queue
      - short
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:  
      - erp-db-network
      - erp-bench-network   

  redis-queue:
    image: redis:6.2-alpine
    deploy:
      restart_policy:
        condition: on-failure  
    volumes:
      - redis-queue-data:/data
    networks:  
      - erp-bench-network  

  redis-cache:
    image: redis:6.2-alpine
    command: redis-server --maxmemory 8gb --maxmemory-policy allkeys-lru
    deploy:
      resources:
        limits:
          memory: 10G
      restart_policy:
        condition: on-failure  
    volumes:
      - redis-cache-data:/data
    networks:  
      - erp-bench-network    

  redis-socketio:
    image: redis:6.2-alpine
    deploy:
      restart_policy:
        condition: on-failure  
    volumes:
      - redis-socketio-data:/data
    networks:  
      - erp-bench-network  

  scheduler:
    image: registry.gitlab.com/arcapps/arc-container-registry/erpnext-v14:v2.0.5
    deploy:
      restart_policy:
        condition: on-failure  
    command:
      - bench
      - schedule
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:  
      - erp-db-network
      - erp-bench-network   

  websocket:
    image: registry.gitlab.com/arcapps/arc-container-registry/erpnext-v14:v2.0.5
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 2G
      restart_policy:
        condition: on-failure  
    command:
      - node
      - /home/frappe/frappe-bench/apps/frappe/socketio.js
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:  
      - erp-db-network
      - erp-bench-network   

configs:
  proxysql:
    external: true


volumes:
  db-data:
  redis-queue-data:
  redis-cache-data:
  redis-socketio-data:
  sites:
  logs:
  mariadb_galera_data_node1:
  mariadb_galera_data_node2:
  mariadb_galera_data_node3:
  proxysql_data:

networks:
  erp-bench-network:
    name: erp-bench-network
    external: false
  erp-db-network:
    name: erp-db-network
    external: false
  traefik-public:
    name: traefik-public
    external: true


```
