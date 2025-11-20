# Explicação do Pipeline CI/CD GitOps e Procedimentos de Rollback

## Índice
1. [Introdução ao GitOps](#introdução-ao-gitops)
2. [Como o Código Funciona - Passo a Passo](#como-o-código-funciona---passo-a-passo)
3. [Análise Detalhada do Workflow](#análise-detalhada-do-workflow)
4. [Como Realizar Rollback em Produção](#como-realizar-rollback-em-produção)
5. [Boas Práticas](#boas-práticas)

---

## Introdução ao GitOps

GitOps é uma prática de operações que utiliza o Git como fonte única de verdade para infraestrutura declarativa e aplicações. Neste repositório, implementamos um pipeline de CI/CD completo que automatiza o processo de build, deployment e atualização de aplicações Kubernetes.

### Princípios GitOps Aplicados:
- **Git como fonte de verdade**: Todo o estado desejado está no repositório Git
- **Automação**: Mudanças no Git disparam automaticamente o pipeline
- **Declarativo**: Kubernetes manifests descrevem o estado desejado
- **Auditabilidade**: Histórico completo de mudanças via Git

---

## Como o Código Funciona - Passo a Passo

### Visão Geral da Arquitetura

```
┌─────────────┐      ┌──────────────┐      ┌─────────────┐      ┌──────────────┐
│   Código    │ push │  GitHub      │      │   Docker    │      │  Kubernetes  │
│   Fonte     │─────▶│  Actions     │─────▶│   Hub       │─────▶│   Cluster    │
│  (main.go)  │      │  (cd.yaml)   │      │  (imagem)   │      │  (ArgoCD)    │
└─────────────┘      └──────────────┘      └─────────────┘      └──────────────┘
                            │
                            │
                            ▼
                     ┌──────────────┐
                     │ Atualiza     │
                     │ kustomization│
                     │   (Git)      │
                     └──────────────┘
```

### Componentes do Sistema

#### 1. **Aplicação Go (main.go)**
```go
package main

import "net/http"

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("<h1>Hello Argo</h1>"))
    })
    http.ListenAndServe(":3008", nil)
}
```

**O que faz:**
- Cria um servidor HTTP simples em Go
- Escuta na porta 3008
- Responde com "Hello Argo" para todas as requisições na rota raiz "/"

#### 2. **Dockerfile**
```dockerfile
FROM golang:1.25.4 as build
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o server

FROM scratch
WORKDIR /app
COPY --from=build /app/server .
ENTRYPOINT ["./server"]
```

**O que faz - Build Multi-Stage:**

**Stage 1 - Build:**
- Usa imagem `golang:1.25.4` como ambiente de compilação
- Define `/app` como diretório de trabalho
- Copia todo o código fonte para o container
- Compila o código Go gerando um binário chamado `server`
  - `CGO_ENABLED=0`: Desabilita CGO para criar binário estático
  - `GOOS=linux`: Compila para Linux
  - `GOARCH=amd64`: Compila para arquitetura AMD64

**Stage 2 - Runtime:**
- Usa imagem `scratch` (imagem vazia, mínima)
- Copia apenas o binário compilado da stage anterior
- Define o entrypoint como o executável `./server`
- **Vantagem**: Imagem final extremamente pequena (~6MB vs ~800MB da imagem golang completa)

#### 3. **Manifests Kubernetes (k8s/)**

**deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goserver
spec:
  selector:
    matchLabels:
      app: goserver
  template:
    metadata:
      labels:
        app: goserver
    spec:
      containers:
      - name: goserver
        image: goserver
        ports:
        - containerPort: 3008
```

**O que faz:**
- Define um Deployment Kubernetes
- Gerencia pods com label `app: goserver`
- Container usa a imagem `goserver` (será substituída pelo Kustomize)
- Expõe porta 3008

**service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: goserver-service
spec:
  selector:
    app: goserver
  ports:
  - port: 3008
    targetPort: 3008
```

**O que faz:**
- Cria um Service Kubernetes
- Roteia tráfego para pods com label `app: goserver`
- Expõe porta 3008

**kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml

images:
- name: goserver
  newName: /gitopsfc
  newTag: ee76dc9c56a79ec5830855dbb98be858e8acdcd8
```

**O que faz:**
- Agrupa os recursos Kubernetes
- Substitui a imagem `goserver` por uma imagem específica com tag (SHA do commit)
- Permite gerenciar múltiplos ambientes sem duplicar YAMLs

---

## Análise Detalhada do Workflow

### GitHub Actions Workflow (.github/workflows/cd.yaml)

Vamos analisar cada seção do pipeline:

#### **1. Trigger (Gatilho)**
```yaml
on: 
  push:
    branches: [main]
```

**Por que funciona:**
- O pipeline é disparado automaticamente em cada push para a branch `main`
- Garante que apenas mudanças aprovadas (na branch principal) sejam deployadas
- Implementa **Continuous Deployment**: código → produção automaticamente

#### **2. Permissões**
```yaml
permissions:
  contents: write
```

**Por que funciona:**
- Concede ao GitHub Actions permissão para escrever no repositório
- Necessário para o pipeline commitar mudanças no `kustomization.yaml`
- Essencial para o padrão GitOps: o pipeline atualiza o estado no Git

#### **3. Job: Build**

##### **Step 1: Checkout Code**
```yaml
- name: Checkout code
  uses: actions/checkout@v2
```

**O que acontece:**
- Clona o repositório para o runner do GitHub Actions
- Disponibiliza o código fonte para os próximos steps

##### **Step 2: Build and Push Image**
```yaml
- name: Build and push image to Dockerhub
  uses: docker/build-push-action@v3.0.0
  with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}
    repository: ${{ secrets.DOCKER_USERNAME }}/gitopsfc
    tags: ${{ github.sha }}, latest
```

**O que acontece:**
1. Autentica no Docker Hub usando secrets armazenados no GitHub
2. Executa `docker build` usando o Dockerfile na raiz
3. Cria duas tags para a imagem:
   - `${{ github.sha }}`: SHA único do commit (ex: `ee76dc9c...`)
   - `latest`: Tag sempre apontando para a versão mais recente
4. Faz push da imagem para Docker Hub

**Por que duas tags?**
- `latest`: Facilita desenvolvimento/teste
- `SHA commit`: Rastreabilidade e rollback preciso

##### **Step 3: Setup Kustomize**
```yaml
- name: Setup Kustomize
  uses: imranismail/setup-kustomize@v1
  with: 
    kustomize-version: "5.3.0"
```

**O que acontece:**
- Instala o Kustomize v5.3.0 no runner
- Kustomize é uma ferramenta para customizar manifestos Kubernetes

##### **Step 4: Update Kubernetes Resources**
```yaml
- name: Update Kubernetes resources
  env:
    DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
  run: |
    cd k8s
    kustomize edit set image goserver=$DOCKER_USERNAME/gitopsfc:$GITHUB_SHA
```

**O que acontece:**
1. Navega para o diretório `k8s/`
2. Executa comando Kustomize que atualiza `kustomization.yaml`
3. Substitui a tag da imagem pelo SHA do commit atual

**Exemplo prático:**
```yaml
# Antes
images:
- name: goserver
  newName: usuario/gitopsfc
  newTag: abc123...

# Depois
images:
- name: goserver
  newName: usuario/gitopsfc
  newTag: ee76dc9c56a79ec5830855dbb98be858e8acdcd8  # Novo SHA
```

**Por que isso é importante:**
- Atualiza o estado desejado no Git
- ArgoCD (ou outro CD tool) detecta a mudança e aplica no cluster

##### **Step 5: Commit**
```yaml
- name: Commit
  run: |
    git config --local user.email "action@github.com"
    git config --local user.name "GitHub Action"
    git commit -am "Bump docker version"
```

**O que acontece:**
1. Configura Git com usuário/email do bot
2. Adiciona todas as mudanças (o `kustomization.yaml` alterado)
3. Cria commit com mensagem "Bump docker version"

**Por que funciona:**
- Registra a mudança no histórico do Git
- Mantém auditabilidade: cada versão tem um commit

##### **Step 6: Push**
```yaml
- name: Push
  uses: ad-m/github-push-action@master
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    repository: Daian4/GitOps
```

**O que acontece:**
1. Faz push do commit para o repositório GitHub
2. Usa token automático do GitHub (não precisa configurar secret)
3. Atualiza a branch `main` com novo estado

**Por que funciona:**
- Completa o ciclo GitOps: mudança de código → build → atualização do estado
- ArgoCD detecta a mudança no repositório e sincroniza com Kubernetes

---

## Fluxo Completo em Ação

### Cenário: Desenvolvedor faz mudança no código

```
1. Desenvolvedor edita main.go
   └─> git commit -m "feat: add new endpoint"
   └─> git push origin main

2. GitHub Actions detecta push na main
   └─> Pipeline cd.yaml inicia automaticamente

3. Build da aplicação
   └─> Docker build cria nova imagem
   └─> Push para DockerHub com tag do SHA commit

4. Atualização do manifesto
   └─> Kustomize atualiza kustomization.yaml
   └─> Novo SHA da imagem é registrado

5. Commit e Push do estado
   └─> GitHub Action commita mudança
   └─> Push para repositório

6. ArgoCD detecta mudança
   └─> Compara estado atual vs desejado
   └─> Aplica novos manifests no Kubernetes
   └─> Pods são recriados com nova imagem

7. Aplicação atualizada em produção! ✅
```

---

## Como Realizar Rollback em Produção

### Conceito de Rollback em GitOps

Em GitOps, **o rollback é simplesmente reverter uma mudança no Git**. Como o Git é a fonte de verdade, reverter um commit automaticamente reverte o estado da aplicação.

### Método 1: Rollback via Git Revert (RECOMENDADO)

Este método cria um novo commit que desfaz as mudanças, mantendo o histórico completo.

#### **Passo a Passo:**

```bash
# 1. Clone o repositório (se não tiver localmente)
git clone https://github.com/Daian4/GitOps.git
cd GitOps

# 2. Verifique o histórico de commits
git log --oneline

# Saída exemplo:
# abc1234 Bump docker version  ← Commit atual (com bug)
# def5678 Bump docker version  ← Versão anterior (estável)
# ghi9012 Bump docker version

# 3. Identifique o commit que deseja reverter (o commit problemático)
# Vamos reverter o commit abc1234

git revert abc1234

# Isso cria um novo commit que desfaz as mudanças de abc1234

# 4. Push para disparar o pipeline
git push origin main
```

**O que acontece:**
1. `git revert` cria um novo commit que desfaz as mudanças
2. Push dispara o pipeline do GitHub Actions
3. Pipeline identifica que `kustomization.yaml` foi revertido
4. Nova imagem NÃO é construída (código não mudou)
5. ArgoCD detecta mudança no `kustomization.yaml`
6. ArgoCD aplica a versão anterior da imagem no Kubernetes
7. **Resultado**: Aplicação volta à versão estável

**Vantagens:**
- ✅ Mantém histórico completo
- ✅ Auditável: fica claro que houve rollback
- ✅ Pode ser revertido novamente se necessário
- ✅ Segue boas práticas de Git

### Método 2: Rollback Manual do kustomization.yaml

Se você sabe exatamente qual versão da imagem deseja, pode editar manualmente.

#### **Passo a Passo:**

```bash
# 1. Encontre o SHA do commit da versão estável
git log --oneline k8s/kustomization.yaml

# Saída exemplo:
# abc1234 Bump docker version  (imagem: usuario/gitopsfc:abc1234)
# def5678 Bump docker version  (imagem: usuario/gitopsfc:def5678) ← QUEREMOS ESTA
# ghi9012 Bump docker version

# 2. Edite k8s/kustomization.yaml
vim k8s/kustomization.yaml

# 3. Altere a tag da imagem para a versão estável
images:
- name: goserver
  newName: usuario/gitopsfc
  newTag: def5678  # ← Altere para SHA da versão estável

# 4. Commit e push
git add k8s/kustomization.yaml
git commit -m "rollback: revert to stable version def5678"
git push origin main
```

**O que acontece:**
1. Mudança no `kustomization.yaml` é detectada pelo ArgoCD
2. ArgoCD baixa a imagem com tag `def5678` do Docker Hub
3. Kubernetes recria os pods com a imagem antiga
4. **Resultado**: Rollback completo

### Método 3: Rollback via Interface do ArgoCD (Mais Rápido)

Se você tem ArgoCD configurado, pode fazer rollback direto pela UI.

#### **Passo a Passo:**

```bash
# Via UI do ArgoCD:
1. Acesse a interface web do ArgoCD
2. Selecione a aplicação "goserver"
3. Clique em "History and Rollback"
4. Veja lista de revisões anteriores
5. Selecione a revisão estável
6. Clique em "Rollback"

# Via CLI do ArgoCD:
argocd app rollback goserver <REVISION_NUMBER>

# Exemplo:
argocd app rollback goserver 5  # Volta para revisão 5
```

**O que acontece:**
1. ArgoCD sincroniza o cluster com estado anterior
2. Kubernetes aplica manifests da revisão selecionada
3. **Resultado**: Rollback imediato (segundos)

**Atenção:**
- ⚠️ Isso NÃO atualiza o Git automaticamente
- ⚠️ Você precisa atualizar `kustomization.yaml` manualmente depois
- ⚠️ Caso contrário, próxima sincronização sobrescreve o rollback

### Método 4: Rollback de Emergência - Kubectl

Para situações críticas onde você precisa de rollback IMEDIATO.

#### **Passo a Passo:**

```bash
# 1. Conecte ao cluster Kubernetes
kubectl config use-context production

# 2. Verifique o histórico de deployments
kubectl rollout history deployment/goserver

# Saída exemplo:
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         Bump docker version (def5678)
# 3         Bump docker version (abc1234)  ← Versão atual (com bug)

# 3. Rollback para revisão anterior
kubectl rollout undo deployment/goserver

# OU especifique uma revisão específica:
kubectl rollout undo deployment/goserver --to-revision=2

# 4. Verifique o status do rollback
kubectl rollout status deployment/goserver

# 5. IMPORTANTE: Atualize o Git para refletir o estado
# Edite k8s/kustomization.yaml para apontar para a tag correta
# Commit e push para manter Git como fonte de verdade
```

**O que acontece:**
1. Kubernetes reverte o Deployment para configuração anterior
2. Pods antigos são recriados com imagem estável
3. **Resultado**: Rollback em ~10-30 segundos

**⚠️ ATENÇÃO - DRIFT:**
- Este método cria "drift" entre Git (fonte de verdade) e cluster
- Você DEVE atualizar `kustomization.yaml` logo após
- Caso contrário, próxima sincronização do ArgoCD sobrescreve o rollback

---

## Comparação dos Métodos de Rollback

| Método | Velocidade | Mantém Histórico | GitOps Compliant | Complexidade | Quando Usar |
|--------|-----------|------------------|------------------|--------------|-------------|
| **Git Revert** | 2-5 min | ✅ Sim | ✅ Sim | Baixa | Rollback padrão, situação normal |
| **Manual Edit** | 2-5 min | ✅ Sim | ✅ Sim | Baixa | Quando você sabe exatamente a versão |
| **ArgoCD UI/CLI** | 10-30 seg | ⚠️ Parcial | ⚠️ Requer sync Git | Média | Rollback rápido, mas requer ajuste Git |
| **Kubectl** | 5-15 seg | ❌ Não | ❌ Cria drift | Baixa | EMERGÊNCIA apenas |

---

## Cenários Práticos de Rollback

### Cenário 1: Bug Descoberto em Produção

**Situação:** 
- Nova versão deployada às 14h
- Usuários reportam erro 500 às 14h15
- Precisa fazer rollback imediato

**Ação:**
```bash
# Opção 1: ArgoCD (mais rápido)
argocd app rollback goserver --to-revision=5

# Depois, sincronize o Git:
git revert HEAD
git push origin main

# Opção 2: Kubectl (emergência)
kubectl rollout undo deployment/goserver

# Depois, sincronize o Git:
# Edite k8s/kustomization.yaml
git commit -am "rollback: revert to stable version"
git push origin main
```

### Cenário 2: Rollback Planejado

**Situação:**
- Deploy de nova feature às 10h
- Testes A/B mostram performance pior
- Decisão de rollback após análise

**Ação:**
```bash
# Use git revert (melhor prática)
git log --oneline -5
# Identifique o commit da nova feature

git revert <commit-sha>
git push origin main

# Pipeline automático faz o resto
```

### Cenário 3: Rollback Parcial

**Situação:**
- Múltiplos commits foram feitos
- Só um deles tem problema
- Quer manter os outros

**Ação:**
```bash
# Revert apenas o commit problemático
git log --oneline k8s/kustomization.yaml
git revert <commit-problemático>

# OU edite manualmente para versão específica
vim k8s/kustomization.yaml
# Altere newTag para SHA desejado
git commit -am "rollback: revert image to specific version"
git push origin main
```

---

## Boas Práticas

### 1. **Sempre Teste em Ambientes Não-Produtivos Primeiro**

```yaml
# Exemplo: Use branches diferentes para ambientes
trigger:
  push:
    branches: 
      - main        # Produção
      - staging     # Staging
      - develop     # Desenvolvimento
```

### 2. **Implemente Health Checks**

Adicione ao deployment:

```yaml
spec:
  containers:
  - name: goserver
    image: goserver
    livenessProbe:
      httpGet:
        path: /health
        port: 3008
      initialDelaySeconds: 10
      periodSeconds: 5
    readinessProbe:
      httpGet:
        path: /ready
        port: 3008
      initialDelaySeconds: 5
      periodSeconds: 3
```

### 3. **Use Tags Semânticas (SemVer)**

Além de SHA, considere tags semânticas:

```bash
# No pipeline, adicione tag com versão
docker tag usuario/gitopsfc:$GITHUB_SHA usuario/gitopsfc:v1.2.3
docker push usuario/gitopsfc:v1.2.3
```

### 4. **Monitore e Observe**

Implemente:
- **Logs centralizados** (ELK, Loki)
- **Métricas** (Prometheus, Grafana)
- **Alertas** (AlertManager, PagerDuty)
- **Tracing** (Jaeger, Zipkin)

### 5. **Tenha Runbook de Rollback**

Documente:
- ✅ Quem pode fazer rollback
- ✅ Em quais situações
- ✅ Procedimento passo a passo
- ✅ Contatos para escalonamento

### 6. **Blue-Green ou Canary Deployments**

Para deploys mais seguros:

```yaml
# Exemplo de Canary com Argo Rollouts
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: goserver
spec:
  replicas: 5
  strategy:
    canary:
      steps:
      - setWeight: 20    # 20% do tráfego para nova versão
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 100   # 100% para nova versão
```

### 7. **Backup e Restore**

```bash
# Backup dos manifests Kubernetes
kubectl get all -n default -o yaml > backup-$(date +%Y%m%d).yaml

# Backup do estado do ArgoCD
argocd app get goserver -o yaml > argocd-backup.yaml
```

### 8. **Comunicação**

Em caso de rollback:
- 📢 Notifique equipe no Slack/Teams
- 📝 Documente em incident report
- 📊 Analise causa raiz após estabilização

---

## Diagrama: Fluxo de Rollback

```
╔════════════════════════════════════════════════════════════════╗
║                    ROLLBACK EM PRODUÇÃO                        ║
╚════════════════════════════════════════════════════════════════╝

┌─────────────────────────────────────────────────────────────┐
│ 1. DETECÇÃO DO PROBLEMA                                      │
│    - Alertas de monitoramento                                │
│    - Relatórios de usuários                                  │
│    - Testes automáticos falhando                             │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. DECISÃO DE ROLLBACK                                       │
│    - Avaliar severidade                                      │
│    - Identificar versão estável                              │
│    - Escolher método de rollback                             │
└────────────────────┬────────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
┌──────────────┐         ┌──────────────┐
│ EMERGÊNCIA?  │         │   NORMAL     │
│              │         │              │
│ - kubectl    │         │ - git revert │
│   rollback   │         │ - ArgoCD     │
│              │         │              │
│ 5-15 seg     │         │ 2-5 min      │
└──────┬───────┘         └──────┬───────┘
       │                        │
       │                        │
       └────────────┬───────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. SINCRONIZAR GIT (se necessário)                           │
│    - Atualizar kustomization.yaml                            │
│    - Commit e push                                           │
│    - Evitar drift                                            │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. VERIFICAÇÃO                                               │
│    - Checar saúde dos pods                                   │
│    - Validar métricas                                        │
│    - Testes smoke                                            │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. POST-MORTEM                                               │
│    - Documentar incidente                                    │
│    - Análise de causa raiz                                   │
│    - Plano de ação preventivo                                │
└─────────────────────────────────────────────────────────────┘
```

---

## Conclusão

Este sistema GitOps funciona porque:

1. **Separação de Responsabilidades:**
   - CI (GitHub Actions) → Build e atualização de estado
   - CD (ArgoCD) → Sincronização do estado com cluster
   - Git → Fonte única de verdade

2. **Automação Completa:**
   - Cada push dispara pipeline
   - Pipeline atualiza estado automaticamente
   - ArgoCD sincroniza continuamente

3. **Rastreabilidade:**
   - Cada versão tem SHA único
   - Histórico completo no Git
   - Rollback é simplesmente reverter commit

4. **Segurança:**
   - Imagens imutáveis (tagged by SHA)
   - Auditoria completa
   - Processo repetível

Para realizar rollback em produção como um engenheiro experiente:
- **Use git revert para rollback padrão** (mantém histórico)
- **Use kubectl para emergências** (mais rápido, mas cria drift)
- **Use ArgoCD para rollback visual** (bom equilíbrio)
- **SEMPRE sincronize Git após rollback manual**
- **Documente tudo e aprenda com incidentes**

---

## Recursos Adicionais

- [Documentação Oficial do Kustomize](https://kustomize.io/)
- [Documentação Oficial do ArgoCD](https://argo-cd.readthedocs.io/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitOps Principles](https://opengitops.dev/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

---

**Autor:** GitHub Copilot  
**Data:** 2025-11-20  
**Versão:** 1.0
