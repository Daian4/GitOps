# GitOps

Este repositório demonstra um pipeline completo de CI/CD usando GitOps com GitHub Actions, Docker e Kubernetes.

## 📚 Documentação

Para entender como o pipeline funciona e como realizar rollback em produção, consulte:

**[📖 Explicação Completa do CI/CD e Procedimentos de Rollback](./EXPLICACAO_CICD_ROLLBACK.md)**

Este documento contém:
- Explicação passo a passo de como o código funciona
- Análise detalhada do workflow do GitHub Actions
- Procedimentos completos de rollback em produção
- Boas práticas e cenários práticos

## 🚀 Componentes

- **Aplicação**: Servidor HTTP simples em Go
- **CI/CD**: GitHub Actions (`.github/workflows/cd.yaml`)
- **Container**: Docker multi-stage build
- **Orquestração**: Kubernetes + Kustomize
- **GitOps**: ArgoCD (recomendado)

## 🔄 Fluxo de Deploy

1. Push para `main` → Dispara pipeline
2. Build da imagem Docker
3. Push para Docker Hub com tag do commit SHA
4. Atualização automática do `kustomization.yaml`
5. ArgoCD sincroniza com Kubernetes
