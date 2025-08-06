# mariadb-galera-cluster

```

version: '3.8'

services:
  mariadb-galera-node1:
    image: docker.io/bitnami/mariadb-galera:11.8
    hostname: mariadb-galera-node1
    ports:
      - '3306:3306'
      - '4444:4444'
      - '4567:4567'
      - '4568:4568'
    volumes:
      - 'mariadb_galera_data_node1:/bitnami/mariadb'
    environment:
      - MARIADB_GALERA_CLUSTER_BOOTSTRAP=yes
      - MARIADB_GALERA_FORCE_SAFETOBOOTSTRAP=yes
      - MARIADB_ROOT_PASSWORD=root-password
      - MARIADB_GALERA_MARIABACKUP_PASSWORD=backup-password
      - MARIADB_USER=user01
      - MARIADB_DATABASE=my_database
      - MARIADB_GALERA_CLUSTER_NAME=galera_cluster
      - MARIADB_GALERA_CLUSTER_ADDRESS=gcomm://mariadb-galera-node1,mariadb-galera-node2,mariadb-galera-node3
      - MARIADB_GALERA_NODE_ADDRESS=mariadb-galera-node1
      - MARIADB_ENABLE_LDAP=yes
      - LDAP_URI=ldap://openldap:1389
      - LDAP_BASE=dc=example,dc=org
      - LDAP_BIND_DN=cn=admin,dc=example,dc=org
      - LDAP_BIND_PASSWORD=adminpassword
    networks:
      - galera-network
    healthcheck:
      test: ['CMD', '/opt/bitnami/scripts/mariadb-galera/healthcheck.sh']
      interval: 15s
      timeout: 5s
      retries: 6
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure

  mariadb-galera-node2:
    image: docker.io/bitnami/mariadb-galera:11.8
    hostname: mariadb-galera-node2
    ports:
      - '3307:3306'
      - '4445:4444'
      - '4569:4567'
      - '4570:4568'
    volumes:
      - 'mariadb_galera_data_node2:/bitnami/mariadb'
    environment:
      - MARIADB_GALERA_CLUSTER_BOOTSTRAP=no
      - MARIADB_GALERA_SST_RETRY=1
      - MARIADB_ROOT_PASSWORD=root-password
      - MARIADB_GALERA_MARIABACKUP_PASSWORD=backup-password
      - MARIADB_USER=user01
      - MARIADB_DATABASE=my_database
      - MARIADB_GALERA_CLUSTER_NAME=galera_cluster
      - MARIADB_GALERA_CLUSTER_ADDRESS=gcomm://mariadb-galera-node1,mariadb-galera-node2,mariadb-galera-node3
      - MARIADB_GALERA_NODE_ADDRESS=mariadb-galera-node2
      - MARIADB_ENABLE_LDAP=yes
      - LDAP_URI=ldap://openldap:1389
      - LDAP_BASE=dc=example,dc=org
      - LDAP_BIND_DN=cn=admin,dc=example,dc=org
      - LDAP_BIND_PASSWORD=adminpassword
    networks:
      - galera-network
    depends_on:
      - mariadb-galera-node1
    healthcheck:
      test: ['CMD', '/opt/bitnami/scripts/mariadb-galera/healthcheck.sh']
      interval: 15s
      timeout: 5s
      retries: 6
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure

  mariadb-galera-node3:
    image: docker.io/bitnami/mariadb-galera:11.8
    hostname: mariadb-galera-node3
    ports:
      - '3308:3306'
      - '4446:4444'
      - '4571:4567'
      - '4572:4568'
    volumes:
      - 'mariadb_galera_data_node3:/bitnami/mariadb'
    environment:
      - MARIADB_GALERA_CLUSTER_BOOTSTRAP=no
      - MARIADB_GALERA_SST_RETRY=1
      - MARIADB_ROOT_PASSWORD=root-password
      - MARIADB_GALERA_MARIABACKUP_PASSWORD=backup-password
      - MARIADB_USER=user01
      - MARIADB_DATABASE=my_database
      - MARIADB_GALERA_CLUSTER_NAME=galera_cluster
      - MARIADB_GALERA_CLUSTER_ADDRESS=gcomm://mariadb-galera-node1,mariadb-galera-node2,mariadb-galera-node3
      - MARIADB_GALERA_NODE_ADDRESS=mariadb-galera-node3
      - MARIADB_ENABLE_LDAP=yes
      - LDAP_URI=ldap://openldap:1389
      - LDAP_BASE=dc=example,dc=org
      - LDAP_BIND_DN=cn=admin,dc=example,dc=org
      - LDAP_BIND_PASSWORD=adminpassword
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
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure

  openldap:
    image: 'docker.io/bitnami/openldap:latest'
    hostname: openldap
    ports:
      - '1389:1389'
    environment:
      - LDAP_ADMIN_USERNAME=admin
      - LDAP_ADMIN_PASSWORD=adminpassword
      - LDAP_USERS=user01
      - LDAP_PASSWORDS=password1
    volumes:
      - 'openldap_data:/bitnami/openldap'
    networks:
      - galera-network

volumes:
  mariadb_galera_data_node1:
  mariadb_galera_data_node2:
  mariadb_galera_data_node3:
  openldap_data:
  
networks:
  galera-network:
    name: galera-network
    attachable: true

```
