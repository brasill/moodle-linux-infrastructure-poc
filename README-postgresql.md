# PostgreSQL 16 — Configuração Inicial (Alta Disponibilidade)

> Guia de configuração inicial dos nós de banco de dados e proxy para um cluster PostgreSQL 16 com suporte a TimescaleDB. Base para instalação do Patroni em etapas posteriores.

---

## Índice

- [Topologia do Laboratório](#topologia-do-laboratório)
- [Matriz de Instalação de Pacotes](#matriz-de-instalação-de-pacotes)
- [Etapa 1 — Resolução de Nomes (hosts)](#etapa-1--resolução-de-nomes-hosts)
- [Etapa 2 — Desabilitar SELinux e Firewall](#etapa-2--desabilitar-selinux-e-firewall)
- [Etapa 3 — Repositório PGDG](#etapa-3--repositório-pgdg)
- [Etapa 4 — Instalação do PostgreSQL 16](#etapa-4--instalação-do-postgresql-16)
- [Etapa 5 — Symlinks e Estrutura de Diretórios](#etapa-5--symlinks-e-estrutura-de-diretórios)
- [Etapa 6 — Desabilitar o Serviço PostgreSQL](#etapa-6--desabilitar-o-serviço-postgresql)
- [Etapa 7 — Repositório e Instalação do TimescaleDB](#etapa-7--repositório-e-instalação-do-timescaledb)

---

## Topologia do Laboratório

| Host          | IP              | Função                        |
|---------------|-----------------|-------------------------------|
| `pgha1`       | 192.168.15.170  | Banco de dados primário/réplica |
| `pgha2`       | 192.168.15.171  | Banco de dados primário/réplica |
| `pgha3`       | 192.168.15.172  | Banco de dados primário/réplica |
| `pghaproxy`   | 192.168.15.173  | Proxy de conexão (HAProxy)    |
| `pghaproxy2`  | 192.168.15.174  | Proxy de conexão (HAProxy)    |

---

## Matriz de Instalação de Pacotes

> Consulte esta tabela antes de executar qualquer etapa para não instalar pacotes desnecessários nos proxies.

| Pacote / Ação                    | pgha1 | pgha2 | pgha3 | pghaproxy | pghaproxy2 |
|----------------------------------|:-----:|:-----:|:-----:|:---------:|:----------:|
| `postgresql16-server postgresql16` | ✅   | ✅    | ✅    | ❌        | ❌         |
| `postgresql16` (client apenas)   | ❌    | ❌    | ❌    | ✅        | ✅         |
| Estrutura `/usr/local/pgsql`     | ✅    | ✅    | ✅    | ❌        | ❌         |
| Symlinks `/usr/pgsql-16/bin/*`   | ✅    | ✅    | ✅    | ✅        | ✅         |
| TimescaleDB                      | ✅    | ✅    | ✅    | ❌        | ❌         |

---

## Etapa 1 — Resolução de Nomes (hosts)

> **Executar em TODAS as máquinas.**

Adicionar entradas no `/etc/hosts` para que os nós se comuniquem por nome, sem depender de DNS externo:

```bash
vim /etc/hosts
```

```
192.168.15.170  pgha1
192.168.15.171  pgha2
192.168.15.172  pgha3
192.168.15.173  pghaproxy
192.168.15.174  pghaproxy2
```

---

## Etapa 2 — Desabilitar SELinux e Firewall

> **Executar em TODAS as máquinas.**

> ⚠️ **Nota:** Estas configurações são adequadas para ambiente de laboratório controlado. Em produção, prefira configurar regras específicas de SELinux e firewall.

### Desabilitar SELinux permanentemente e em tempo de execução

```bash
sed -i "s/SELINUX=enforcing/SELINUX=disabled/" /etc/selinux/config
setenforce 0
```

### Desabilitar o Firewalld e reiniciar

```bash
systemctl disable --now firewalld.service
init 6
```

> O sistema será reiniciado. Reconecte via SSH após a reinicialização antes de continuar.

---

## Etapa 3 — Repositório PGDG

> **Executar em TODAS as máquinas.**

Adicionar o repositório oficial do PostgreSQL Global Development Group para o RHEL 9:

```bash
sudo dnf install -y \
  https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

---

## Etapa 4 — Instalação do PostgreSQL 16

> **Escopo diferenciado — leia com atenção antes de executar.**

### Nos nós de banco de dados — `pgha1`, `pgha2`, `pgha3`

Instalar servidor **e** cliente:

```bash
dnf install -y postgresql16-server postgresql16
```

### Nos proxies — `pghaproxy`, `pghaproxy2`

Instalar **somente o cliente** (o motor de banco de dados ficará inativo nos proxies, que serão gerenciados pelo HAProxy):

```bash
dnf install -y postgresql16
```

---

## Etapa 5 — Symlinks e Estrutura de Diretórios

> **Executar em TODAS as máquinas** (ajusta PATH para uso do `psql` e demais ferramentas).

### Criar symlinks dos binários no PATH do sistema

```bash
ln -s /usr/pgsql-16/bin/* /usr/sbin/
which psql
```

### Criar e configurar diretório de dados customizado

> **Executar APENAS nos nós de banco de dados: `pgha1`, `pgha2`, `pgha3`.**
>
> O diretório `/usr/local/pgsql/data` será usado no lugar do padrão `/var/lib/pgsql/16/data`, facilitando gerenciamento e separação de volume em produção.

```bash
mkdir -p /usr/local/pgsql
chown postgres:postgres /usr/local/pgsql
chmod 0750 -R /usr/local/pgsql
ls -la /usr/local/pgsql/
```

### Redirecionar o diretório padrão do PostgreSQL via symlink

```bash
rm -rf /var/lib/pgsql/16/data
ln -s /usr/local/pgsql/data/ /var/lib/pgsql/16/
```

---

## Etapa 6 — Desabilitar o Serviço PostgreSQL

> **Executar APENAS nos nós de banco de dados: `pgha1`, `pgha2`, `pgha3`.**
>
> O Patroni será responsável por iniciar e gerenciar o PostgreSQL. Deixar o `postgresql-16.service` ativo causaria conflito com o Patroni.

```bash
systemctl disable --now postgresql-16.service
```

---

## Etapa 7 — Repositório e Instalação do TimescaleDB

> **Executar APENAS nos nós de banco de dados: `pgha1`, `pgha2`, `pgha3`.**

### Por que não instalar nos proxies?

O TimescaleDB é uma extensão do motor PostgreSQL, otimizada para séries temporais (métricas do Zabbix, por exemplo). Como os proxies não executam o motor de banco de dados — apenas roteiam conexões via HAProxy — instalar o TimescaleDB neles seria desperdício de espaço e violaria o princípio de servidores enxutos.

### Adicionar o repositório TimescaleDB

```bash
sudo tee /etc/yum.repos.d/timescale_timescaledb.repo <<EOL
[timescale_timescaledb]
name=timescale_timescaledb
baseurl=https://packagecloud.io/timescale/timescaledb/el/$(rpm -E %{rhel})/\$basearch
repo_gpgcheck=1
gpgcheck=0
enabled=1
gpgkey=https://packagecloud.io/timescale/timescaledb/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
EOL
```

### Instalar o TimescaleDB para PostgreSQL 16

```bash
dnf -y install timescaledb-2-postgresql-16
```

---

## Resumo de Execução por Host

| Etapa                              | pgha1 | pgha2 | pgha3 | pghaproxy | pghaproxy2 |
|------------------------------------|:-----:|:-----:|:-----:|:---------:|:----------:|
| 1 — /etc/hosts                     | ✅    | ✅    | ✅    | ✅        | ✅         |
| 2 — SELinux + Firewall             | ✅    | ✅    | ✅    | ✅        | ✅         |
| 3 — Repositório PGDG               | ✅    | ✅    | ✅    | ✅        | ✅         |
| 4 — postgresql16-server + client   | ✅    | ✅    | ✅    | ❌        | ❌         |
| 4 — postgresql16 (client apenas)   | ❌    | ❌    | ❌    | ✅        | ✅         |
| 5 — Symlinks /usr/sbin             | ✅    | ✅    | ✅    | ✅        | ✅         |
| 5 — Diretório /usr/local/pgsql     | ✅    | ✅    | ✅    | ❌        | ❌         |
| 6 — Disable postgresql-16.service  | ✅    | ✅    | ✅    | ❌        | ❌         |
| 7 — TimescaleDB                    | ✅    | ✅    | ✅    | ❌        | ❌         |

---

**Status:** Configuração inicial concluída. Próxima etapa: instalação e configuração do Patroni.
