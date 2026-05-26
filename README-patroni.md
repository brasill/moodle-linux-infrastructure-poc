# Aula 4 - Configurando o servidor patroni

## ==========================================
## 1. PREPARAÇÃO (Executar no pgha1, pgha2 e pgha3 como ROOT)
## ==========================================
# Instalar dependências do sistema
dnf install -y python3-devel
pip3 install --upgrade setuptools

# Instalar o Patroni e pacotes Python GLOBALMENTE (evita o erro 203 de permissão)
pip3 install --prefix=/usr --ignore-installed psycopg2-binary python-etcd wheel patroni==3.3.6


## ==========================================
## 2. CRIANDO O SERVIÇO (Executar no pgha1, pgha2 e pgha3 como ROOT)
## ==========================================
cat <<EOL > /usr/lib/systemd/system/patroni.service
[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/bin/patroni /etc/patroni.yaml
KillMode=process
TimeoutSec=30
Restart=no
[Install]
WantedBy=multi-user.target
EOL


## ==========================================
## 3. CONFIGURANDO O NÓ 1 (Apenas no pgha1)
## ==========================================
# vi /etc/patroni.yaml

scope: postgres
namespace: /pg_cluster/
name: pg_node1
restapi:
  listen: pgha1:8008
  connect_address: pgha1:8008
etcd3:
  hosts: 192.168.122.172:2379, 192.168.122.173:2379, 192.168.122.174:2379
  protocol: https
  cacert: /var/lib/pgsql/cert/etcd/ca_cert.pem
  cert: /var/lib/pgsql/cert/etcd/etcd_cert.pem
  key: /var/lib/pgsql/cert/etcd/etcd_key.pem

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        archive_mode: "on"
        archive_timeout: 1800s
        archive_command: mkdir -p ../wal_archive && test ! -f ../wal_archive/%f && cp %p ../wal_archive/%f
      recovery_conf:
        restore_command: cp ../wal_archive/%f %p

  initdb:
  - auth-host: scram-sha-256
  - auth-local: trust
  - encoding: UTF8
  - data-checksums
   
  post_init:
  - timescaledb-tune --quiet --yes --pg-config=/usr/pgsql-16/bin/pg_config
  
  users:
    replication:
      password: replicator
      options:
        - replication
    admin:
      password: admin
      options:
        - createrole
        - createdb

  pg_hba:
  - host replication replicator 127.0.0.1/32 scram-sha-256
  - host replication replicator 192.168.122.170/32 scram-sha-256
  - host replication replicator 192.168.122.171/32 scram-sha-256
  - host replication replicator 192.168.122.172/32 scram-sha-256
  - hostssl all all 192.168.122.0/24 scram-sha-256
  
postgresql:
  listen: pgha1:5432
  connect_address: pgha1:5432
  data_dir: /usr/local/pgsql/data/
  bin_dir: /usr/pgsql-16/bin/
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: replicator
    superuser:
      username: postgres
      password: postgres
  parameters:
    unix_socket_directories: "/var/run/postgresql"
    shared_preload_libraries: "timescaledb"
    # Configurações de Memória Otimizadas para VMs de 1.5GB RAM
    shared_buffers: 256MB
    work_mem: 4MB
    maintenance_work_mem: 128MB
    max_worker_processes: 4
    effective_cache_size: 512MB
    # Fim das configurações de memória
    fsync: on
    temp_buffers: 4MB
    max_connections: 100
    wal_level: replica
    hot_standby: "on"
    wal_keep_size: 128MB
    max_wal_senders: 10
    max_replication_slots: 10
    max_prepared_transactions: 0
    max_locks_per_transaction: 64
    wal_log_hints: "on"
    track_commit_timestamp: "on"
    timescaledb.telemetry_level: "off"
    ssl: true
    ssl_ca_file: /var/lib/pgsql/cert/root.crt
    ssl_cert_file: /var/lib/pgsql/cert/server.crt
    ssl_ciphers: HIGH:!aNULL
    ssl_crl_file: ''
    ssl_key_file: /var/lib/pgsql/cert/server.key
    ssl_min_protocol_version: TLSv1.3
    ssl_prefer_server_ciphers: true

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false


## ==========================================
## 4. GERANDO E COPIANDO CERTIFICADOS (Apenas no pgha1)
## ==========================================
# mkdir -p /var/lib/pgsql/cert/etcd
# cd /var/lib/pgsql/cert

# openssl genrsa 2048 > server.key 
# openssl req -new -key server.key -days 365 -out server.crt -x509 -subj "/C=BR/ST=SP/L=SaoCaetano/O=M3Tecnologia/CN=pgha"
# cp server.crt root.crt

# Enviar certificados PG para os nós 2 e 3 usando o usuário comum (para evitar bloqueio de root via SSH)
# scp -rp /var/lib/pgsql/cert/{root,server}.crt server.key samu@pgha2:/tmp/
# scp -rp /var/lib/pgsql/cert/{root,server}.crt server.key samu@pgha3:/tmp/

## ==========================================
## 4.1. COPIANDO CERTIFICADOS DO ETCD (Acessar o pghaproxy)
## ==========================================
# Logar no servidor HAProxy e enviar os certificados para o /tmp dos 3 nós
# scp -p /etc/etcd/cert/{ca_cert,etcd_cert,etcd_key}.pem samu@pgha1:/tmp/
# scp -p /etc/etcd/cert/{ca_cert,etcd_cert,etcd_key}.pem samu@pgha2:/tmp/
# scp -p /etc/etcd/cert/{ca_cert,etcd_cert,etcd_key}.pem samu@pgha3:/tmp/


## ==========================================
## 5. AJUSTANDO O NÓ 1 (Apenas no pgha1 logado como root)
## ==========================================
# Mover os certificados do ETCD que chegaram no /tmp
# mv /tmp/{ca_cert,etcd_cert,etcd_key}.pem /var/lib/pgsql/cert/etcd/

# Permissões do YAML
# chown -R postgres:postgres /etc/patroni.yaml
# chmod 0740 /etc/patroni.yaml

# Permissões Certificados PostgreSQL
# chown -R postgres:postgres /var/lib/pgsql/cert/
# chmod 0740 -R /var/lib/pgsql/cert/
# chmod 0600 /var/lib/pgsql/cert/server.key

# Permissões Certificados ETCD
# chown -R postgres:postgres /var/lib/pgsql/cert/etcd
# chmod 0740 -R /var/lib/pgsql/cert/etcd

## INICIAR PATRONI NO NÓ 1
# systemctl daemon-reload
# systemctl enable --now patroni.service
# systemctl status patroni

## ==========================================
## 5.1. AJUSTANDO SENHAS E USUÁRIOS NO NÓ 1
## ==========================================
# (Se o banco já existia, o Patroni não roda o bootstrap inicial. Precisamos recriar as senhas em SCRAM na mão:)
# su - postgres
# psql -c "DROP ROLE IF EXISTS replicator;"
# psql -c "CREATE ROLE replicator WITH REPLICATION LOGIN ENCRYPTED PASSWORD 'replicator';"
# psql -c "SET password_encryption = 'scram-sha-256'; ALTER ROLE postgres WITH PASSWORD 'postgres';"

## ==========================================
## 5.2. APLICANDO REGRAS DE REDE NO CLUSTER
## ==========================================
# Como as regras de 'pg_hba' do arquivo YAML só são lidas no bootstrap, precisamos injetá-las na configuração viva do cluster.
# Logado no pgha1, abra a configuração viva:
# patronictl -c /etc/patroni.yaml edit-config
#
# Adicione este bloco INTEIRO DEBAIXO de 'postgresql:', cuidando da indentação:
#
#  pg_hba:
#    - host replication replicator 127.0.0.1/32 scram-sha-256
#    - host replication replicator 192.168.122.170/32 scram-sha-256
#    - host replication replicator 192.168.122.171/32 scram-sha-256
#    - host replication replicator 192.168.122.172/32 scram-sha-256
#    - hostssl all all 192.168.122.0/24 scram-sha-256
#
# Salve e saia (:wq). O Patroni replicará a segurança para todos os nós instantaneamente.


## ==========================================
## 6. CONFIGURANDO O NÓ 2 (Apenas no pgha2 logado como root)
## ==========================================

# vi /etc/patroni.yaml
# (Cole o MESMO conteúdo do patroni.yaml do pgha1 acima, mas altere apenas estas linhas):
# name: pg_node2
# listen: pgha2:8008
# connect_address: pgha2:8008
# listen: pgha2:5432
# connect_address: pgha2:5432

# Mover os certificados que chegaram no /tmp
# mkdir -p /var/lib/pgsql/cert/etcd
# mv /tmp/{root,server}.crt /tmp/server.key /var/lib/pgsql/cert/
# mv /tmp/{ca_cert,etcd_cert,etcd_key}.pem /var/lib/pgsql/cert/etcd/

## AJUSTANDO PERMISSÕES NO NÓ 2
# chown -R postgres:postgres /etc/patroni.yaml
# chmod 0740 /etc/patroni.yaml
# chown -R postgres:postgres /var/lib/pgsql/cert/
# chmod 0740 -R /var/lib/pgsql/cert/
# chmod 0600 /var/lib/pgsql/cert/server.key
# chown -R postgres:postgres /var/lib/pgsql/cert/etcd
# chmod 0740 -R /var/lib/pgsql/cert/etcd

## INICIAR PATRONI NO NÓ 2
# systemctl daemon-reload
# systemctl restart patroni
# systemctl enable --now patroni.service
# systemctl status patroni


## ==========================================
## 7. CONFIGURANDO O NÓ 3 (Apenas no pgha3 logado como root)
## ==========================================

# vi /etc/patroni.yaml
# (Cole o MESMO conteúdo do patroni.yaml do pgha1 acima, mas altere apenas estas linhas):
# name: pg_node3
# listen: pgha3:8008
# connect_address: pgha3:8008
# listen: pgha3:5432
# connect_address: pgha3:5432

# Mover os certificados que chegaram no /tmp
# mkdir -p /var/lib/pgsql/cert/etcd
# mv /tmp/{root,server}.crt /tmp/server.key /var/lib/pgsql/cert/
# mv /tmp/{ca_cert,etcd_cert,etcd_key}.pem /var/lib/pgsql/cert/etcd/

## AJUSTANDO PERMISSÕES NO NÓ 3
# chown -R postgres:postgres /etc/patroni.yaml
# chmod 0740 /etc/patroni.yaml
# chown -R postgres:postgres /var/lib/pgsql/cert/
# chmod 0740 -R /var/lib/pgsql/cert/
# chmod 0600 /var/lib/pgsql/cert/server.key
# chown -R postgres:postgres /var/lib/pgsql/cert/etcd
# chmod 0740 -R /var/lib/pgsql/cert/etcd

## INICIAR PATRONI NO NÓ 3
# systemctl daemon-reload
# systemctl restart patroni
# systemctl enable --now patroni.service
# systemctl status patroni


## ==========================================
## 8. VALIDAÇÃO DO CLUSTER
## ==========================================

# (Em qualquer um dos nós, logue como postgres)
# su - postgres
$ patronictl -c /etc/patroni.yaml list

# Validando conexão criptografada (SSL)
# psql "host=192.168.122.170 port=5432 user=postgres dbname=postgres sslmode=require"
postgres=# SELECT * FROM pg_stat_ssl;