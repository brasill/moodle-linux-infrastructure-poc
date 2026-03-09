
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

# 🎯 Objetivo do Projeto

Este projeto é uma Prova de Conceito (POC) de uma plataforma Moodle
adaptada para ensino de Português como segunda língua para pessoas surdas.

A infraestrutura foi construída manualmente com foco em:

• Arquitetura Linux minimalista  
• Hardening de segurança  
• Boas práticas de deploy  
• Documentação técnica  
• Escalabilidade futura  

O objetivo é validar a viabilidade da plataforma
para uso educacional e futura patente da metodologia pedagógica.

A metodologia educacional foi criada pelo pesquisador **Amauri** e utiliza
uma estrutura inovadora baseada em **blocos linguísticos** para ensino.

Essa metodologia está em processo de **registro e patenteamento** e,
portanto, **não é publicada neste repositório**.

Este projeto documenta **somente a infraestrutura técnica utilizada
para executar e testar a plataforma**.

---

# 📚 Contexto da Plataforma

A plataforma permitirá:

- criação de conteúdos educacionais
- testes com alunos voluntários
- experimentação de metodologias de ensino
- futura integração com avatares educacionais
- integração com dicionários e conteúdos multimídia

Inicialmente a infraestrutura será usada para:

- testes pedagógicos
- experimentos com alunos
- validação da metodologia
- evolução da plataforma

---

# 🧠 Competências Demonstradas

Este projeto demonstra experiência prática em:

• Administração Linux (Rocky Linux / RHEL-like)  
• Deploy de aplicações web  
• Configuração de Nginx  
• Integração PHP-FPM  
• Banco de dados PostgreSQL  
• Hardening com Fail2ban  
• Gerenciamento de firewall (Firewalld)  
• Segurança com SELinux  
• Virtualização com VirtualBox  
• Publicação segura usando Cloudflare Tunnel  
• Acesso administrativo seguro via Tailscale  
• Documentação técnica de troubleshooting  

---

# 🏗 Arquitetura da Infraestrutura

Stack utilizada:

Linux  
Nginx  
PostgreSQL  
PHP  

Arquitetura baseada em **LEPP Stack**

Componentes principais:

| Camada | Tecnologia |
|------|------|
| Sistema Operacional | Rocky Linux |
| Web Server | Nginx |
| Backend | PHP-FPM |
| Banco de Dados | PostgreSQL |
| Plataforma | Moodle |

---

# 🖥 Ambiente de Execução

Sistema operacional:

Rocky Linux 10 (Minimal Install)

Características:

- instalação mínima
- serviços reduzidos
- superfície de ataque menor
- foco em performance

---

# 🔐 Segurança da Infraestrutura

Camadas de segurança implementadas:

### Firewall

Gerenciado via **Firewalld**

Políticas:

- bloqueio de portas não utilizadas
- exposição mínima de serviços
- regras restritivas

---

### Proteção contra ataques

Ferramenta utilizada:

Fail2ban

Proteções:

- brute force SSH
- scanners automatizados
- bots maliciosos
- ataques ao Nginx

---

### Segurança do Sistema

Configurações aplicadas:

- SELinux ativo
- permissões restritas
- monitoramento de logs
- separação de serviços

---

# 🌐 Publicação da Plataforma

Para evitar exposição direta da infraestrutura durante os testes,
foi utilizada publicação segura via:

Cloudflare Tunnel

Benefícios:

- não expor IP público
- camada adicional de proteção
- mitigação básica de ataques
- controle de acesso externo

---

# 🔑 Acesso Seguro à Infraestrutura

Para acesso administrativo remoto foi utilizado:

Tailscale

Benefícios:

- VPN baseada em WireGuard
- acesso seguro entre dispositivos
- administração remota segura
- sem exposição de portas SSH públicas

---

# 💻 Virtualização

Ambiente inicial rodando em:

VirtualBox

Motivos:

- facilidade de replicação
- ambiente de testes isolado
- snapshots para rollback
- distribuição fácil para colaboradores

---

# ⚙ Deploy da Plataforma

Fluxo de deploy:

1. Instalação do Rocky Linux minimal
2. Configuração de firewall e SELinux
3. Instalação do Nginx
4. Instalação do PHP-FPM
5. Instalação do PostgreSQL
6. Deploy do Moodle
7. Configuração de segurança
8. Publicação via Cloudflare Tunnel

---

# 🔎 Monitoramento e Segurança Futura

Planejado para evolução da plataforma:

- análise de vulnerabilidades
- auditoria de segurança
- testes de hardening adicionais

Ferramentas em avaliação:

- Greenbone Vulnerability Manager
- scanners de segurança web

---

# 🔐 Autenticação (Planejamento)

Para minimizar armazenamento de dados sensíveis:

Planejado:

- autenticação SSO
- autenticação federada
- MFA

Objetivo:

reduzir armazenamento local de credenciais.

---

# 🚀 Evolução da Infraestrutura

Possível evolução futura:

- cloud infrastructure
- balanceamento de carga
- containers
- alta disponibilidade
- orquestração

---

# 📚 Referências Técnicas

Documentação oficial utilizada:

- https://docs.nginx.com
- https://moodle.org/documentation
- https://www.postgresql.org/docs
- https://access.redhat.com/documentation
- https://fail2ban.readthedocs.io
- https://tailscale.com/kb
- https://developers.cloudflare.com
- https://www.virtualbox.org/wiki/Documentation

---

# ⚠️ Nota sobre a Metodologia Educacional

Este repositório documenta **apenas a infraestrutura técnica**.

A metodologia pedagógica utilizada na plataforma está em processo de:

- registro
- patenteamento
- documentação científica

Detalhes pedagógicos **não são publicados neste repositório**.

---
