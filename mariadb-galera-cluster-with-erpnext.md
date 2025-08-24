# After deploy :

Ah—that error is because the ProxySQL admin interface uses a SQLite-style grammar, not MySQL’s. So ON DUPLICATE KEY UPDATE isn’t supported there. Use INSERT OR REPLACE (or do a DELETE+INSERT) and then load to runtime + save to disk.


Run these exactly in the ProxySQL admin shell (mysql -h 127.0.0.1 -P6032 -u admin -padmin):


1) Register your Galera backends


```

-- Either replace all rows:

DELETE FROM mysql_servers WHERE hostgroup_id=0;

-- Then add nodes (one row per node)
INSERT OR REPLACE INTO mysql_servers (hostgroup_id, hostname, port) VALUES (0,'mariadb-galera-node1',3306);
INSERT OR REPLACE INTO mysql_servers (hostgroup_id, hostname, port) VALUES (0,'mariadb-galera-node2',3306);
INSERT OR REPLACE INTO mysql_servers (hostgroup_id, hostname, port) VALUES (0,'mariadb-galera-node3',3306);

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;

-- sanity
SELECT * FROM runtime_mysql_servers;

```

2) Add frontend users (root for bench, user01 for app)


```
-- Add/replace root and user01

INSERT OR REPLACE INTO mysql_users
  (username,password,active,default_hostgroup,transaction_persistent,fast_forward)
VALUES
  ('root','root-password',1,0,1,0);

INSERT OR REPLACE INTO mysql_users
  (username,password,active,default_hostgroup,transaction_persistent,fast_forward)
VALUES
  ('user01','user01-password',1,0,1,0);

LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;

-- sanity
SELECT username,active,default_hostgroup FROM runtime_mysql_users;


```

(If your ProxySQL version complains about columns, use the minimal set: username,password,active,default_hostgroup and omit the others.)


3) Make sure Galera accepts root from %


On any Galera node (port 3306, directly to MariaDB):

```
CREATE USER IF NOT EXISTS 'root'@'%' IDENTIFIED BY 'root-password';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;


CREATE USER IF NOT EXISTS 'user01'@'%' IDENTIFIED BY 'user01-password';
GRANT ALL PRIVILEGES ON *.* TO 'user01'@'%';

FLUSH PRIVILEGES;
SELECT user,host FROM mysql.user WHERE user IN ('root','user01');

```

4) Quick auth test from an ERPNext container


`mysql -h proxysql -P6033 -uroot -proot-password -e "SHOW DATABASES;"`



If this fails → recheck step 2 (are users in runtime_mysql_users? password exactly right?).


If it passes → proceed with site creation:

```

bench new-site dev-erp.arcapps.org \
  --mariadb-root-username root \
  --mariadb-root-password root-password \
  --db-name my_database \
  --db-password user01-password

bench --site dev-erp.arcapps.org install-app erpnext

```


That should clear the 1045 you were seeing.


# Extra Notes : 

This config I'm able to run the apps: 

```
frappe@a4d90647657c:~/frappe-bench$ cat sites/common_site_config.json 
{
 "db_host": "mariadb-galera-node1",
 "db_port": 3306,
 "redis_cache": "redis://redis-cache:6379",
 "redis_queue": "redis://redis-queue:6379",
 "redis_socketio": "redis://redis-socketio:6379",
 "socketio_port": 9000
}

```



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
version: "3.8"

services:
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
      DB_HOST: proxysql
      DB_PORT: "6033"
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:  
      - galera-network
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
      DB_HOST: proxysql
      DB_PORT: "6033"
      REDIS_CACHE: redis-cache:6379
      REDIS_QUEUE: redis-queue:6379
      REDIS_SOCKETIO: redis-socketio:6379
      SOCKETIO_PORT: "9000"
    volumes:
      - sites:/home/frappe/frappe-bench/sites
      - logs:/home/frappe/frappe-bench/logs
    networks:
      - galera-network
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
        traefik.http.routers.erpnext-testing--http.rule: Host(${SITES:?No sites set})
        traefik.http.routers.erpnext-testing--http.entrypoints: http
        traefik.http.routers.erpnext-testing--http.middlewares: https-redirect
        traefik.http.routers.erpnext-testing--https.rule: Host(${SITES})
        traefik.http.routers.erpnext-testing--https.entrypoints: https
        traefik.http.routers.erpnext-testing--https.tls: "true"
        traefik.http.routers.erpnext-testing--https.tls.certresolver: le
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
      - galera-network
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
      - galera-network
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
      - galera-network
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
      - galera-network
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
      - galera-network
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
      - galera-network
      - erp-bench-network   

volumes:
  redis-queue-data:
  redis-cache-data:
  redis-socketio-data:
  sites:
  logs:

networks:
  erp-bench-network:
    name: erp-bench-network
    external: false
  galera-network:
    name: galera-network
    external: true
  traefik-public:
    name: traefik-public
    external: true

```
