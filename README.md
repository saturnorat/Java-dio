# RELATÓRIO DE IMPLEMENTAÇÃO DE SERVIÇOS AWS

**Data:** 30/09/2025
**Empresa:** SimFarma
**Responsável:** Logan Maia

> Observação: setembro possui 30 dias; a data foi ajustada de 31/09/2025 para 30/09/2025.

## Introdução

Este relatório apresenta o processo de implementação de ferramentas na empresa **SimFarma**, realizado por **Logan Maia**. O objetivo do projeto foi **elencar 3 serviços AWS** para **suportar uma plataforma virtual de farmácia**, contemplando vitrine de produtos, carrinho de compras, upload e validação de receitas, processamento de pedidos e autenticação de usuários, com foco em **escalabilidade, segurança e otimização de custos**.

## Arquitetura (resumo textual)

* **Frontend** hospedado e versionado com *CI/CD*.
* **Autenticação** de clientes e time farmacêutico com contas e MFA.
* **Backend serverless** exposto via API REST.
* **Banco de dados NoSQL** para catálogo, estoque, carrinho e pedidos.
* **Armazenamento de objetos** para imagens de produtos e anexos de receitas (via módulo de armazenamento do frontend).
* **Observabilidade** com logs e métricas integrados (CloudWatch) e trilhas de auditoria (CloudTrail).

## Descrição do Projeto

O projeto foi dividido em **3 etapas**, cada uma associada a um serviço principal da AWS.

### Etapa 1

* **Nome da Ferramenta:** Amazon Amplify (Hosting & Auth)
* **Foco da Ferramenta:** Entrega do frontend com *CI/CD*, autenticação de usuários (Cognito via Amplify), e integração simplificada com armazenamento de arquivos (S3) e CDN (CloudFront) gerenciados pelo serviço.
* **Descrição de caso de uso:**

  * Hospedar o site React/Next.js da SimFarma com *builds* automáticos a cada *push* no repositório Git.
  * Ativar cadastro/login com e-mail e MFA; grupos para perfis **Cliente** e **Farmacêutico**.
  * Fluxo seguro de **upload de receitas**: o app usa o módulo *Amplify Storage* para colocar imagens/PDFs em um bucket dedicado, restringindo acesso com políticas por usuário e *links assinados*.
  * Entrega global via CDN, *custom domain* (ex.: `app.simfarma.com`) e *HTTPS* por padrão.

### Etapa 2

* **Nome da Ferramenta:** AWS Lambda (com Amazon API Gateway)
* **Foco da Ferramenta:** Backend *serverless* para regras de negócio e integração com dados; exposto por **API REST** com *throttling* e *authorizers*.
* **Descrição de caso de uso:**

  * Endpoints: `/produtos`, `/carrinho`, `/pedido`, `/receita/validar`.
  * Lógicas: cálculo de total e frete, verificação de estoque, criação de pedidos, associação de uploads de receita ao pedido, e validações de permissão (ex.: somente usuários do grupo **Farmacêutico** podem aprovar medicamentos controlados).
  * Integrações: leitura/escrita no DynamoDB; geração e validação de *pre-signed URLs* para anexos; logs e métricas no CloudWatch.
  * Segurança: *JWT authorizer* do Cognito (via Amplify) no API Gateway, limites de requisições e *CORS* configurado.

### Etapa 3

* **Nome da Ferramenta:** Amazon DynamoDB
* **Foco da Ferramenta:** Banco de dados NoSQL altamente escalável e de baixa latência para catálogo, carrinho e pedidos.
* **Descrição de caso de uso:**

  * **Modelagem de Tabela Única** com *partition key* (PK) e *sort key* (SK):

    * `PK`: `TENANT#SIMFARMA#<RECURSO>` (ex.: `TENANT#SIMFARMA#PROD#<ID>`)
    * `SK`: diferentes *entities* e estados (ex.: `PROD#<ID>`, `CART#<USER>#<TIMESTAMP>`, `ORDER#<ID>`)
  * **GSIs** para consultas por categoria, status do pedido e carrinhos por usuário.
  * **TTL** para carrinhos abandonados e *soft deletes*.
  * **Atributos de Compliance**: carimbo de data/hora de aprovação de receita e identificador do farmacêutico aprovador.

## Fluxos principais

