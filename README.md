# togglemaster-gitops

Fonte da verdade do que está rodando no cluster EKS do **ToggleMaster**
(Tech Challenge Fase 3 — POSTECH FIAP).

Este repositório contém **apenas manifestos Kubernetes**. Não há código de
aplicação, não há Dockerfile e não há segredo. O código das aplicações e a
infraestrutura vivem em
[`tech-challenge-fase-3`](https://github.com/Marcelo6arcia/tech-challenge-fase-3).

```
tech-challenge-fase-3   →  como o software é construído
togglemaster-gitops     →  o que está rodando agora
```

---

## Como uma alteração chega ao cluster

```
1. Dev faz push em tech-challenge-fase-3
2. GitHub Actions: build → lint → SAST → SCA → imagem → scan → push no ECR
3. Último job do CI:
       kustomize edit set image <servico>=<ecr>/<servico>:v1.0.0-<sha>
       git commit && git push          ← neste repositório
4. Argo CD detecta o commit (reconciliação a cada 30 s)
5. Argo CD aplica no cluster
```

Nenhum ser humano roda `kubectl apply`. Como `selfHeal: true` está ligado,
qualquer alteração feita direto no cluster é revertida na reconciliação
seguinte.

---

## Estrutura

```
.
├── argocd/applications/
│   ├── platform.yaml                 Application da camada de plataforma
│   └── togglemaster-services.yaml    ApplicationSet: 1 Application por serviço
│
├── apps/
│   ├── base/<servico>/               Manifestos comuns a todos os ambientes
│   │   ├── deployment.yaml           securityContext endurecido, 3 probes
│   │   ├── service.yaml              ClusterIP 80 → 8080
│   │   ├── configmap.yaml            variáveis não sensíveis
│   │   ├── serviceaccount.yaml       annotation de IRSA quando aplicável
│   │   ├── hpa.yaml                  evaluation-service e analytics-service
│   │   └── pdb.yaml                  serviços com mais de uma réplica
│   │
│   └── overlays/dev/<servico>/
│       └── kustomization.yaml        namespace, tag da imagem, ARN da role IRSA
│
└── platform/
    ├── external-secrets/             SecretStore + 6 ExternalSecrets
    ├── bootstrap/                    esquema dos bancos + Job PreSync
    ├── ingress.yaml                  roteamento dos 5 serviços por prefixo
    └── kustomization.yaml
```

---

## O bloco que o CI escreve

Em cada `apps/overlays/dev/<servico>/kustomization.yaml`:

```yaml
images:
  - name: flag-service
    newName: 123456789012.dkr.ecr.us-east-1.amazonaws.com/flag-service
    newTag: v1.0.0-a1b2c3d      # ← escrito pelo pipeline
```

Não edite `newTag` à mão: o próximo build sobrescreve.

---

## Segredos

Não existe nenhum `Secret` com valor neste repositório. O que está versionado é
o **ExternalSecret** — a referência a um segredo do AWS Secrets Manager:

```yaml
data:
  - secretKey: DATABASE_URL
    remoteRef:
      key: togglemaster/dev/flag-service/database
      property: DATABASE_URL
```

O External Secrets Operator, autenticado por IRSA, lê o Secrets Manager e
materializa o `Secret` do Kubernetes em runtime. A senha foi gerada pelo
Terraform e nunca foi digitada por ninguém.

---

## Rotas expostas pelo Ingress

| Rota pública | Serviço | Rota interna |
|---|---|---|
| `/auth/health`, `/auth/validate`, `/auth/admin/keys` | auth-service | `/health`, `/validate`, `/admin/keys` |
| `/flag/health`, `/flag/flags`, `/flag/flags/<nome>` | flag-service | `/health`, `/flags`, `/flags/<nome>` |
| `/targeting/health`, `/targeting/rules`, `/targeting/rules/<flag>` | targeting-service | `/health`, `/rules`, `/rules/<flag>` |
| `/evaluation/health`, `/evaluation/evaluate` | evaluation-service | `/health`, `/evaluate` |
| `/analytics/health` | analytics-service | `/health` |

O prefixo é removido pelo `rewrite-target`. Os cinco serviços expõem `/health` —
sem o prefixo, não haveria como distingui-los.

---

## Configuração da conta

Os overlays já apontam para a conta `456788081240` em `us-east-1`: o registry
do ECR e os ARNs das roles IRSA. Para apontar para outra conta:

```bash
grep -rl 456788081240 apps/ | xargs sed -i '' "s/456788081240/<nova-conta>/g"
```

Os valores corretos saem do Terraform:

```bash
terraform -chdir=terraform/envs/dev/infra output -json gitops_wiring
```

---

## Validar antes de commitar

```bash
for d in apps/overlays/dev/*/ platform/; do
  echo "== $d"; kustomize build "$d" > /dev/null && echo OK
done
```

---

## Operações comuns

| Operação | Comando |
|---|---|
| Rollback de um serviço | `git revert <commit de deploy> && git push` |
| Ver o histórico de deploys | `git log --oneline -- apps/overlays/dev/<servico>` |
| Trocar uma variável de ambiente | editar `apps/base/<servico>/configmap.yaml` |
| Ajustar réplicas ou limites do HPA | editar `apps/base/<servico>/hpa.yaml` |
| Forçar uma sincronização | `argocd app sync <servico>` ou o botão **Sync** na UI |
