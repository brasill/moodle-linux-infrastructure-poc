# etcd Cluster — Alta Disponibilidade com TLS

> Guia completo de instalação, configuração e operação de um cluster etcd com 5 nós, TLS e quorum tolerante a falhas. Orientado a ambientes de laboratório e produção controlada.

---

## Índice

- [Topologia do Laboratório](#topologia-do-laboratório)
- [Pré-requisitos](#pré-requisitos)
- [Instalação](#instalação)
  - [Etapa 1 — Firewall](#etapa-1--firewall)
  - [Etapa 2 — Instalação do etcd](#etapa-2--instalação-do-etcd)
  - [Etapa 3 — Geração dos Certificados TLS](#etapa-3--geração-dos-certificados-tls)
  - [Etapa 4 — Distribuição dos Certificados](#etapa-4--distribuição-dos-certificados)
  - [Etapa 5 — Permissões e SELinux](#etapa-5--permissões-e-selinux)
  - [Etapa 5.1 — Configuração do etcd.conf](#etapa-51--configuração-do-etcdconf)
  - [Etapa 6 — Ativação do Serviço](#etapa-6--ativação-do-serviço)
- [Verificações de Saúde](#verificações-de-saúde)
- [Troubleshooting](#troubleshooting)
- [Boas Práticas SRE](#boas-práticas-sre)

---

## Topologia do Laboratório

| Host          | IP               | Nome etcd |
|---------------|------------------|-----------|
| `pgha1`       | 192.168.122.170  | etcd0     |
| `pgha2`       | 192.168.122.171  | etcd1     |
| `pgha3`       | 192.168.122.172  | etcd2     |
| `pgha-proxy`  | 192.168.122.173  | etcd3     |
| `pgha-proxy2` | 192.168.122.174  | etcd4     |

**Quorum:** mínimo de **3 de 5** nós disponíveis para operação normal.

---

## Pré-requisitos

- Rocky Linux 9 (ou RHEL 9 compatível)
- Repositório `pgdg-rhel9-extras` configurado
- Acesso SSH entre todos os nós (preferencialmente via chave)
- `openssl` disponível no nó `pgha-proxy`
- Usuário com privilégios `root` ou `sudo`

---

## Instalação

### Etapa 1 — Firewall

> **Executar em TODAS as máquinas.**

| Porta | Protocolo | Finalidade |
|-------|-----------|------------|
| 2379  | TCP       | Comunicação de clientes (Patroni, etcdctl) |
| 2380  | TCP       | Comunicação Peer-to-Peer (entre membros) |

```bash
firewall-cmd --permanent --add-port=2379/tcp
firewall-cmd --permanent --add-port=2380/tcp
firewall-cmd --reload
```

---

### Etapa 2 — Instalação do etcd

> **Executar em TODAS as máquinas.**

```bash
dnf --enablerepo=pgdg-rhel9-extras install -y etcd
```

---

### Etapa 3 — Geração dos Certificados TLS

> **Executar APENAS no `pgha-proxy`.**
>
> A CA é criada centralmente para evitar conflitos de assinatura no cluster. Os mesmos certificados são distribuídos para todos os nós.

```bash
mkdir ~/cert && cd ~/cert
```

#### 3.1 — Criar o arquivo de configuração OpenSSL

```bash
cat <<EOF > etcd_cert.cnf
[req]
default_bits  = 4096
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
countryName = BR
stateOrProvinceName = SP
localityName = Sao Paulo
organizationName = Lab
commonName = etcd-host

[v3_req]
keyUsage = digitalSignature, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names

[alt_names]
IP.1 = 127.0.0.1
IP.2 = 192.168.122.170
IP.3 = 192.168.122.171
IP.4 = 192.168.122.172
IP.5 = 192.168.122.173
IP.6 = 192.168.122.174
DNS.1 = localhost
EOF
```

> [!IMPORTANT]
> Qualquer alteração de IP ou DNS exige a geração de um **novo certificado** a partir do zero. O etcd rejeita certificados cujo SAN não bata com o IP do nó.

---

#### 3.2 — Gerar a CA (Autoridade Certificadora)

> [!NOTE]
> A CA é o "cartório" do cluster. Ela deve ser criada **antes** do certificado do etcd, pois o próximo passo depende de `ca_cert.pem` e `ca_key.pem` já existirem para assinar o CSR.

```bash
openssl req -x509 -noenc -newkey rsa:4096 \
  -subj '/CN=labCA' \
  -keyout ca_key.pem \
  -out ca_cert.pem \
  -days 36500
```

---

#### 3.3 — Gerar o certificado do etcd e assinar com a CA

```bash
# Gerar a chave privada e o CSR (Certificate Signing Request)
openssl req -noenc -newkey rsa:4096 \
  -keyout etcd_key.pem \
  -out etcd_cert.csr \
  -config etcd_cert.cnf

# Verificar o conteúdo do CSR antes de assinar
openssl req -in etcd_cert.csr -noout -text

# Assinar o CSR com a CA — gera o certificado final
openssl x509 -req -days 36500 \
  -in etcd_cert.csr \
  -CA ca_cert.pem \
  -CAkey ca_key.pem \
  -out etcd_cert.pem \
  -copy_extensions copy
```

---

#### 3.4 — Mover para o diretório oficial

```bash
sudo mkdir -p /etc/etcd/cert
sudo cp *.pem /etc/etcd/cert/
```

---

### Etapa 4 — Distribuição dos Certificados

> **Executar conforme indicado abaixo — leia antes de rodar.**

> [!WARNING]
> O Rocky Linux bloqueia cópias via SCP usando o usuário `root` por padrão. O processo abaixo abre uma **janela de manutenção temporária** para a cópia e a fecha logo em seguida. Não pule o passo de restauração da segurança.

**4.1 — Criar diretório e liberar acesso SSH temporário (executar em `pgha1`, `pgha2`, `pgha3` e `pgha-proxy2`):**

```bash
sudo mkdir -p /etc/etcd/cert

# Habilitar login de root temporariamente
sudo sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo sed -i 's/^PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

**4.2 — Enviar os certificados (executar APENAS no `pgha-proxy`):**

```bash
scp -rp /etc/etcd/cert/* root@pgha1:/etc/etcd/cert/
scp -rp /etc/etcd/cert/* root@pgha2:/etc/etcd/cert/
scp -rp /etc/etcd/cert/* root@pgha3:/etc/etcd/cert/
scp -rp /etc/etcd/cert/* root@pgha-proxy2:/etc/etcd/cert/
```

**4.3 — Restaurar a segurança SSH (executar em `pgha1`, `pgha2`, `pgha3` e `pgha-proxy2`):**

```bash
sudo sed -i 's/^PermitRootLogin yes/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

---

### Etapa 5 — Permissões e SELinux

> **Executar em TODAS as máquinas.**

```bash
chown -R etcd:etcd /etc/etcd/
chmod 0750 -R /etc/etcd/
restorecon -Rv /etc/etcd/
```

> [!NOTE]
> Esta etapa deve ser repetida sempre que novos arquivos forem copiados para `/etc/etcd/` (ex.: após redistribuição de certificados), pois o SELinux pode bloquear a leitura de arquivos com contexto incorreto.

---

### Etapa 5.1 — Configuração do etcd.conf

> **Executar em TODAS as máquinas**, alterando apenas os campos `ETCD_NAME` e os IPs correspondentes.

```bash
# Fazer backup da configuração padrão
mv /etc/etcd/etcd.conf /etc/etcd/etcd.conf.bkp

# Editar nova configuração
vi /etc/etcd/etcd.conf
```

**Template — substituir `etcd0` e `[IP DESSE HOST]` conforme a tabela de hosts:**

```bash
ETCD_NAME="etcd0"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"

ETCD_LISTEN_PEER_URLS="https://[IP DESSE HOST]:2380"
ETCD_LISTEN_CLIENT_URLS="https://localhost:2379,https://[IP DESSE HOST]:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://[IP DESSE HOST]:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://[IP DESSE HOST]:2379"

ETCD_INITIAL_CLUSTER="etcd0=https://192.168.122.170:2380,etcd1=https://192.168.122.171:2380,etcd2=https://192.168.122.172:2380,etcd3=https://192.168.122.173:2380,etcd4=https://192.168.122.174:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-lab"
ETCD_INITIAL_CLUSTER_STATE="new"

ETCD_CERT_FILE="/etc/etcd/cert/etcd_cert.pem"
ETCD_KEY_FILE="/etc/etcd/cert/etcd_key.pem"
ETCD_CLIENT_CERT_AUTH="false"
ETCD_TRUSTED_CA_FILE="/etc/etcd/cert/ca_cert.pem"

ETCD_PEER_CERT_FILE="/etc/etcd/cert/etcd_cert.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/cert/etcd_key.pem"
ETCD_PEER_CLIENT_CERT_AUTH="false"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/cert/ca_cert.pem"
```

> [!IMPORTANT]
> O bloco `ETCD_INITIAL_CLUSTER` deve ser **idêntico** em todos os nós. Qualquer divergência impede a formação do quorum.

---

### Etapa 6 — Ativação do Serviço

> **Executar em TODAS as máquinas.**

```bash
systemctl enable --now etcd.service
```

Após subir o serviço em todos os nós, reaplique as permissões para garantir que arquivos gerados em runtime tenham o owner correto:

```bash
chown -R etcd:etcd /etc/etcd/
chmod 0750 -R /etc/etcd/
restorecon -Rv /etc/etcd/
systemctl restart etcd
```

---

## Verificações de Saúde

### Verificar portas locais

```bash
ss -tunelp | grep 2379
ss -tunelp | grep 2380
```

---

### Health Check do Cluster

> Executar em **apenas um** dos hosts.

O comando abaixo é o "raio-X" da alta disponibilidade: verifica se todos os nós estão respondendo, sincronizados e com quorum estabelecido.

```bash
etcdctl \
  --endpoints=https://192.168.122.170:2379,https://192.168.122.171:2379,https://192.168.122.172:2379,https://192.168.122.173:2379,https://192.168.122.174:2379 \
  --cacert=/etc/etcd/cert/ca_cert.pem \
  --cert=/etc/etcd/cert/etcd_cert.pem \
  --key=/etc/etcd/cert/etcd_key.pem \
  endpoint health -w table
```

**Saída esperada com cluster saudável:**

```
+------------------------------+--------+-------------+-------+
|           ENDPOINT           | HEALTH |    TOOK     | ERROR |
+------------------------------+--------+-------------+-------+
| https://192.168.122.171:2379 |   true | 51.985314ms |       |
| https://192.168.122.172:2379 |   true | 58.501098ms |       |
| https://192.168.122.170:2379 |   true | 57.479207ms |       |
| https://192.168.122.173:2379 |   true | 58.637052ms |       |
| https://192.168.122.174:2379 |   true |  59.97284ms |       |
+------------------------------+--------+-------------+-------+
```

**Como interpretar:**

- `HEALTH: true` em todos os nós → cluster 100% operacional
- `HEALTH: false` ou timeout em algum nó → isolar a máquina e investigar com `journalctl -u etcd -f`

---

### Listar membros do cluster

```bash
etcdctl \
  --endpoints=https://192.168.122.170:2379,https://192.168.122.171:2379,https://192.168.122.172:2379,https://192.168.122.173:2379,https://192.168.122.174:2379 \
  --cacert=/etc/etcd/cert/ca_cert.pem \
  --cert=/etc/etcd/cert/etcd_cert.pem \
  --key=/etc/etcd/cert/etcd_key.pem \
  member list
```

---

### Verificar líder e status (tabela)

```bash
etcdctl \
  --endpoints=https://192.168.122.170:2379,https://192.168.122.171:2379,https://192.168.122.172:2379,https://192.168.122.173:2379,https://192.168.122.174:2379 \
  --cacert=/etc/etcd/cert/ca_cert.pem \
  --cert=/etc/etcd/cert/etcd_cert.pem \
  --key=/etc/etcd/cert/etcd_key.pem \
  endpoint status -w table
```

**Como interpretar a saída:**

- **`IS LEADER: true`** — somente um nó exibirá isso. Prova que o protocolo Raft elegeu um líder. Se esse nó cair, os demais elegem um novo automaticamente em milissegundos.
- **`RAFT INDEX` = `RAFT APPLIED INDEX`** em todos os nós — todos estão 100% sincronizados. Nenhum dado foi perdido.
- Comando executando com sucesso via certificados — comunicação criptografada e autenticada por mTLS está funcionando.

---

### Verificar saúde de cada nó

```bash
etcdctl \
  --endpoints=https://192.168.122.170:2379,https://192.168.122.171:2379,https://192.168.122.172:2379,https://192.168.122.173:2379,https://192.168.122.174:2379 \
  --cacert=/etc/etcd/cert/ca_cert.pem \
  --cert=/etc/etcd/cert/etcd_cert.pem \
  --key=/etc/etcd/cert/etcd_key.pem \
  endpoint health
```

---

## Troubleshooting

### Reingressar um nó no cluster (reset completo)

> [!IMPORTANT]
> Sempre limpe o diretório de dados antes de tentar subir um nó que falhou anteriormente. Dados residuais de uma sessão anterior impedem o etcd de ingressar no cluster como membro novo.

```bash
systemctl stop etcd
rm -rf /var/lib/etcd/default.etcd
systemctl start etcd
```

> [!TIP]
> Se ao colar comandos no terminal aparecerem caracteres estranhos como `^[[200~` no início, o terminal está em *bracketed paste mode*. Limpe o prompt com `Ctrl+C` e cole novamente.

### Monitorar logs em tempo real

```bash
journalctl -u etcd -f
```

---

## Boas Práticas SRE

- ✅ Validar conectividade de rede entre os nós **antes** de iniciar o troubleshooting
- ✅ **Nunca** reiniciar todos os nós simultaneamente — isso derruba o quorum
- ✅ Garantir quorum mínimo de **3 de 5** nós operacionais
- ✅ Monitorar logs com `journalctl` em caso de falha ou comportamento inesperado
- ✅ Guardar backups dos certificados da CA (`ca_cert.pem` e `ca_key.pem`) em local seguro e fora dos nós do cluster

---

## Estrutura de Certificados

```
/etc/etcd/cert/
├── ca_cert.pem       # Certificado da CA (público)
├── ca_key.pem        # Chave privada da CA (⚠️ proteger — quem tem isso assina qualquer cert)
├── etcd_cert.pem     # Certificado do nó etcd
└── etcd_key.pem      # Chave privada do nó etcd
```

---

**Status:** Documento revisado, consistente e operacionalmente seguro para uso em laboratório ou produção controlada.
