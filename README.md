# helm-bt

Helm chart genérico da BT para deploy de aplicações no Kubernetes.

---

## 1. Adicionar o repositório Helm

```bash
helm repo add helm-bt https://raw.githubusercontent.com/VerdeCard-BusinessTech/helm-bt/main/
helm repo update
```

---

## 2. Exportar o `values.yaml` de referência

Para gerar um arquivo `values.yaml` com todos os campos disponíveis e seus valores padrão:

```bash
helm show values helm-bt/chartBt > values.yaml
```

Isso cria um `values.yaml` no diretório atual que serve como base para configurar seu projeto.

---

## 3. Estrutura do `values.yaml`

Abaixo estão todos os campos disponíveis com explicação de cada um.

### Configurações do projeto

| Campo | Descrição | Exemplo |
|---|---|---|
| `projectName` | Nome do projeto. Usado em labels, selectors e hostnames do ingress. | `api-voll` |

### Container

| Campo | Descrição | Exemplo |
|---|---|---|
| `container.replicas` | Número de réplicas do deployment | `1` |
| `container.restartPolicy` | Política de restart do pod | `Always` |
| `container.terminationGracePeriodSeconds` | Tempo (em segundos) para o pod finalizar antes de ser encerrado | `30` |

### Imagem

| Campo | Descrição | Exemplo |
|---|---|---|
| `container.image.path` | Caminho completo da imagem (registry/imagem:tag) | `myregistry.com/api:v1.0.0` |
| `container.image.pullPolicy` | Política de pull da imagem (`Always`, `IfNotPresent`, `Never`) | `IfNotPresent` |
| `container.image.pullSecrets` | Nome do secret para autenticação no registry | `bt-pipeline` |

### Variáveis de ambiente

```yaml
container:
  env:
    - DATABASE_URL: "postgres://user:pass@host:5432/db"
    - API_KEY: "my-api-key"
```

### Argumentos do container (opcional)

```yaml
container:
  args:
    - "-text=banana"
    - "--port=8080"
```

### Health checks

```yaml
container:
  livenessProbe:
    enabled: true            # ativar/desativar liveness
    httpGet:
      enabled: true
      path: /livez           # endpoint de liveness
      port: http             # nome da porta
    initialDelaySeconds: 30  # delay antes da primeira verificação
    periodSeconds: 10        # intervalo entre verificações
    failureThreshold: 3      # falhas antes de reiniciar

  readinessProbe:
    enabled: true            # ativar/desativar readiness
    httpGet:
      enabled: true
      path: /livez           # endpoint de readiness
      port: http             # nome da porta
    initialDelaySeconds: 30
    periodSeconds: 10
    successThreshold: 3      # sucessos para ser considerado pronto
    failureThreshold: 3
```

### Portas

```yaml
container:
  ports:
    - name: http
      containerPort: 8080
      protocol: TCP
    - name: grpc
      containerPort: 9090
      protocol: TCP
```

### Recursos (CPU e Memória)

```yaml
container:
  resources:
    enabled: true
    limits:
      cpu: 100m
      memory: 100Mi
    requests:
      cpu: 10m
      memory: 10Mi
```

### Estratégia de deploy

| Campo | Descrição | Valores |
|---|---|---|
| `container.strategy.enabled` | Ativar configuração de estratégia | `true` / `false` |
| `container.strategy.type` | Tipo da estratégia | `RollingUpdate` ou `Recreate` |
| `container.strategy.rollingUpdate.maxSurge` | Pods extras durante update | `1` |
| `container.strategy.rollingUpdate.maxUnavailable` | Pods indisponíveis durante update | `0` |

### DNS / Ingress

```yaml
dns:
  http:
    enabled: true             # cria ingress HTTP
    paths:
      - pathName: /v1
        pathType: Prefix
        port: 8080
  grpc:
    enabled: true             # cria ingress gRPC
    paths:
      - pathName: /
        pathType: Prefix
        port: 9090
  public:                     # (opcional) domínios públicos adicionais
    domain:
      - mydomain.com
      - mydomain.xyz
    absolute:                 # hosts absolutos (sem prefixo do projectName)
      - abs.mydomain.br
```

> O hostname interno é gerado automaticamente como `{projectName}.verdecard.bt`.
> Domínios em `public.domain` geram `{projectName}.{domain}`.
> Domínios em `public.absolute` são usados como host completo.

---

## 4. Montar o `values.yaml` do seu projeto

1. Exporte o values de referência (passo 2).
2. Edite os valores conforme a necessidade do seu projeto.

**Exemplo mínimo** para um serviço HTTP simples:

```yaml
projectName: meu-servico

container:
  replicas: 2
  restartPolicy: Always
  terminationGracePeriodSeconds: 30

  image:
    path: myregistry.com/meu-servico:v1.0.0
    pullPolicy: IfNotPresent
    pullSecrets: bt-pipeline

  env:
    - DATABASE_URL: "postgres://user:pass@host:5432/db"

  livenessProbe:
    enabled: true
    httpGet:
      enabled: true
      path: /healthz
      port: http
    initialDelaySeconds: 15
    periodSeconds: 10
    failureThreshold: 3

  readinessProbe:
    enabled: true
    httpGet:
      enabled: true
      path: /healthz
      port: http
    initialDelaySeconds: 15
    periodSeconds: 10
    successThreshold: 1
    failureThreshold: 3

  ports:
    - name: http
      containerPort: 8080
      protocol: TCP

  resources:
    enabled: true
    limits:
      cpu: 200m
      memory: 256Mi
    requests:
      cpu: 50m
      memory: 64Mi

  strategy:
    enabled: true
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

dns:
  http:
    enabled: true
    paths:
      - pathName: /
        pathType: Prefix
        port: 8080
  grpc:
    enabled: false
```

---

## 5. Deploy

### Primeiro deploy (install)

```bash
helm install <nome-release> helm-bt/chartBt \
  -f values.yaml \
  -n <namespace>
```

Exemplo:

```bash
helm install meu-servico helm-bt/chartBt \
  -f values.yaml \
  -n meu-namespace
```

### Atualizar deploy existente (upgrade)

```bash
helm upgrade <nome-release> helm-bt/chartBt \
  -f values.yaml \
  -n <namespace>
```

### Install ou upgrade automático

```bash
helm upgrade --install <nome-release> helm-bt/chartBt \
  -f values.yaml \
  -n <namespace>
```

### Verificar status

```bash
helm list -n <namespace>
kubectl get pods -n <namespace>
```

### Rollback

```bash
helm rollback <nome-release> <revisão> -n <namespace>
```

### Remover

```bash
helm uninstall <nome-release> -n <namespace>
```

---

## 6. Dicas

- Use `helm template . -f values.yaml` para renderizar os manifests localmente sem aplicar no cluster.
- Use `helm lint .` para validar o chart antes de empacotar.
- Defina `dns.grpc.enabled: false` se o serviço não usar gRPC (evita criar ingress desnecessário).
- Mantenha `pullPolicy: IfNotPresent` para melhor performance de deploy.
- O secret `bt-pipeline` deve existir no namespace antes do deploy.