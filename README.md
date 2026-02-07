# Arquitetura de Alta Disponibilidade - E-commerce AWS

Este repositório mantém o mesmo diagrama em dois formatos:

- `challenge.eraserdiagram`: fonte em DSL (Eraser Diagram).
- `challenge.drawio`: versão convertida para XML (`mxfile/mxGraphModel`) compatível com diagrams.net/draw.io.

## Objetivo da arquitetura

A arquitetura foi desenhada para um e-commerce com foco em:

- Alta disponibilidade entre zonas de disponibilidade (AZs).
- Escalabilidade horizontal da camada de aplicacao.
- Seguranca de rede e controle de acesso por identidade.
- Observabilidade centralizada.
- Recuperacao de desastres com replicacao cross-region e backups.

## Componentes e papel de cada um

### 1) Borda de entrada e distribuicao de trafego

- `Usuarios`: origem do trafego HTTP/HTTPS.
- `Application Load Balancer (ALB)`: ponto unico de entrada na VPC.
  - Recebe requisicoes dos usuarios.
  - Distribui para instancias EC2 nas AZ A, B e C via target groups.
  - Remove dependencia de uma instancia especifica, reduzindo risco de indisponibilidade.

Por que existe:
Sem ALB, clientes dependeriam de IPs de instancias e teriamos acoplamento direto com a camada de computacao. Com ALB, o endpoint permanece estavel mesmo com escala in/out ou substituicao de instancias.

### 2) Camada de computacao (EC2 + Auto Scaling Group)

- `EC2 Linux A/B/C`: instancias da aplicacao em subnets privadas, uma por AZ.
- `Auto Scaling Group (Min: 3, Max: 6)`: garante:
  - Capacidade minima distribuida entre AZs.
  - Escala automatica sob aumento de carga.
  - Reposicao automatica de instancia com falha.

Por que existe:
Aplicacoes de e-commerce tem variacao forte de carga. O ASG evita superprovisionamento permanente e protege SLA em picos (campanhas, sazonalidade, eventos).

### 3) Camada de dados relacional (RDS Multi-AZ)

- `RDS Multi-AZ Primary (HA)` em `DB Subnet A`.
- `RDS Multi-AZ Standby (HA)` em `DB Subnet B`.
- `DB Subnet C` reservada dentro do `RDS Subnet Group`.
- `RDS Subnet Group (A/B/C)`: define onde o RDS pode operar com isolamento em subnets de banco.

Por que existe:
- Multi-AZ separa disponibilidade do banco entre zonas diferentes.
- Falhas de AZ ou instancia de banco nao derrubam o servico por completo; ocorre failover automatico.
- Subnets dedicadas de DB reduzem exposicao e melhoram controle de rota/seguranca.

### 4) Seguranca e identidade

- `Security Groups (Firewall)`:
  - ALB: entrada controlada (ex.: 443).
  - EC2: aceita trafego apenas do ALB.
  - RDS: aceita conexoes apenas das EC2 autorizadas.
- `IAM Roles (RDS IAM Auth)`:
  - Concede permissao para EC2 acessar o banco.
  - Permite autenticacao IAM no RDS (evita credenciais estaticas hardcoded).

Por que existe:
Aplica o principio de menor privilegio e reduz superficie de ataque. A combinacao IAM + Security Groups segmenta identidade e rede como controles complementares.

### 5) Observabilidade operacional

- `CloudWatch (Logs/Metricas/Alarmes)` integrado com ALB, EC2 e RDS.

Por que existe:
Sem telemetria centralizada, incidentes viram diagnosticos lentos. CloudWatch permite:

- Alertas proativos (latencia, erro, CPU, conexoes DB).
- Analise de comportamento em picos.
- Suporte a autoscaling baseado em metrica real.

### 6) Continuidade e recuperacao de desastres

- `Backups Automaticos`: retencao e restauracao pontual.
- `RDS Read Replica (Cross-Region DR)` em regiao secundaria.

Por que existe:
- Backup cobre corrupcao/logical errors e recuperacao temporal.
- Replica cross-region reduz risco de desastre regional.
- Estrategia combinada oferece camadas de resiliencia para RTO/RPO mais agressivos.

## Fluxo de ponta a ponta

1. Usuarios acessam o ALB.
2. ALB distribui requisicoes para EC2 A/B/C.
3. EC2 realizam operacoes de leitura/escrita no `RDS Primary`.
4. `RDS Primary` replica de forma sincronizada para `RDS Standby` (failover Multi-AZ).
5. `RDS Primary` replica de forma assíncrona para `Read Replica` na regiao secundaria.
6. Logs e metricas de ALB/EC2/RDS sao enviados ao CloudWatch.
7. Backups automaticos protegem dados para restauracao e DR.

## Decisoes arquiteturais e trade-offs

- Multi-AZ + ASG em 3 AZs:
  - Beneficio: alta disponibilidade e tolerancia a falha de zona.
  - Trade-off: custo maior de infraestrutura minima.
- RDS gerenciado:
  - Beneficio: reduz carga operacional de patching/failover.
  - Trade-off: menos flexibilidade que banco autogerenciado.
- IAM Auth no RDS:
  - Beneficio: melhora postura de seguranca de credenciais.
  - Trade-off: adiciona complexidade de integracao e expiracao de token.
- Read Replica cross-region:
  - Beneficio: aumenta resiliencia para desastre regional.
  - Trade-off: eventual consistencia para cenarios de leitura fora da regiao primaria.

## Conversao realizada para `challenge.drawio`

A conversao seguiu a estrutura XML nativa do diagrams.net:

- Documento raiz `mxfile`.
- Pagina com `diagram`.
- Modelo com `mxGraphModel`.
- Elementos em `mxCell`:
  - `vertex=\"1\"` para componentes visuais.
  - `edge=\"1\"` com `source` e `target` para relacoes.

Essa modelagem permite abrir e editar o diagrama diretamente no draw.io/diagrams.net sem perder os relacionamentos arquiteturais definidos no arquivo `.eraserdiagram`.

## Referencia utilizada (Context7)

- Draw.io docs (`/websites/drawio_doc`): estrutura `mxGraphModel`, edicao manual e formato XML.
