# Arquitetura de Alta Disponibilidade - E-commerce AWS

Este repositorio mantem a mesma arquitetura em dois formatos:

- `challenge.eraserdiagram`: fonte em DSL (Eraser Diagram).
- `challenge.drawio`: representacao XML (`mxfile/mxGraphModel`) para diagrams.net/draw.io.

## Observacao importante sobre equivalencia

Os dois arquivos representam o mesmo desenho logico, com uma diferenca de detalhamento:

- `challenge.eraserdiagram` prioriza a modelagem conceitual.
- `challenge.drawio` inclui ajustes visuais (roteamento manual de setas para legibilidade) e alguns rotulos tecnicos mais especificos (ex.: portas TCP em Security Groups e descricao explicita da DB Subnet C como reservada).

## Objetivo da arquitetura

Arquitetura para e-commerce com foco em:

- Alta disponibilidade entre AZs.
- Escalabilidade horizontal da camada de aplicacao.
- Seguranca de rede e controle de acesso por identidade.
- Observabilidade centralizada.
- Recuperacao de desastres com replicacao cross-region e backups.

## Componentes principais

### 1) Entrada e distribuicao de trafego

- `Usuarios`
- `Application Load Balancer` em `Public Subnets (A/B/C)`
- Distribuicao de trafego para:
  - `EC2 Linux VM A (ASG instances)`
  - `EC2 Linux VM B (ASG instances)`
  - `EC2 Linux VM C (ASG instances)`

### 2) Computacao e elasticidade

- `Auto Scaling Group (Min: 3, Max: 6, Linux)` provisionando as VMs nas 3 AZs.

### 3) Dados (PaaS/HA)

- `RDS Multi-AZ Primary (PaaS/HA)` em `DB Subnet A`
- `RDS Multi-AZ Standby (PaaS/HA)` em `DB Subnet B`
- `DB Subnet C` sem instancia ativa (subnet reservada no `RDS Subnet Group`)
- `RDS Subnet Group (A/B/C, Multi-AZ)`

### 4) Seguranca e identidade

- `IAM Roles (RDS IAM Auth)` para acesso das VMs ao banco.
- `Security Groups (Firewall)`:
  - ALB: `Inbound TCP 443` (origens autorizadas)
  - EC2: `Inbound TCP 80/443` somente do ALB
  - RDS: `Inbound TCP 3306/5432` somente das EC2

### 5) Observabilidade

- `CloudWatch (Logs/Metricas/Alarmes)` para ALB, EC2 e RDS.

### 6) DR e continuidade

- `Backups Automaticos (Retencao/DR)`
- `RDS Read Replica (Cross-Region DR)` em `Regiao Secundaria (VPC)`

## Fluxo resumido

1. Usuarios acessam o ALB.
2. ALB distribui requisicoes para EC2 em AZ A/B/C.
3. EC2 fazem leitura/escrita no `RDS Primary`.
4. `RDS Primary` replica de forma sincrona para `RDS Standby` (failover Multi-AZ).
5. `RDS Primary` replica de forma assincrona para `RDS Read Replica` (DR cross-region).
6. Logs e metricas seguem para CloudWatch.
7. Backups automaticos suportam restauracao e DR.

## Arquivos

- `challenge.eraserdiagram`: fonte editavel do desenho logico.
- `challenge.drawio`: arquivo final para abertura/edicao no draw.io.
