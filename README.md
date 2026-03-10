# Moodle Infrastructure POC – Linux Deployment

![Linux](https://img.shields.io/badge/Linux-Rocky%20Linux-blue?logo=linux)
![Nginx](https://img.shields.io/badge/Web%20Server-Nginx-green?logo=nginx)
![PHP](https://img.shields.io/badge/PHP-FPM-purple?logo=php)
![PostgreSQL](https://img.shields.io/badge/Database-PostgreSQL-blue?logo=postgresql)
![Security](https://img.shields.io/badge/Security-Hardening-red)
![Fail2ban](https://img.shields.io/badge/Protection-Fail2ban-orange)
![Firewall](https://img.shields.io/badge/Firewall-Firewalld-yellow)
![Cloudflare](https://img.shields.io/badge/Edge-Cloudflare-F38020?logo=cloudflare)
![Tailscale](https://img.shields.io/badge/VPN-Tailscale-242424?logo=tailscale)
![Virtualization](https://img.shields.io/badge/Virtualization-VirtualBox-blue?logo=virtualbox)

Infraestrutura técnica para execução de uma plataforma educacional baseada em Moodle, implantada em ambiente Linux com foco em **performance, segurança e escalabilidade**.

Este repositório documenta a **Prova de Conceito (POC)** da infraestrutura utilizada para hospedar a plataforma.

---

## 🎯 Objetivo do Projeto

Este projeto é uma Prova de Conceito (POC) de uma plataforma Moodle
adaptada para ensino de Português como segunda língua para pessoas surdas.

A infraestrutura foi construída manualmente com foco em:

- Arquitetura Linux minimalista
- Hardening de segurança
- Boas práticas de deploy
- Documentação técnica
- Escalabilidade futura

O objetivo é validar a viabilidade da plataforma para uso educacional e futura patente da metodologia pedagógica.

A metodologia educacional foi criada pelo pesquisador **Amauri Viana Fuzaro** e utiliza uma estrutura inovadora de ensino.

Essa metodologia está em processo de **registro e patenteamento** e, portanto, **não é publicada neste repositório**.

Este projeto documenta **somente a infraestrutura técnica utilizada para executar e testar a plataforma**.

---

## 📚 Contexto da Plataforma

A plataforma permitirá:

- Criação de conteúdos educacionais
- Testes com alunos voluntários
- Experimentação de metodologias de ensino
- Futura integração com avatares educacionais
- Integração com dicionários e conteúdos multimídia

Inicialmente a infraestrutura será usada para testes pedagógicos, experimentos com alunos, validação da metodologia e evolução da plataforma.

---

## 🧠 Competências Demonstradas

Este projeto demonstra experiência prática em:

- Administração Linux (Rocky Linux / RHEL-like)
- Deploy de aplicações web
- Configuração de Nginx
- Integração PHP-FPM
- Banco de dados PostgreSQL
- Hardening com Fail2ban
- Gerenciamento de firewall (Firewalld)
- Segurança com SELinux (enforcing — sem desativar)
- Virtualização com VirtualBox e VMware Workstation
- Publicação segura usando Cloudflare Tunnel
- Acesso administrativo seguro via Tailscale VPN
- Documentação técnica de troubleshooting

---

## 🏗 Arquitetura da Infraestrutura

Stack baseada em **LEPP (Linux · Nginx · PostgreSQL · PHP)**

| Camada | Tecnologia | Justificativa |
|---|---|---|
| Sistema Operacional | Rocky Linux 10 (Minimal) | Sucessor do CentOS, ciclo de vida longo, superfície de ataque reduzida |
| Web Server | Nginx | Arquitetura orientada a eventos, ideal para streaming de vídeos em Libras |
| Backend | PHP 8.3 via PHP-FPM | Requisito do Moodle 4.x/5.x; FPM isola o PHP do Nginx |
| Banco de Dados | PostgreSQL 16 | Maior integridade transacional; escolha padrão para aplicações críticas |
| Plataforma | Moodle 4.5 | LMS open source, suporte a multimídia e extensível |
| Virtualização | VirtualBox / VMware Workstation | Snapshots para rollback antes de alterações críticas |
| Acesso Admin | Tailscale (WireGuard) | SSH seguro sem exposição de portas públicas |
| Publicação | Cloudflare Tunnel | Exposição sem Port Forwarding; esconde o IP real; proteção DDoS |

---

## 🖥 Ambiente de Execução

Rocky Linux 10 (Minimal Install) em VirtualBox, host Windows 10.

- Instalação mínima — sem interface gráfica
- Serviços reduzidos ao necessário
- Superfície de ataque menor
- Foco em performance com RAM limitada (3 GB na VM)

---

## 💽 Particionamento

Esquema de particionamento aplicado no instalador do Rocky Linux 10:

| Partição | Tamanho | Justificativa |
|---|---|---|
| `/boot/efi` | 512 MiB | Boot seguro UEFI |
| `/boot` | 1 GiB | Separa o kernel por segurança |
| `/` (raiz) | 10 GiB | Sistema operacional base |
| `/var` | 15 GiB | Logs e dados do PostgreSQL isolados — evita que estouros de log travem o boot |
| `/var/www` | 11,5 GiB | Arquivos do Moodle e vídeos de Libras |
| `swap` | 2 GiB | Auxílio para os 3 GB de RAM da VM |

> 📌 Isolar `/var` impede que um log descontrolado consuma o espaço do sistema operacional e provoque falhas de boot.

---

## 🚀 Instalação — Passo a Passo

### 1. Primeiro Boot — Firewall e Atualização

Após o primeiro boot, libere o SSH e atualize o sistema:

```bash
# Libera o SSH no firewall permanentemente
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload

# Atualiza o sistema completo
sudo dnf update -y

# Ajusta o timezone
sudo timedatectl set-timezone America/Sao_Paulo
```

> 📌 **Conexão via PuTTY com NAT no VirtualBox:** use o endereço do hospedeiro (`localhost`) na porta configurada para SSH (não o IP interno da VM). O VirtualBox redireciona a porta do Windows para dentro da VM.

---

### 2. Snapshot Pós-SSH

Após validar o acesso remoto via SSH, tire um snapshot no VirtualBox:

**Nome sugerido:** `SSH_Configurado`

---

### 3. Repositórios EPEL e Remi

O PHP 8.3 exigido pelo Moodle não está no repositório padrão do Rocky Linux. O Remi é o padrão da indústria para RHEL/Rocky:

```bash
# Instala o EPEL e o repositório Remi
sudo dnf install -y epel-release
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-10.rpm

# Habilita especificamente o módulo PHP 8.3
sudo dnf module reset php
sudo dnf module enable php:remi-8.3 -y
```

---

### 4. Instalação da Stack (Nginx + PHP-FPM + PostgreSQL)

```bash
# Instala wget (necessário em instalação Minimal)
sudo dnf install -y wget

# Instala Nginx, PostgreSQL 16 e todas as extensões PHP do Moodle
sudo dnf install -y nginx postgresql-server postgresql-contrib \
  php-fpm php-intl php-soap php-gd php-pgsql php-xmlrpc php-mbstring \
  php-xml php-curl php-zip php-opcache php-json php-sodium
```

---

### 5. Configuração do PostgreSQL

O Rocky Linux 10 **não inicializa o cluster de dados automaticamente** após instalar o PostgreSQL. É obrigatório executar o setup antes de iniciar o serviço:

```bash
# Inicializa o cluster de dados (específico do RHEL/Rocky)
sudo postgresql-setup --initdb

# Habilita e inicia o serviço
sudo systemctl enable --now postgresql
```

**Criando o banco de dados e o usuário do Moodle:**

```bash
sudo -u postgres psql
```

Dentro do prompt `postgres=#`:

```sql
CREATE DATABASE moodle;
CREATE USER moodleuser WITH PASSWORD '<DATABASE_PASSWORD>';
GRANT ALL PRIVILEGES ON DATABASE moodle TO moodleuser;
GRANT ALL ON SCHEMA public TO moodleuser;
\q
```

> 📌 O `GRANT ALL ON SCHEMA public` é necessário a partir do PostgreSQL 15+, que protege o esquema `public` por padrão como medida de hardening. Sem esse comando, o Moodle falha ao criar suas tabelas com `ERRO: permissão negada para esquema public`.

---

### 6. Estrutura de Diretórios Segura

O Moodle exige dois locais separados: o código-fonte (acessível via web) e os dados (obrigatoriamente fora da raiz web por segurança):

```bash
# Diretório de dados — FORA da raiz web (nunca acessível por URL)
sudo mkdir -p /var/moodledata
sudo chown -R nginx:nginx /var/moodledata
sudo chmod -R 770 /var/moodledata

# Contexto SELinux para escrita (upload de aulas de Libras)
sudo chcon -R -t httpd_sys_rw_content_t /var/moodledata
```

---

### 7. Download e Deploy do Moodle

```bash
cd /var/www/html

# Baixa o Moodle 4.5 (versão estável)
sudo wget https://download.moodle.org/download.php/direct/stable405/moodle-latest-405.tgz

# Descompacta
sudo tar -zxvf moodle-latest-405.tgz

# Define o Nginx como dono
sudo chown -R nginx:nginx /var/www/html/moodle

# Contexto SELinux — permite leitura e escrita pelo servidor web
sudo chcon -R -t httpd_sys_rw_content_t /var/www/html/moodle
```

---

### 8. Configuração do SELinux (Princípio do Menor Privilégio)

Em vez de desativar o SELinux, liberamos apenas o necessário via booleans:

```bash
# Permite que o Nginx faça conexões de rede (PHP-FPM e PostgreSQL)
sudo setsebool -P httpd_can_network_connect 1

# Permite que o Nginx envie e-mails (notificações do Moodle)
sudo setsebool -P httpd_can_sendmail 1

# Libera HTTP e HTTPS no firewall
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

> 📌 Ajustar as booleans do SELinux em vez de desativá-lo demonstra a aplicação do **Princípio do Menor Privilégio** — diferencial relevante para vagas de Infraestrutura Crítica e SecOps.

---

### 9. Configuração do Nginx

Crie o arquivo de configuração dedicado ao Moodle:

```bash
sudo vi /etc/nginx/conf.d/moodle.conf
```

```nginx
server {
    listen 80;

    # Catch-all — aceita requisições do localhost, Tailscale ou Cloudflare
    server_name _;

    root /var/www/html/moodle;
    index index.php index.html;

    # Limite aumentado para suportar upload dos vídeos pesados em Libras
    client_max_body_size 512M;

    # Otimização de vídeo — streaming fluído de MP4
    location ~ \.mp4$ {
        mp4;
        mp4_buffer_size       1m;
        mp4_max_buffer_size   5m;
        expires max;
        add_header Cache-Control "public";
    }

    # Roteamento padrão do Moodle
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # Processamento do PHP-FPM
    location ~ [^/]\.php(/|$) {
        fastcgi_split_path_info  ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php-fpm/www.sock;
        fastcgi_index index.php;
        include fastcgi_params;

        # Correção Cloudflare — avisa o PHP que a conexão externa é HTTPS
        fastcgi_param HTTPS on;
        fastcgi_param HTTP_X_FORWARDED_PROTO https;

        # Correção NAT — força a porta para casar com o Port Forwarding
        fastcgi_param SERVER_PORT 8080;

        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

**Remova o server block padrão** que concorre com o do Moodle. Edite `/etc/nginx/nginx.conf` e apague o bloco `server { listen 80; ... }` presente nesse arquivo.

**Valide sempre antes de reiniciar:**

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

### 10. Configuração do PHP-FPM

No Rocky Linux, o PHP-FPM roda por padrão com o usuário `apache`. Como o servidor web é o Nginx, é necessário ajustar:

```bash
sudo sed -i 's/^user = apache/user = nginx/' /etc/php-fpm.d/www.conf
sudo sed -i 's/^group = apache/group = nginx/' /etc/php-fpm.d/www.conf

sudo systemctl enable --now php-fpm
sudo systemctl restart nginx
```

**Aumente os limites do PHP para suportar vídeos e múltiplos campos de formulário:**

```bash
echo "upload_max_filesize = 512M" | sudo tee -a /etc/php.ini
echo "post_max_size = 512M"       | sudo tee -a /etc/php.ini
echo "memory_limit = 512M"        | sudo tee -a /etc/php.ini
echo "max_input_vars = 5000"      | sudo tee -a /etc/php.ini

sudo systemctl restart php-fpm nginx
```

---

### 11. Port Forwarding no VirtualBox

Para acessar o Moodle pelo navegador do Windows com a VM em modo NAT:

Vá em **Configurações → Rede → Avançado → Redirecionamento de Portas** e adicione:

| Nome | Protocolo | IP Hospedeiro | Porta Hospedeiro | Porta Convidado |
|---|---|---|---|---|
| SSH | TCP | localhost | 2222 | 22 |
| HTTP | TCP | localhost | 8080 | 80 |

Após configurar, acesse no navegador do Windows pelo endereço `localhost` na porta `8080`.

---

### 12. Assistente de Instalação Visual do Moodle

Abra `localhost:8080` no navegador. O assistente pedirá:

| Campo | Valor |
|---|---|
| Idioma | Português - Brasil (pt_br) |
| Diretório Moodle | `/var/www/html/moodle` |
| Diretório de Dados | `/var/moodledata` ← remova o `/www/html` sugerido |
| Driver do Banco | PostgreSQL (pgsql) |
| Servidor do banco | `localhost` |
| Nome do banco | `moodle` |
| Usuário do banco | `moodleuser` |
| Senha | *(senha real — sem as aspas simples)* |
| Prefixo das tabelas | `mdl_` (padrão) |
| Porta | `5432` ou em branco |
| Unix socket | *(deixe em branco)* |

> 📌 O campo **Endereço Web** pode aparecer bloqueado pelo instalador. Se isso acontecer, prossiga — a correção da porta é feita manualmente no `config.php` após a instalação.

---

### 13. Criação Manual do `config.php`

O instalador pode não conseguir criar o `config.php` automaticamente (o Nginx não tem permissão de escrita na raiz do código-fonte — o que é correto do ponto de vista de segurança). Crie manualmente:

```bash
sudo vi /var/www/html/moodle/config.php
```

Pressione `i` para inserir, cole o conteúdo abaixo, depois `Esc` + `:wq` para salvar:

```php
<?php  // Moodle configuration file

unset($CFG);
global $CFG;
$CFG = new stdClass();

$CFG->dbtype    = 'pgsql';
$CFG->dblibrary = 'native';
$CFG->dbhost    = 'localhost';
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodleuser';
$CFG->dbpass    = '<DATABASE_PASSWORD>';
$CFG->prefix    = 'mdl_';
$CFG->dboptions = array (
  'dbpersist' => 0,
  'dbport' => 5432,
  'dbsocket' => '',
);

$CFG->wwwroot   = 'https://<SEU_DOMINIO_OU_TUNNEL>';
$CFG->dataroot  = '/var/moodledata';
$CFG->admin     = 'admin';

$CFG->directorypermissions = 02777;

require_once(__DIR__ . '/lib/setup.php');

// There is no php closing tag in this file,
// it is intentional because it prevents trailing whitespace problems!
```

---

## 🔐 Segurança da Infraestrutura

### Firewall

Gerenciado via **Firewalld** com políticas de exposição mínima:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

# Verificar regras ativas
sudo firewall-cmd --list-all
```

---

### Fail2ban — Proteção contra Intrusão

#### Instalação

```bash
sudo dnf install -y fail2ban fail2ban-firewalld ipset
sudo systemctl enable --now fail2ban
```

> 📌 Nunca edite o `jail.conf` original. Sempre crie um override em `jail.local`.

#### Configuração

```bash
sudo vi /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime   = 1h
findtime  = 10m
maxretry  = 5
backend   = systemd
banaction = firewallcmd-ipset
ignoreip  = 127.0.0.1/8 ::1

# Proteção SSH — bloqueia tentativas de acesso repetidas
[sshd]
enabled  = true
port     = ssh
logpath  = %(sshd_log)s
maxretry = 4
bantime  = 24h

# Ataques de autenticação no Nginx
[nginx-http-auth]
enabled  = true
logpath  = /var/log/nginx/error.log
maxretry = 3
backend  = auto

# Proteção contra scanners e requisições automatizadas
[nginx-botsearch]
enabled  = true
logpath  = /var/log/nginx/access.log
maxretry = 2
bantime  = 12h
backend  = auto

# Rate limit — limita requisições excessivas
[nginx-limit-req]
enabled  = true
logpath  = /var/log/nginx/error.log
maxretry = 5
backend  = auto
```

**Garanta que os arquivos de log existam** (em servidores recém-criados o Nginx pode não tê-los gerado ainda):

```bash
sudo touch /var/log/nginx/access.log
sudo touch /var/log/nginx/error.log
```

**Reinicie e verifique:**

```bash
sudo systemctl restart fail2ban

# Teste de sintaxe
sudo fail2ban-server -t

# Lista de jails ativas
sudo fail2ban-client status
```

Saída esperada:

```
Status
|- Number of jail:      4
`- Jail list:   nginx-botsearch, nginx-http-auth, nginx-limit-req, sshd
```

**Comandos úteis de operação:**

```bash
# Ver IPs banidos em uma jail específica
sudo fail2ban-client status sshd

# Desbanir um IP
sudo fail2ban-client set sshd unbanip IP_AQUI
```

---

### Hardening do SSH

```bash
sudo vi /etc/ssh/sshd_config
```

Ajuste as diretivas:

```
MaxAuthTries 3
PermitRootLogin no
PasswordAuthentication no
```

```bash
sudo systemctl restart sshd
```

---

### SELinux — Configurações Aplicadas

O SELinux permanece no modo **enforcing** durante toda a instalação. Nenhum `setenforce 0` foi executado. As permissões foram concedidas por booleans e contextos específicos:

| Configuração | Propósito |
|---|---|
| `httpd_can_network_connect 1` | Nginx conecta ao PHP-FPM e PostgreSQL |
| `httpd_can_sendmail 1` | Moodle envia notificações por e-mail |
| `httpd_sys_rw_content_t` em `/var/moodledata` | Nginx pode gravar uploads e dados |
| `httpd_sys_rw_content_t` em `/var/www/html/moodle` | Nginx pode gravar durante a instalação |

---

## 🌐 Publicação da Plataforma

### Cloudflare Tunnel

Expõe o Moodle para a internet sem abrir portas no roteador, sem expor o IP real e com proteção DDoS integrada da Cloudflare:

```bash
# Instala o cloudflared
sudo dnf install -y \
  https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-x86_64.rpm

# Verifica instalação
cloudflared --version

# Autentica com a conta Cloudflare (gera link para o browser)
cloudflared tunnel login
```

**Quick Tunnel — para testes sem domínio:**

```bash
cloudflared tunnel --url http://localhost:80
```

O cloudflared gera uma URL pública temporária do tipo `https://palavra-aleatoria.trycloudflare.com`.

**Após obter a URL do tunnel, atualize o `config.php`:**

```bash
# Atualiza o wwwroot com a URL do tunnel
sudo sed -i "s|http://127.0.0.1:8080|https://sua-url.trycloudflare.com|g" \
  /var/www/html/moodle/config.php

# Adiciona sslproxy=true — informa ao Moodle que está atrás de proxy HTTPS
sudo sed -i "s|\$CFG->wwwroot|\$CFG->sslproxy = true;\n\$CFG->wwwroot|" \
  /var/www/html/moodle/config.php

sudo systemctl restart php-fpm nginx
```

**Verifique o resultado:**

```bash
grep -E "sslproxy|wwwroot" /var/www/html/moodle/config.php
```

---

### Tailscale — Acesso Administrativo Seguro

O Tailscale cria uma VPN WireGuard privada — acesso SSH seguro sem expor portas públicas:

```bash
# Instala o Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Habilita e inicia
sudo systemctl enable --now tailscaled

# Autentica (gera link para autorização no browser)
sudo tailscale up
```

**Desativar expiração de chave para servidores permanentes:**
No painel em `https://login.tailscale.com`, localize a máquina `srv-rocky_moodle`, clique nos três pontinhos e selecione **Disable key expiry**.

**Tailscale Funnel — publicação via Tailscale sem Cloudflare:**

```bash
sudo tailscale funnel --bg 80
sudo tailscale funnel status
```

---

## 🛠 Troubleshooting

### Autenticação do PostgreSQL falha (`ident` method)

**Sintoma:** `FATAL: autenticação do tipo Ident falhou para o usuário "moodleuser"`

**Causa:** O Rocky Linux configura o PostgreSQL em modo `ident` por padrão — aceita conexões somente quando o usuário do SO coincide com o usuário do banco.

**Solução:**

```bash
sudo vi /var/lib/pgsql/data/pg_hba.conf
```

Encontre as linhas de conexões locais e altere `ident` para `md5`:

```
# IPv4 local connections:
host    all    all    127.0.0.1/32    md5
# IPv6 local connections:
host    all    all    ::1/128         md5
```

```bash
sudo systemctl restart postgresql
```

---

### Loop Infinito de Redirecionamento (`ERR_TOO_MANY_REDIRECTS`)

**Causa:** Combinação de dois fatores — a porta interna (80) difere da configurada no `config.php` (8080); e o PHP-FPM rodando como `apache` não consegue gravar sessões na pasta do `nginx`.

**Solução:**

```bash
# Ajusta PHP-FPM para rodar como nginx
sudo sed -i 's/^user = apache/user = nginx/' /etc/php-fpm.d/www.conf
sudo sed -i 's/^group = apache/group = nginx/' /etc/php-fpm.d/www.conf

sudo systemctl restart php-fpm nginx
```

Certifique-se também de que o bloco PHP no `moodle.conf` contém:

```nginx
fastcgi_param HTTPS on;
fastcgi_param HTTP_X_FORWARDED_PROTO https;
fastcgi_param SERVER_PORT 8080;
```

Limpe o cache do navegador ou abra uma **Janela Anônima** antes de testar novamente.

---

### Diretório `moodledata` não pode ser criado pelo instalador

**Causa:** O Moodle tenta criar a pasta em `/var` mas o servidor web não tem permissão — comportamento correto de segurança.

**Solução:** Crie manualmente com permissões temporárias:

```bash
sudo mkdir -p /var/moodledata
sudo chmod -R 777 /var/moodledata
sudo chown -R nginx:nginx /var/moodledata
sudo chcon -R -t httpd_sys_rw_content_t /var/moodledata
```

Após a instalação concluir, **reverta para permissões restritas (hardening)**:

```bash
sudo chmod -R 770 /var/moodledata
```

> 📌 A permissão `777` é temporária — usada porque o PHP-FPM pode rodar como `apache` durante a instalação. Revertê-la após o wizard é obrigatório.

---

### `permissão negada para esquema public` no PostgreSQL 15+

**Causa:** A partir do PostgreSQL 15, o esquema `public` é protegido por padrão. O `GRANT ALL PRIVILEGES ON DATABASE` não inclui permissão para criar tabelas.

**Solução:**

```bash
sudo -u postgres psql -d moodle
```

```sql
GRANT ALL ON SCHEMA public TO moodleuser;
\q
```

---

### Pacote de idioma `pt_br` falha ao baixar

**Causa:** Extensão `php-curl` ausente ou PHP-FPM não reiniciado após instalar.

**Solução automática:**

```bash
sudo dnf install -y php-curl
sudo systemctl restart php-fpm nginx
```

**Solução manual (se o download automático continuar falhando):**

```bash
sudo dnf install -y unzip
sudo mkdir -p /var/moodledata/lang
cd /var/moodledata/lang

sudo wget https://download.moodle.org/download.php/langpack/4.5/pt_br.zip
sudo unzip pt_br.zip
sudo chown -R nginx:nginx /var/moodledata/lang
```

---

### Server block padrão do Nginx sobrepõe o Moodle

**Causa:** O `nginx.conf` possui um server block na porta 80 com precedência sobre o `moodle.conf`.

**Solução:** Edite `/etc/nginx/nginx.conf` e remova o bloco:

```nginx
server {
    listen       80;
    listen       [::]:80;
    server_name  _;
    root         /usr/share/nginx/html;
    include /etc/nginx/default.d/*.conf;
}
```

```bash
sudo nginx -t && sudo systemctl restart nginx
```

---

### Moodle sem CSS após publicação com Cloudflare

**Causa:** O Moodle detecta que a porta interna difere da URL pública e entra em loop de redirecionamento.

**Solução:** Certifique-se de que o `config.php` possui:

```php
$CFG->sslproxy = true;
$CFG->wwwroot   = 'https://sua-url-publica.trycloudflare.com';
```

E que o bloco PHP no Nginx contém os parâmetros:

```nginx
fastcgi_param HTTPS on;
fastcgi_param HTTP_X_FORWARDED_PROTO https;
```

---

## 🖥 Importando para VMware Workstation

Ao migrar a VM do VirtualBox para o VMware, ajuste as configurações abaixo:

**1. Remova o arquivo de lock antes de ligar a VM:**
Delete a pasta `srv-rocky_moodle.vmx.lck` — ela é um cadeado de sessão anterior que impede o boot.

**2. Corrija o guestOS no `.vmx`:**

```ini
# De:
guestos = "other"

# Para:
guestos = "rhel9-64"
```

**3. Desative o conflito com Hyper-V:**
Em **Settings → Options → Advanced**, desmarque *Enable side channel mitigations for Hyper-V enabled hosts*.

**4. Corrija as VMnets no Virtual Network Editor:**

- Selecione **VMnet1** → marque **Host-only** → Apply
- Selecione **VMnet8** → marque **NAT** → Apply
- Em **NAT Settings**, adicione redirecionamento: `Host Port 8080 → VM Port 80`

---

## 🔎 Monitoramento e Segurança Futura

Planejado para evolução da plataforma:

- Análise de vulnerabilidades com Greenbone Vulnerability Manager
- Auditoria de segurança e testes de hardening adicionais
- Scanners de segurança web

---

## 🔐 Autenticação (Planejamento)

Para minimizar armazenamento de dados sensíveis:

- Autenticação SSO / Federada
- MFA
- Objetivo: reduzir armazenamento local de credenciais

---

## 🚀 Evolução da Infraestrutura

Possível evolução futura:

- Cloud infrastructure
- Balanceamento de carga
- Containers (Docker / Podman)
- Alta disponibilidade
- Orquestração

---

## 🔐 Security Hardening — Resumo

| Camada | Configuração |
|---|---|
| SELinux | Enforcing — sem desativar |
| Fail2ban | 4 jails ativas (sshd, nginx-http-auth, nginx-botsearch, nginx-limit-req) |
| FirewallD | Ativo — apenas HTTP / HTTPS / SSH liberados |
| Nginx | Configuração hardened com headers e limite de tamanho |
| Moodle data | Fora da raiz web (`/var/moodledata`) |
| SSH | `MaxAuthTries 3` · `PermitRootLogin no` · `PasswordAuthentication no` |
| PostgreSQL | Autenticação `md5` · schema `public` com permissões explícitas |

---

## ⚙ Deploy da Plataforma — Fluxo Resumido

```
 1. Instalação do Rocky Linux 10 Minimal com particionamento manual
 2. Primeiro boot: firewall, SSH, timezone, dnf update
 3. Snapshot: SSH_Configurado
 4. Repositórios EPEL + Remi → PHP 8.3
 5. Instalação: Nginx + PHP-FPM + PostgreSQL 16
 6. PostgreSQL: initdb + banco + usuário + GRANT SCHEMA public
 7. Estrutura de diretórios: /var/moodledata fora da raiz web
 8. Download e extração do Moodle 4.5
 9. SELinux: booleans + contextos httpd_sys_rw_content_t
10. Nginx: moodle.conf com otimização MP4 e parâmetros FastCGI
11. PHP-FPM: usuário nginx + limites de upload (512M)
12. Port Forwarding VirtualBox: 8080 → 80
13. Assistente visual: localhost:8080
14. config.php manual com wwwroot + sslproxy
15. Fail2ban: 4 jails (SSH + Nginx)
16. Hardening SSH: MaxAuthTries, PermitRootLogin no
17. Tailscale: VPN para acesso admin seguro
18. Cloudflare Tunnel: publicação sem exposição de IP
19. Snapshot final: MOODLE_LIBRAS_FINAL_OPT
```

---

## 📚 Referências Técnicas

| Recurso | Link |
|---|---|
| Moodle Documentation | https://moodle.org/documentation |
| Nginx Documentation | https://docs.nginx.com |
| PostgreSQL Docs | https://www.postgresql.org/docs |
| Rocky Linux / RHEL Docs | https://access.redhat.com/documentation |
| Fail2ban Docs | https://fail2ban.readthedocs.io |
| Tailscale KB | https://tailscale.com/kb |
| Cloudflare Tunnel Docs | https://developers.cloudflare.com |
| VirtualBox Documentation | https://www.virtualbox.org/wiki/Documentation |
| SELinux User's Guide | https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/using_selinux |
| PHP-FPM Configuration | https://www.php.net/manual/en/install.fpm.configuration.php |

---

## ⚠️ Nota sobre a Metodologia Educacional

Este repositório documenta **apenas a infraestrutura técnica**.

A metodologia pedagógica utilizada na plataforma está em processo de registro, patenteamento e documentação científica. Detalhes pedagógicos **não são publicados neste repositório**.
