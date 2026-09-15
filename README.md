# ToggleMaster — GitOps Repository

> Repositório de configuração declarativa (GitOps) do projeto **ToggleMaster**, gerenciado pelo **ArgoCD**.
> Este repositório **não contém código de aplicação** — apenas os manifestos Kubernetes (YAML) que descrevem como cada microsserviço deve rodar no cluster EKS.

---

## 📖 Sobre este Repositório

Este é o repositório **GitOps** do ToggleMaster, uma plataforma de *feature flags* baseada em microsserviços. Ele segue o princípio de **separação de responsabilidades**:

| Repositório | Responsabilidade |
| :--- | :--- |
| [`togglemaster-phase3`](https://github.com/luisfelipems/togglemaster-phase3) | Código-fonte das aplicações, Dockerfiles, pipelines CI/CD e Infraestrutura (Terraform) |
| **`togglemaster-gitops`** (este) | Estado desejado do cluster — manifestos Kubernetes sincronizados pelo ArgoCD |

O **ArgoCD** monitora continuamente este repositório e garante que o estado do cluster Kubernetes seja **exatamente igual** ao que está declarado aqui (*single source of truth*).

---

## 🔄 Como Funciona o Fluxo GitOps

```
┌─────────────────────┐     git push      ┌──────────────────────┐
│  Pipeline CI/CD     │  ───────────────► │  togglemaster-gitops │
│ (togglemaster-      │  atualiza a tag   │   (este repositório) │
│      phase3)        │   da imagem       └──────────┬───────────┘
└─────────────────────┘                              │
                                                     │ monitora (poll)
                                                     ▼
                                          ┌──────────────────────┐
                                          │       ArgoCD         │
                                          │  (no cluster EKS)    │
                                          └──────────┬───────────┘
                                                     │ sync automático
                                                     ▼
                                          ┌──────────────────────┐
                                          │   Cluster Kubernetes │
                                          │  (namespace:         │
                                          │    togglemaster)     │
                                          └──────────────────────┘
```

1. O desenvolvedor faz *push* de código no repositório `togglemaster-phase3`.
2. O pipeline CI/CD builda a imagem Docker, roda os *security gates* e faz *push* para o **Amazon ECR**.
3. Na etapa final (`update-gitops`), o pipeline (via `github-actions[bot]`) atualiza a **tag da imagem** no `deployment.yaml` correspondente **neste repositório** (commit com `[skip ci]`).
4. O **ArgoCD** detecta a mudança e sincroniza automaticamente o cluster, recriando os pods com a nova imagem.

---

## 📁 Estrutura do Repositório

```
togglemaster-gitops/
├── apps/                          # Manifestos Kubernetes de cada microsserviço
│   ├── analytics/
│   │   ├── deployment.yaml        # Deployment (a tag da imagem é atualizada pelo CI/CD)
│   │   └── service.yaml           # Service (ClusterIP)
│   ├── auth/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── evaluation/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── flag/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── targeting/
│       ├── deployment.yaml
│       └── service.yaml
│
└── argocd/
    └── applications.yaml          # Definição das 5 Applications do ArgoCD
```

---

## 🧩 Microsserviços Gerenciados

| Serviço | Descrição | Linguagem |
| :--- | :--- | :--- |
| **auth** | Autenticação e emissão de tokens JWT | Go |
| **flag** | CRUD de *feature flags* | Python (Flask) |
| **targeting** | Regras de segmentação/*targeting* | Python (Flask) |
| **evaluation** | Avaliação de flags (consome SQS, usa cache) | Go |
| **analytics** | Coleta e processamento de métricas | Python (Flask) |

---

## ⚙️ Configuração das Applications (ArgoCD)

Cada microsserviço tem uma **Application** definida em [`argocd/applications.yaml`](argocd/applications.yaml) com sincronização automática habilitada:

```yaml
syncPolicy:
  automated:
    prune: true      # remove recursos deletados do Git
    selfHeal: true   # reverte mudanças manuais não autorizadas no cluster
  syncOptions:
    - CreateNamespace=true          # cria o namespace se não existir
    - PrunePropagationPolicy=foreground
    - ApplyOutOfSyncOnly=true       # só aplica o que mudou
```

**Principais características:**

- **`prune: true`** → Se um recurso for removido do Git, o ArgoCD o remove do cluster.
- **`selfHeal: true`** → Qualquer alteração manual feita diretamente no cluster (`kubectl edit`) é automaticamente revertida para o estado declarado no Git.
- **Namespace de destino:** `togglemaster`
- **Cluster de destino:** `https://kubernetes.default.svc` (o próprio cluster onde o ArgoCD está instalado)

---

## 📦 Detalhes dos Manifestos

### Deployment (exemplo: `apps/auth/deployment.yaml`)

- **Réplicas:** 2 (alta disponibilidade)
- **Estratégia:** `RollingUpdate` (`maxSurge: 1`, `maxUnavailable: 0`) — atualização sem downtime
- **Imagem:** proveniente do Amazon ECR — `940104439733.dkr.ecr.sa-east-1.amazonaws.com/togglemaster/<serviço>`
  > ⚠️ A linha da `image` é **atualizada automaticamente pelo pipeline CI/CD** a cada novo deploy.
- **Variáveis de ambiente:** injetadas via `Secret` (ex.: `DATABASE_URL`, `JWT_SECRET`)
- **Health Checks:** `livenessProbe` e `readinessProbe` no endpoint `/health`
- **Resources:** *requests* e *limits* de CPU/memória definidos

### Service (exemplo: `apps/auth/service.yaml`)

- **Tipo:** `ClusterIP` (acessível apenas dentro do cluster)
- **Porta:** `80` → `targetPort` do container

---

## 🚀 Como Aplicar (Setup Inicial)

Estes passos são executados **uma única vez**, para registrar as Applications no ArgoCD:

```bash
# 1. Garantir que o ArgoCD está instalado no cluster
kubectl get pods -n argocd

# 2. Aplicar as definições das Applications
kubectl apply -f argocd/applications.yaml

# 3. Verificar que as Applications foram criadas e sincronizadas
kubectl get applications -n argocd
```

**Saída esperada:**

```
NAME                 SYNC STATUS   HEALTH STATUS
analytics-service    Synced        Healthy
auth-service         Synced        Healthy
evaluation-service   Synced        Healthy
flag-service         Synced        Healthy
targeting-service    Synced        Healthy
```

A partir daí, o ArgoCD passa a monitorar este repositório e **sincroniza automaticamente** qualquer mudança.

---

## 🔍 Comandos Úteis

```bash
# Listar as Applications e seu status de sincronização
kubectl get applications -n argocd

# Ver detalhes de uma Application específica
kubectl describe application auth-service -n argocd

# Acompanhar os pods gerenciados no namespace da aplicação
kubectl get pods -n togglemaster

# Acessar a UI do ArgoCD (port-forward)
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Acesse: https://localhost:8080
```

---

## 🔐 Pré-requisitos no Cluster

Para que os deployments funcionem corretamente, os seguintes recursos devem existir **previamente** no namespace `togglemaster` (criados fora deste repositório, pois contêm dados sensíveis):

- **Secrets** com credenciais de banco de dados, JWT e API keys de cada serviço
- **ConfigMaps** com configurações não sensíveis

> 💡 Os `Secrets` **não são versionados neste repositório** por conterem dados sensíveis. Consulte o guia de setup no repositório principal (`togglemaster-phase3`).

---

## 🤖 Commits Automatizados

Commits feitos pelo usuário `github-actions[bot]` com mensagens no padrão:

```
chore: update <serviço> image to v1.0.0-<commit-hash> [skip ci]
```

são gerados **automaticamente pelo pipeline CI/CD** e representam atualizações da tag de imagem após um build bem-sucedido. A tag `[skip ci]` evita que esses commits disparem novos pipelines.

---

## 📚 Repositórios Relacionados

- **Código & Infraestrutura:** [luisfelipems/togglemaster-phase3](https://github.com/luisfelipems/togglemaster-phase3)
- **GitOps (este):** [luisfelipems/togglemaster-gitops](https://github.com/luisfelipems/togglemaster-gitops)

---

## 👤 Autor

**Luis Felipe Martins da Silva** — RM372751
POSTECH — Tech Challenge Fase 3