1. **Cadastro/Login**: usuário cria conta → confirma e-mail → MFA → acessa vitrine. Perfis de acesso separados por grupo.
2. **Navegar e Comprar**: lista produtos (DynamoDB) → adiciona ao carrinho → Lambda recalcula totais → checkout.
3. **Receitas**: cliente anexa receita (upload seguro) → pedido fica “pendente de validação” → farmacêutico aprova via painel → pedido segue para faturamento/entrega.

## Boas práticas adotadas

* **Segurança**: IAM de menor privilégio; criptografia em repouso e em trânsito; segregação de ambientes (dev/homolog/produção) com contas/estágios distintos; *rotation* de segredos (AWS Secrets Manager) para integrações futuras.
* **Observabilidade**: logs estruturados no CloudWatch, métricas e alarmes (latência, erros 5xx, *throttling*), *dashboards* operacionais.
* **Custos**: arquitetura *pay-per-use* (Amplify, Lambda, DynamoDB sob demanda); *auto-scaling* nativo; remoção de *idle capacity*.

## Conclusão

A implementação dos serviços **Amazon Amplify**, **AWS Lambda (com API Gateway)** e **Amazon DynamoDB** fornece à SimFarma uma base **escalável, segura e econômica** para a sua plataforma virtual. Espera-se:

* **Disponibilidade e desempenho** consistentes para picos sazonais.
* **Segurança e compliance** com trilhas de auditoria e controle de acesso granular.
* **Agilidade de entrega** com *CI/CD* e desacoplamento entre frontend e backend.

Recomenda-se, como próximos passos, avaliar **AWS WAF** (proteção a nível de aplicação), **Amazon CloudFront** (CDN dedicada quando não usar Amplify Hosting), **Amazon S3 Glacier** (arquivamento de receitas expiradas), **Amazon SNS/SQS** (notificações e filas), e **Amazon Athena** sobre **S3** para relatórios analíticos.

## Anexos

### 1. Diagrama lógico da arquitetura

> Inserir aqui uma imagem (ex.: `docs/arquitetura.png`):

```markdown
![Arquitetura SimFarma](docs/arquitetura.png)
```

### 2. Especificação de endpoints (API Gateway)

| Método | Endpoint           | Descrição                            | Autorização        |
| ------ | ------------------ | ------------------------------------ | ------------------ |
| GET    | `/produtos`        | Lista produtos disponíveis           | Público            |
| GET    | `/carrinho`        | Retorna itens do carrinho do usuário | JWT (Cliente)      |
| POST   | `/carrinho`        | Adiciona item ao carrinho            | JWT (Cliente)      |
| POST   | `/pedido`          | Cria um pedido a partir do carrinho  | JWT (Cliente)      |
| POST   | `/receita/validar` | Valida uma receita anexada           | JWT (Farmacêutico) |

### 3. Modelo de tabela DynamoDB

| PK                          | SK                    | Atributos principais                   |
| --------------------------- | --------------------- | -------------------------------------- |
| `TENANT#SIMFARMA#PROD#123`  | `PROD#123`            | Nome, Preço, Categoria, Estoque        |
| `TENANT#SIMFARMA#USER#456`  | `CART#456#2025-09-30` | Itens, Quantidade, Valor total         |
| `TENANT#SIMFARMA#ORDER#789` | `ORDER#789`           | Status, Data criação, Valor, ReceitaID |

### 4. Fluxo de autenticação e perfis

* **Cliente:**

  * Cadastro/Login (Amplify + Cognito)
  * MFA obrigatório
  * Pode visualizar catálogo, adicionar ao carrinho, criar pedidos e enviar receitas
* **Farmacêutico:**

  * Autenticação via grupo especial no Cognito
  * Pode aprovar/reprovar pedidos com medicamentos controlados

### 5. Políticas de acesso (IAM)

* **Amplify Hosting:** permissões de *deploy* limitadas ao pipeline.
* **Lambda:** execução apenas com permissões para DynamoDB e S3 específicos.
* **DynamoDB:** acesso com chave de partição restrita ao tenant `SIMFARMA`.
* **S3:** políticas de bucket com *pre-signed URLs*, segregando acesso de clientes e farmacêuticos.
* **CloudWatch/CloudTrail:** somente leitura para time de auditoria.

---

**Logan Maia**
