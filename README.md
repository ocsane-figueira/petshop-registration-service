# 📋 Petshop Registration Service (`petshop-registration-service`)

Este repositório contém o **Microsserviço de Cadastro** da arquitetura distribuída do Petshop. Ele representa um dos lados de **Escrita (Command)** no padrão CQRS (Command Query Responsibility Segregation).

---

## 🏗️ Papel e Funcionalidade no Ecossistema

O `registration-service` gerencia os registros fundamentais do negócio:
1. **Domínio de Cadastro de Clientes**: Efetua a criação e validação de clientes (por exemplo, bloqueando CPFs duplicados na base de dados).
2. **Domínio de Cadastro de Animais**: Efetua o registro de animais de estimação associados logicamente aos clientes.
3. **Persistência Relacional**: Utiliza **PostgreSQL** para gravação de dados transacionais estruturados.
   * **Ambiente DEV**: Grava na base através do schema `dev`.
   * **Ambiente HOMOL/PROD**: Grava na base através do schema `homol`.
4. **Mensageria Assíncrona (Event Sourcing / CQRS Sync)**: Assim que uma gravação é confirmada com sucesso no banco de dados relacional, o microsserviço publica eventos de negócio reativos na fila do **RabbitMQ**:
   * Evento: `ClientCreatedEvent` (com os dados do cliente criado).
   * Evento: `AnimalCreatedEvent` (com os dados do animal criado).
   * Esses eventos serão consumidos pelo `query-service` para montar as views unificadas.

---

## 🛠️ Tecnologias Principais

* **Java 21** e **Quarkus Framework**
* **Hibernate ORM com Panache** (Simplificação de repositórios e entidades com Active Record)
* **JDBC Driver - PostgreSQL** (Conectividade relacional robusta)
* **SmallRye Reactive Messaging - RabbitMQ Connector** (Mensageria orientada a eventos assíncrona)
* **Quarkus Micrometer & Prometheus Registry** (Telemetria)

---

## 💻 Como Rodar o Serviço Localmente

### Pré-requisitos
* Java 21 JDK instalado localmente
* Maven instalado localmente (ou use o `./mvnw` incluso)
* Docker ativo para executar as bases de dados de suporte locais (consulte o repositório `petshop-infra`)

### Executando em Modo de Desenvolvimento (Live Coding)

Para iniciar o Quarkus localmente, conectado aos bancos rodando no Docker local:

```bash
./mvnw compile quarkus:dev
```

* **Porta local padrão**: `8082`
* **Painel Dev UI do Quarkus**: `http://localhost:8082/q/dev/`

---

## 🧪 Testes Automatizados e Ajustes de Configuração

O projeto possui suíte de testes unitários e de integração utilizando **JUnit 5**, **Mockito** e **RestAssured**:

### ⚠️ Observação Importante para Executar Testes Locais
Para rodar os testes fora do Docker de forma isolada usando os containers em memória (Testcontainers) do Quarkus:
1. Abra o arquivo `src/main/resources/application.properties`.
2. Comente a URL do banco principal (`quarkus.datasource.jdbc.url=...`) adicionando um caractere `#` na frente da propriedade.
3. Execute o comando de testes:
   ```bash
   ./mvnw clean verify
   ```
4. O Jacoco gerará o relatório visual em `target/jacoco-report/index.html` para auditar a cobertura do código (exigido mínimo de **50%** de cobertura).

---

## 🎛️ Observabilidade

O serviço expõe telemetria rica em tempo real para monitoramento corporativo:
* **Endpoint de Métricas**: `GET http://localhost:8082/q/metrics`
* Expõe latências de transações SQL, contadores de inserções, tempos de resposta HTTP e uso de conexões do pool do Postgres.
* **Integração**: Coletado pelo Prometheus e encaminhado ao Grafana Cloud via `remote_write`.

---

## 📖 Documentação da API (Swagger / OpenAPI)

O microsserviço está configurado com suporte nativo ao **Swagger UI** e geração de especificação **OpenAPI** via extensão `quarkus-smallrye-openapi`.

### 🌐 Endpoints de Acesso em Desenvolvimento (DEV)

Em ambiente de desenvolvimento (local ou na nuvem), você pode acessar a documentação diretamente no microsserviço (completamente independente do API Gateway):

* **Swagger UI (Interface Visual)**: `http://localhost:8081/q/swagger-ui/`
  * No Render (DEV): [https://petshop-registration-service-dev.onrender.com/q/swagger-ui/](https://petshop-registration-service-dev.onrender.com/q/swagger-ui/)
* **OpenAPI Spec (Esquema JSON)**: `http://localhost:8081/q/openapi`
  * No Render (DEV): [https://petshop-registration-service-dev.onrender.com/q/openapi](https://petshop-registration-service-dev.onrender.com/q/openapi)

### 🔒 Controle de Ambientes e Segurança

Para alinhar segurança e performance em produção/homologação, a exibição da documentação segue esta estratégia:

1. **Inclusão na Compilação (`Build Time`)**:
   A propriedade `quarkus.swagger-ui.always-include=true` está configurada no arquivo principal `application.properties`. Isso garante que o Quarkus compile e empacote os arquivos estáticos do Swagger no JAR de produção gerado no Dockerfile.
2. **Bloqueio em Homologação/Produção (`Runtime`)**:
   Para evitar a exposição pública indesejada de ferramentas de teste, o Swagger é desativado em tempo de execução no perfil de homologação através da propriedade:
   ```properties
   quarkus.swagger-ui.enable=false
   ```
   Qualquer tentativa de acesso fora do ambiente DEV retornará erro `404 Not Found`.

---

## 🚀 Pipeline de CI/CD (GitHub Actions)

Este repositório possui fluxos totalmente automatizados integrando as melhores práticas DevOps:

1. **Continuous Integration (`ci.yml`)**:
   * Executado a cada push/pull request para as branches `main` e `develop`.
   * Realiza a compilação e validação do código com Java 21.
   * Envia análises de qualidade estática para o **SonarCloud** (Project Key: `ocsane-figueira_petshop-registration-service`).
   * Para pushes aprovados em `main`, constrói a imagem Docker oficial multi-stage e envia para o Docker Hub com tags SHA e `main` (`ocsane/petshop-registration-service`).

2. **Automatic Release (`release.yml`)**:
   * Executado na branch `main` pós-CI bem-sucedido.
   * Utiliza **Semantic Release** para analisar commits convencionais e atualizar o SemVer no GitHub automaticamente.

3. **Continuous Deployment (`cd.yml`)**:
   * O fluxo monitora a conclusão do CI. Caso a validação de testes finalize com sucesso:
     * Branch `develop`: Invoca o webhook do Render para atualizar o ambiente de desenvolvimento (`petshop-registration-service-dev`).
     * Branch `main`: Invoca o webhook do Render para atualizar o ambiente de produção (`petshop-registration-service`).
