# HAProxy + Keepalived — Proxy de Alta Disponibilidade com VIP Failover

> Este componente integra o stack de alta disponibilidade do repositório [`brasill/moodle-linux-infrastructure-poc`](https://github.com/brasill/moodle-linux-infrastructure-poc).
> Implementa balanceamento de carga para o cluster PostgreSQL/Patroni com failover automático de IP virtual (VIP) via protocolo VRRP.

---

## Índice

- [Visão Geral da Arquitetura](#visão-geral-da-arquitetura)
- [Topologia](#topologia)
- [Como Funciona o Failover](#como-funciona-o-failover)
- [Pré-requisitos](#pré-requisitos)
- [Etapa 1 — Instalação dos Pacotes](#etapa-1--instalação-dos-pacotes)
- [Etapa 2 — Configuração do Keepalived](#etapa-2--configuração-do-keepalived)
- [Etapa 3 — Configuração do HAProxy](#etapa-3--configuração-do-haproxy)
- [Etapa 4 — Ativação dos Serviços](#etapa-4--ativação-dos-serviços)
- [Verificação e Validação](#verificação-e-validação)
- [Troubleshooting](#troubleshooting)
- [Boas Práticas SRE](#boas-práticas-sre)
- [Referências Técnicas](#referências-técnicas)

---

## Visão Geral da Arquitetura

Este lab utiliza dois serviços complementares para garantir alta disponibilidade na camada de acesso ao banco de dados:

| Serviço      | Função                                                                 |
|--------------|------------------------------------------------------------------------|
| **Keepalived** | Gerencia o VIP (IP Virtual) entre os dois proxies via protocolo VRRP. Se o MASTER falhar, o BACKUP assume o VIP automaticamente em menos de 2 segundos. |
| **HAProxy**    | Recebe conexões no VIP e as distribui para os nós do cluster PostgreSQL/Patroni, separando tráfego de escrita (primário) e leitura (réplicas). |

```
Clientes / Aplicações
        │
        ▼
  192.168.122.180 (VIP — Keepalived)
        │
   ┌────┴────┐
   │         │
pghaproxy  pghaproxy2
(MASTER)   (BACKUP)
   │
   ▼ HAProxy
┌──────────────────────┐
│  Porta 5000 → PRIMARY │  (escrita — Patroni /master)
│  Porta 5001 → REPLICA │  (leitura — Patroni /replica)
│  Porta 7000 → Stats   │  (dashboard HTTP)
└──────────────────────┘
        │
   ┌────┼────┐
   ▼    ▼    ▼
 pgha1 pgha2 pgha3
 :5432 :5432 :5432
 (health check :8008 via Patroni REST API)
```

---

## Topologia

| Host          | IP               | Função VRRP | Prioridade |
|---------------|------------------|-------------|------------|
| `pghaproxy`   | 192.168.122.173  | MASTER      | 101        |
| `pghaproxy2`  | 192.168.122.174  | BACKUP      | 100        |
| VIP           | 192.168.122.180  | Virtual IP  | —          |

| Host    | IP               | Porta PostgreSQL | Porta Health Check |
|---------|------------------|------------------|--------------------|
| `pgha1` | 192.168.122.170  | 5432             | 8008 (Patroni)     |
| `pgha2` | 192.168.122.171  | 5432             | 8008 (Patroni)     |
| `pgha3` | 192.168.122.172  | 5432             | 8008 (Patroni)     |

---

## Como Funciona o Failover

### Camada 1 — VIP Failover (Keepalived / VRRP)

O Keepalived monitora a saúde do processo HAProxy a cada 2 segundos usando o script `killall -0 haproxy`. Se o HAProxy no MASTER parar de responder:

1. O MASTER perde pontos de prioridade e libera o VIP
2. O BACKUP eleva sua prioridade e anuncia o VIP via VRRP
3. O IP `192.168.122.180` passa a responder no `pghaproxy2`

O failover ocorre em **menos de 4 segundos** (2s de intervalo de check + 2s de propagação VRRP).

### Camada 2 — Roteamento de Conexões (HAProxy / Patroni REST API)

O HAProxy não se conecta diretamente na porta 5432 para descobrir quem é o primário. Ele usa a **API REST do Patroni** na porta `8008` de cada nó:

| Endpoint Patroni  | Retorno HTTP | Significado              |
|-------------------|--------------|--------------------------|
| `OPTIONS /master` | `200 OK`     | Este nó é o PRIMARY      |
| `OPTIONS /replica`| `200 OK`     | Este nó é uma REPLICA    |
| Qualquer outro    | `503`        | Nó indisponível ou em transição |

Isso garante que o HAProxy sempre encaminhe escritas apenas para o nó que o Patroni elegeu como líder — sem risco de split-brain na camada de proxy.

---

## Pré-requisitos

- Rocky Linux 9 (ou RHEL 9 compatível) em `pghaproxy` e `pghaproxy2`
- Interface de rede: `enp1s0` (ajuste se diferente — verifique com `ip link show`)
- PostgreSQL 16 client instalado (sem o server — ver README de configuração inicial)
- Cluster etcd + Patroni já operacional nos nós `pgha1`, `pgha2`, `pgha3`
- Firewall desabilitado ou regras específicas configuradas para as portas abaixo

| Porta | Serviço    | Finalidade                          |
|-------|------------|-------------------------------------|
| 5000  | HAProxy    | Conexões de escrita (PRIMARY)       |
| 5001  | HAProxy    | Conexões de leitura (REPLICA)       |
| 7000  | HAProxy    | Dashboard de estatísticas (HTTP)    |
| 112   | Keepalived | Protocolo VRRP (multicast/unicast)  |

---

## Etapa 1 — Instalação dos Pacotes

> **Executar em `pghaproxy` e `pghaproxy2`.**

```bash
dnf -y install epel-release
dnf install -y keepalived haproxy
```

---

## Etapa 2 — Configuração do Keepalived

> **Executar separadamente em cada proxy — as configurações diferem entre MASTER e BACKUP.**

```bash
cd /etc/keepalived
mv keepalived.conf keepalived.conf.bkp
```

---

### 2.1 — Configuração do MASTER (`pghaproxy` — 192.168.122.173)

```bash
vim /etc/keepalived/keepalived.conf
```

```nginx
global_defs {
}

vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    interface enp1s0
    state MASTER
    priority 101
    virtual_router_id 51

    authentication {
        auth_type PASS
        auth_pass Xtr54sdD
    }

    virtual_ipaddress {
        192.168.122.180
    }

    unicast_src_ip 192.168.122.173
    unicast_peer {
        192.168.122.174
    }

    track_script {
        chk_haproxy
    }
}
```

---

### 2.2 — Configuração do BACKUP (`pghaproxy2` — 192.168.122.174)

```bash
vim /etc/keepalived/keepalived.conf
```

```nginx
global_defs {
}

vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    interface enp1s0
    state BACKUP
    priority 100
    virtual_router_id 51

    authentication {
        auth_type PASS
        auth_pass Xtr54sdD
    }

    virtual_ipaddress {
        192.168.122.180
    }

    unicast_src_ip 192.168.122.174
    unicast_peer {
        192.168.122.173
    }

    track_script {
        chk_haproxy
    }
}
```

> [!IMPORTANT]
> Os campos que **obrigatoriamente diferem** entre MASTER e BACKUP são: `state`, `priority`, `unicast_src_ip` e o IP dentro de `unicast_peer`. Todos os outros blocos são idênticos.

> [!TIP]
> Antes de tentar subir o serviço, valide a sintaxe do arquivo com o próprio binário do Keepalived — ele aponta a linha exata do erro:
> ```bash
> keepalived -t -f /etc/keepalived/keepalived.conf
> ```

---

## Etapa 3 — Configuração do HAProxy

> **Executar em `pghaproxy` e `pghaproxy2` — a configuração é idêntica nos dois.**

### 3.1 — Ajustar o arquivo de unit do systemd

```bash
cd /usr/lib/systemd/system/
vi haproxy.service
```

Substituir o conteúdo pelo template abaixo, que habilita recarregamento sem queda de conexões (`-USR2`):

```ini
[Unit]
Description=HAProxy Load Balancer
Documentation=man:haproxy(1)
StartLimitInterval=0
StartLimitBurst=0
After=network.target syslog.service
Wants=syslog.service

[Service]
Type=forking
Environment="CONFIG=/etc/haproxy/haproxy.cfg" "PIDFILE=/run/haproxy.pid"
ExecStartPre=/usr/sbin/haproxy -f $CONFIG -c -q
ExecStart=/usr/sbin/haproxy -f $CONFIG -p $PIDFILE
ExecReload=/usr/sbin/haproxy -f $CONFIG -c -q
ExecReload=/bin/kill -USR2 $MAINPID
KillMode=mixed
Restart=on-failure
RestartSec=5
```

### 3.2 — Criar a configuração do HAProxy

```bash
mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bkp
vim /etc/haproxy/haproxy.cfg
```

```haproxy
global
    maxconn 100

defaults
    log global
    mode tcp
    retries 2
    timeout client  30m
    timeout connect 4s
    timeout server  30m
    timeout check   5s

# Dashboard de estatísticas — acessível via http://192.168.122.180:7000
listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

# Tráfego de ESCRITA — encaminha apenas para o nó PRIMARY (Patroni /master → HTTP 200)
listen production
    bind 192.168.122.180:5000
    option httpchk OPTIONS /master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pgha1 192.168.122.170:5432 maxconn 100 check port 8008
    server pgha2 192.168.122.171:5432 maxconn 100 check port 8008
    server pgha3 192.168.122.172:5432 maxconn 100 check port 8008

# Tráfego de LEITURA — distribui entre réplicas ativas (Patroni /replica → HTTP 200)
listen standby
    bind 192.168.122.180:5001
    option httpchk OPTIONS /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pgha1 192.168.122.170:5432 maxconn 100 check port 8008
    server pgha2 192.168.122.171:5432 maxconn 100 check port 8008
    server pgha3 192.168.122.172:5432 maxconn 100 check port 8008
```

> [!NOTE]
> O parâmetro `on-marked-down shutdown-sessions` encerra conexões existentes com um nó assim que ele for marcado como DOWN pelo health check. Isso evita que aplicações fiquem presas em conexões obsoletas após um failover do Patroni.

---

## Etapa 4 — Ativação dos Serviços

> **Executar em `pghaproxy` e `pghaproxy2`.**

```bash
# Recarregar o systemd para ler o haproxy.service modificado
systemctl daemon-reload

# Subir e habilitar ambos os serviços
systemctl enable --now keepalived
systemctl enable --now haproxy

# Verificar status
systemctl status keepalived
systemctl status haproxy
```

---

## Verificação e Validação

### Confirmar que o VIP está ativo no MASTER

```bash
# Executar no pghaproxy — deve listar 192.168.122.180
ip addr show enp1s0
```

### Verificar portas em escuta

```bash
ss -tunelp | grep -E '5000|5001|7000'
```

### Acessar o dashboard do HAProxy

Abrir no navegador: `http://192.168.122.180:7000`

O dashboard exibe em tempo real o status de cada servidor backend (`pgha1`, `pgha2`, `pgha3`), incluindo qual está UP/DOWN e o número de conexões ativas.

### Testar o failover do VIP manualmente

```bash
# No pghaproxy (MASTER): parar o HAProxy e observar o VIP migrar
systemctl stop haproxy

# No pghaproxy2: confirmar que o VIP foi assumido
ip addr show enp1s0

# Restaurar
systemctl start haproxy
```

### Verificar logs do Keepalived

```bash
journalctl -e | grep Keepalived
journalctl -u keepalived -f
```

---

## Troubleshooting

### VIP não aparece em nenhum dos proxies

Verificar se o processo do Keepalived está rodando e se a interface de rede configurada (`enp1s0`) está correta:

```bash
ip link show
systemctl status keepalived
journalctl -u keepalived --no-pager | tail -30
```

### HAProxy não sobe — erro de sintaxe no .cfg

```bash
haproxy -c -f /etc/haproxy/haproxy.cfg
```

### Nenhum backend marcado como UP no HAProxy

O health check usa a API REST do Patroni na porta `8008`. Verificar se o Patroni está respondendo:

```bash
curl -s http://192.168.122.170:8008/master
curl -s http://192.168.122.171:8008/replica
```

Se retornar `503`, o Patroni ainda não elegeu um líder ou o nó está em transição.

### Logs em tempo real

```bash
journalctl -u haproxy -f
journalctl -u keepalived -f
```

---

## Boas Práticas SRE

- ✅ **Nunca pare o HAProxy no MASTER sem antes garantir que o BACKUP está saudável** — o VIP só migra se o Keepalived no BACKUP detectar a falha
- ✅ **Mantenha `virtual_router_id` idêntico** nos dois proxies — IDs diferentes impedem o pareamento VRRP
- ✅ **Mantenha `auth_pass` idêntico** nos dois proxies — autenticação divergente faz os nós se ignorarem silenciosamente
- ✅ **Não exponha a porta 7000 do dashboard** em redes públicas — ele não tem autenticação por padrão
- ✅ **Monitore o health check do Patroni** (porta 8008) separadamente do status do PostgreSQL — é o que o HAProxy usa para rotear conexões
- ✅ Após qualquer alteração no `haproxy.cfg`, valide antes de recarregar: `haproxy -c -f /etc/haproxy/haproxy.cfg`

---

## Referências Técnicas

| Recurso | Link |
|---------|------|
| Documentação oficial do Keepalived | [keepalived.readthedocs.io](https://keepalived.readthedocs.io/en/latest/) |
| Keepalived — Configuração VRRP e scripts | [keepalived.readthedocs.io/en/latest/configuration_synopsis.html](https://keepalived.readthedocs.io/en/latest/configuration_synopsis.html) |
| Documentação oficial do HAProxy | [docs.haproxy.org](https://docs.haproxy.org/) |
| HAProxy — Guia de Configuração (versão estável) | [cbonte.github.io/haproxy-dconv](https://cbonte.github.io/haproxy-dconv/2.8/configuration.html) |
| Patroni REST API (health checks) | [patroni.readthedocs.io/en/latest/rest_api.html](https://patroni.readthedocs.io/en/latest/rest_api.html) |
| RFC 5798 — VRRP v3 (protocolo base do Keepalived) | [datatracker.ietf.org/doc/html/rfc5798](https://datatracker.ietf.org/doc/html/rfc5798) |

---

**Status:** Documento revisado, consistente e operacionalmente seguro para uso em laboratório ou produção controlada.
Parte integrante do projeto [`brasill/moodle-linux-infrastructure-poc`](https://github.com/brasill/moodle-linux-infrastructure-poc).
