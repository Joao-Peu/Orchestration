# Orchestration

Este repositório orquestra os serviços da aplicação (RabbitMQ, SQL Server para múltiplos domínios e APIs) usando Docker Compose e descreve um fluxo de deploy em Kubernetes.

Pré-requisitos
- Docker e Docker Compose instalados
- Kubernetes (kubectl configurado) e um cluster disponível (Minikube, kind ou AKS/EKS/GKE)
- Imagens das APIs construídas localmente ou via registry

Execução com Docker
1) Configure variáveis sensíveis
- As senhas e chaves em `docker-compose.yml` estão com valores de exemplo. Altere `MSSQL_SA_PASSWORD` e `Jwt__Key`.

2) Build das imagens locais
- Certifique-se que os diretórios das APIs referenciados em `docker-compose.yml` existem:
  - `../UsersAPI/src/UsersAPI`
  - `../CatalogAPI/CatalogAPI`
  - `../PaymentsAPI/PaymentsAPI`
  - `../NotificationsAPI/NotificationsAPI`
- O Compose já executa o build nas seções `build`. Se preferir, faça o build manualmente nos projetos.

3) Subir o ambiente
- Na raiz do repositório, execute:
  - `docker compose up -d --build`
- Serviços expostos:
  - RabbitMQ Management UI: http://localhost:15672 (user: `fiap`, pass: `fiap123`)
  - Users API: http://localhost:5001
  - Catalog API: http://localhost:5002
  - Payments API: http://localhost:5003
  - Notifications API: http://localhost:5004
- Bancos SQL Server (acessíveis via host local):
  - `localhost:14331` (sql-users)
  - `localhost:14332` (sql-catalog)
  - `localhost:14333` (sql-payments)

4) Logs e troubleshooting
- `docker compose ps`
- `docker compose logs -f <serviço>`
- Se as APIs não iniciarem por dependência de banco, verifique se as bases foram criadas/migradas. Ajuste as `ConnectionStrings__*` conforme necessário.

Encerrando o ambiente Docker
- `docker compose down` (mantém volumes)
- `docker compose down -v` (remove volumes e dados)

Deploy em Kubernetes (guia básico)
Observação: Este guia assume que você criará manifests para cada serviço (Deployments, Services, Secrets e ConfigMaps). Adapte nomes e namespaces conforme seu cluster.

1) Preparar imagens em um registry
- Suba as imagens das APIs para um registry acessível pelo cluster (Docker Hub, ACR, ECR, etc.). Exemplo:
  - `docker build -t <registry>/users-api:latest ../UsersAPI/src/UsersAPI`
  - `docker push <registry>/users-api:latest`
  - Repita para `catalog-api`, `payments-api`, `notifications-api`.
- Para RabbitMQ e SQL Server use as imagens oficiais:
  - `rabbitmq:3-management`
  - `mcr.microsoft.com/mssql/server:2019-latest`

2) Criar namespace
- `kubectl create namespace orchestration`

3) Secrets e ConfigMaps
- Crie `Secret` para senhas/keys (ex.: `sa` e `Jwt__Key`):
  - `kubectl -n orchestration create secret generic app-secrets --from-literal=MSSQL_SA_PASSWORD=StrongPassword!123 --from-literal=Jwt__Key="<sua-chave>" --from-literal=RabbitMQ_User=fiap --from-literal=RabbitMQ_Pass=fiap123`
- Crie `ConfigMap` para configurações não sensíveis (hosts, connection strings base) ou injete diretamente nas `Deployments`.

4) Manifests de infraestrutura
- RabbitMQ `Deployment` + `Service` (ClusterIP) + `PersistentVolumeClaim`.
- SQL Server por domínio (users, catalog, payments): para cada instância, `StatefulSet` + `Service` + `PVC`.
- Exemplo de `Service` para RabbitMQ (ClusterIP):
  - Porta 5672 para AMQP, 15672 para UI (se expor a UI, use `NodePort` ou `Ingress`).

5) Deploy das APIs
- Crie `Deployment` e `Service` para cada API.
- Injete variáveis via `env` lendo de `Secret` e `ConfigMap`:
  - `RabbitMQ__HostName=rabbitmq` (service name no cluster)
  - `ConnectionStrings__UserDb=Server=sql-users-svc,1433;Database=UsersDb;User Id=sa;Password=$(MSSQL_SA_PASSWORD);TrustServerCertificate=True;`
  - Ajuste para `CatalogDb` e `PaymentsDb` com os respectivos services.
- Exponha externamente com `Ingress` ou `LoadBalancer` conforme o ambiente.

6) Aplicar manifests
- Estruture uma pasta `k8s/` com arquivos yaml por serviço. Exemplo:
  - `k8s/namespace.yaml`
  - `k8s/rabbitmq.yaml`
  - `k8s/sql-users.yaml`, `k8s/sql-catalog.yaml`, `k8s/sql-payments.yaml`
  - `k8s/users-api.yaml`, `k8s/catalog-api.yaml`, `k8s/payments-api.yaml`, `k8s/notifications-api.yaml`
  - `k8s/secrets.yaml`, `k8s/configmap.yaml`, `k8s/ingress.yaml`
- Aplicar:
  - `kubectl apply -n orchestration -f k8s/`

7) Verificação
- `kubectl -n orchestration get pods,svc`
- `kubectl -n orchestration logs -f deploy/users-api`

Notas e boas práticas
- Use `StatefulSet` para SQL Server e `PersistentVolume` para dados.
- Configure `Readiness`/`Liveness` probes nas APIs e bancos.
- Utilize `Ingress` com TLS para expor APIs.
- Parametrize com `Helm` para ambientes (dev/staging/prod).
- Proteja segredos e rotacione senhas periodicamente.

Licença
- Consulte a licença do repositório original e das imagens utilizadas.
