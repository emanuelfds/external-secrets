<div align="center">
  <img src="images/eso-logo.png" alt="External Secrets Operator" width="150" />

  <h1>External Secrets Operator × Vaultwarden API</h1>

  <p><strong>Segredos do Vaultwarden sincronizados automaticamente para o Kubernetes.</strong></p>

  <p>
    Integração declarativa, em YAML puro, entre o
    <a href="https://external-secrets.io/">External Secrets Operator</a> (ESO)
    e o <a href="https://github.com/Turbootzz/Vaultwarden-API">vaultwarden-api</a>,
    usando o provider genérico <code>webhook</code>.
  </p>

  <p>
    <img alt="External Secrets Operator v0.20.2" src="https://img.shields.io/badge/ESO-v0.20.2-5B4EE9?logo=kubernetes&logoColor=white" />
    <img alt="Kubernetes" src="https://img.shields.io/badge/Kubernetes-%E2%89%A51.19-326CE5?logo=kubernetes&logoColor=white" />
    <img alt="Webhook provider" src="https://img.shields.io/badge/provider-webhook-0AA1DD" />
    <img alt="Vaultwarden" src="https://img.shields.io/badge/backend-Vaultwarden-175DDC?logo=bitwarden&logoColor=white" />
    <img alt="YAML only" src="https://img.shields.io/badge/deploy-kubectl%20apply%20--f-6B4FBB?logo=yaml&logoColor=white" />
    <img alt="MIT License" src="https://img.shields.io/badge/license-MIT-2EA44F" />
  </p>
</div>

> **Status:** manifests prontos para aplicação em cluster Kubernetes. A instalação utiliza YAML estático e não depende de Helm, Kustomize ou ArgoCD durante o deploy.

---

## Sumário

- [Visão geral](#visão-geral)
- [Por que esta arquitetura](#por-que-esta-arquitetura)
- [Conceitos fundamentais](#conceitos-fundamentais)
- [Arquitetura](#arquitetura)
- [Pré-requisitos](#pré-requisitos)
- [Quick start](#quick-start)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Como cadastrar segredos no Vaultwarden](#como-cadastrar-segredos-no-vaultwarden)
- [Referência dos recursos](#referência-dos-recursos)
- [Segurança](#segurança)
- [Operação](#operação)
- [Atualização e rollback](#atualização-e-rollback)
- [Troubleshooting](#troubleshooting)
- [Decisões de implementação](#decisões-de-implementação)
- [Referências](#referências)
- [Licença](#licença)

---

## Visão geral

Este projeto conecta três componentes:

| Camada | Componente | Responsabilidade |
|---|---|---|
| Backend | `vaultwarden-api` | Consulta e descriptografa os itens do Vaultwarden e expõe `GET /secret/<nome>`. |
| Integração | External Secrets Operator | Consulta o backend externo e reconcilia `Secrets` nativos do Kubernetes. |
| Aplicação | `ExternalSecret` | Declara quais itens devem ser buscados e em qual `Secret` devem ser entregues. |

O `vaultwarden-api` retorna os valores no formato:

```http
GET /secret/DATABASE_URL
Authorization: Bearer <API_KEY>
Accept: application/json
```

```json
{
  "name": "DATABASE_URL",
  "value": "postgresql://user:password@database:5432/app"
}
```

O ESO extrai o campo `value` e cria ou atualiza um `Secret` do Kubernetes. As aplicações consomem esse `Secret` por `env`, `envFrom` ou volume, sem acessar diretamente o Vaultwarden.

> **Importante:** este `vaultwarden-api` é a API REST do projeto Turbootzz. Ele não é o `bitwarden-sdk-server` esperado pelo provider `bitwarden` nativo do ESO. Por isso, esta integração usa o provider genérico `webhook`.

### O que este projeto entrega

- ESO v0.20.2 instalado a partir de um bundle YAML estático.
- CRDs, RBAC, controller, webhook e cert-controller do ESO.
- `ClusterSecretStore` reutilizável por aplicações em diferentes namespaces.
- Autenticação com `Authorization: Bearer <API_KEY>`.
- Exemplos de segredo único, múltiplos segredos e template composto.
- Affinity obrigatória para nodes com `apps=services`.
- Toleration para o taint existente `servicesonly=true:PreferNoSchedule`.
- Documentação de operação, rotação, atualização e troubleshooting.

### O que este projeto não faz

- Não instala ou configura o Vaultwarden.
- Não cria itens dentro do cofre do Vaultwarden.
- Não grava a `API_KEY` real no Git.
- Não usa Helm, Kustomize ou ArgoCD durante a aplicação dos manifests.
- Não expõe o `vaultwarden-api` publicamente; utiliza o Service interno do cluster.

---

## Por que esta arquitetura

A aplicação não precisa conhecer a API do Vaultwarden nem carregar credenciais de acesso ao cofre. O ESO centraliza a reconciliação e entrega apenas os valores declarados no `ExternalSecret`.

### Benefícios

- **Declarativo:** a intenção da aplicação fica descrita em YAML.
- **Automático:** alterações no Vaultwarden são sincronizadas conforme o `refreshInterval`.
- **Desacoplado:** a aplicação consome um `Secret` Kubernetes padrão.
- **Auditável:** Store, mapeamentos e destino podem ser revisados separadamente.
- **Menor exposição:** o `vaultwarden-api` permanece acessível somente pela rede interna.
- **Reutilizável:** um `ClusterSecretStore` pode atender vários namespaces.

### Trade-offs

| Decisão | Consequência |
|---|---|
| `ClusterSecretStore` | Simples para múltiplas aplicações, mas exige controle de RBAC e revisão dos `ExternalSecrets`. |
| HTTP interno | Evita exposição externa, mas não oferece criptografia entre pods. Use HTTPS se o modelo de ameaça exigir. |
| Bundle ESO versionado | Deploy reproduzível, mas upgrades exigem renderização e revisão do YAML estático. |
| API key compartilhada pelo Store | Operação simples; para menor privilégio, use chaves escopadas no `vaultwarden-api` ou Stores por namespace. |

---

## Conceitos fundamentais

A ordem de aplicação é deliberada:

```text
00-operator  →  01-store  →  02-examples
```

### 1. Operator — `k8s/00-operator/`

O **External Secrets Operator** é o software que roda no cluster. Ele instala as CRDs e executa os controllers que observam recursos como `SecretStore`, `ClusterSecretStore` e `ExternalSecret`.

Nesta instalação existem três workloads:

- `external-secrets`: controller principal, responsável pela reconciliação.
- `external-secrets-webhook`: webhook de validação e suporte ao provider webhook.
- `external-secrets-cert-controller`: gera e injeta certificados nos webhooks e mantém a configuração de validação funcional.
- `Secret/s-external-secrets-webhook`: Secret TLS interno usado pelo webhook; o nome é diferente do Service `external-secrets-webhook`.

Sem o Operator, os demais YAMLs não têm controller para processá-los.

### 2. Store — `k8s/01-store/`

O **Store** define **onde** e **como** os valores são buscados. O recurso principal deste projeto é:

- `ClusterSecretStore/vaultwarden-api`
- provider: `webhook`
- URL: `http://svc-vaultwarden-api.vaultwarden.svc.cluster.local:8080/secret/{{ .remoteRef.key }}?collection_name={{ .remoteRef.property }}`
- coleção do Vaultwarden: informada em `remoteRef.property`
- autenticação: `Authorization: Bearer <API_KEY>`
- extração: `$.value`

O Secret de credenciais é lido pelo Store, não pela aplicação:

```text
Secret/s-vaultwarden-api-webhook-credentials (namespace `external-secrets`)
└── api-key: <API_KEY do vaultwarden-api>
```

O Secret precisa conter o label `external-secrets.io/type: webhook`.

### 3. ExternalSecret — `k8s/02-examples/`

O **ExternalSecret** é a declaração da aplicação. Ele informa:

- qual Store usar;
- qual item buscar no Vaultwarden (`remoteRef.key`);
- em qual coleção buscar (`remoteRef.property`);
- qual chave criar no Secret Kubernetes (`secretKey`);
- qual Secret deve ser gerado (`target.name`);
- com que frequência sincronizar (`refreshInterval`).

Em uma frase: **o Operator executa, o Store define a origem e o ExternalSecret define o contrato de consumo.**

Para buscar itens de uma coleção específica, informe o nome dela em `remoteRef.property`. Por exemplo, no namespace `argus`, use `secretStoreRef: {name: vaultwarden-api, kind: ClusterSecretStore}` e defina `property: Argus` em cada `remoteRef`. A API key fica somente no Secret `s-vaultwarden-api-webhook-credentials` do namespace `external-secrets`; aplicações não precisam de uma cópia local.

Como o `ClusterSecretStore` é compartilhado, qualquer `ExternalSecret` autorizado a usá-lo pode consultar itens que a API key consegue ler. Restrinja o acesso ao recurso e os itens disponíveis conforme o modelo de confiança do cluster.

---

## Arquitetura

```mermaid
flowchart LR
    subgraph VWNS["namespace: vaultwarden"]
        VW["Vaultwarden\ncofre"]
        API["vaultwarden-api\nsvc-vaultwarden-api:8080"]
        VW <-->|"login e cache em memória"| API
    end

    subgraph ESONS["namespace: external-secrets"]
        CTRL["ESO controller"]
        WEBHOOK["ESO webhook"]
        CERT["ESO cert-controller"]
        STORE["ClusterSecretStore\nvaultwarden-api"]
        CREDS["Secret de credenciais\napi-key"]
        CTRL --> STORE
        CERT --> WEBHOOK
        STORE -.-> CREDS
    end

    subgraph APPNS["namespace da aplicação"]
        ES["ExternalSecret"]
        KSEC["Secret Kubernetes\n gerado"]
        APP["Deployment / Pod"]
        ES --> KSEC
        APP -->|"envFrom, env ou volume"| KSEC
    end

    ES -->|"secretStoreRef"| STORE
    STORE -->|"GET /secret/<key>\nBearer token"| API

    classDef backend fill:#175DDC,stroke:#0f3f96,color:#fff;
    classDef operator fill:#5B4EE9,stroke:#3b32a6,color:#fff;
    classDef app fill:#198754,stroke:#12653f,color:#fff;
    class VW,API backend;
    class CTRL,WEBHOOK,CERT,STORE,CREDS operator;
    class ES,KSEC,APP app;
```

### Fluxo de sincronização

```mermaid
sequenceDiagram
    participant ES as ExternalSecret
    participant CTRL as ESO controller
    participant STORE as ClusterSecretStore
    participant API as vaultwarden-api
    participant VW as Vaultwarden
    participant KSEC as Secret Kubernetes

    ES->>CTRL: reconcile conforme refreshInterval
    CTRL->>STORE: resolve provider e credenciais
    STORE->>API: GET /secret/{remoteRef.key}
    Note over STORE,API: Authorization: Bearer <API_KEY>
    API->>VW: consulta item no cofre/cache
    VW-->>API: valor descriptografado
    API-->>STORE: 200 { name, value }
    STORE-->>CTRL: extrai $.value
    CTRL->>KSEC: cria ou atualiza Secret
    Note over KSEC: Reloader pode reiniciar o workload consumidor
```

---

## Pré-requisitos

| Requisito | Valor / observação |
|---|---|
| Kubernetes | `>= 1.19`; ambiente validado: `Talos-K0S-Cluster`. |
| `kubectl` | Configurado para o contexto correto e com permissão de escrita. |
| Vaultwarden | Em execução no namespace `vaultwarden`. |
| `vaultwarden-api` | Service `svc-vaultwarden-api.vaultwarden.svc.cluster.local:8080`. |
| API key | Secret `s-vaultwarden-api-credentials`, chave `API_KEY`, no namespace `vaultwarden`. |
| Nodes de destino | Label `apps=services`. |
| Taint tolerado | `servicesonly=true:PreferNoSchedule`. |
| Reloader | Opcional; já existente no cluster e usado pelo exemplo de Deployment. |

Valide o contexto antes de aplicar qualquer alteração:

```bash
kubectl config current-context
kubectl get nodes -l apps=services
kubectl get pods -n vaultwarden
kubectl get svc svc-vaultwarden-api -n vaultwarden
```

---

## Quick start

A instalação é feita em três etapas. Execute na ordem e não aplique o arquivo de credenciais com o placeholder.

### Etapa 1 — Operator

```bash
kubectl apply -f k8s/00-operator/01-eso-operator-namespace.yaml

# Server-side apply é obrigatório para as CRDs grandes do ESO.
# O apply client-side pode exceder o limite da anotação
# last-applied-configuration e deixar CRDs de Store sem criar.
kubectl apply --server-side --force-conflicts \
  -f k8s/00-operator/02-eso-operator-install.yaml

kubectl -n external-secrets rollout status deploy/external-secrets
kubectl -n external-secrets rollout status deploy/external-secrets-webhook
kubectl -n external-secrets rollout status deploy/external-secrets-cert-controller
```

**Critério de sucesso:** os três Deployments devem estar disponíveis:

```bash
kubectl get deploy -n external-secrets
kubectl get crd | grep external-secrets.io
```

### Etapa 2 — Credenciais e Store

O comando abaixo copia a `API_KEY` já existente no cluster para o Secret consumido pelo `ClusterSecretStore`. O valor não é impresso nem gravado no repositório:

```bash
API_KEY="$(kubectl get secret s-vaultwarden-api-credentials \
  -n vaultwarden -o jsonpath='{.data.API_KEY}' | base64 -d)"

test -n "$API_KEY" || {
  echo "API_KEY não encontrada" >&2
  exit 1
}

kubectl create secret generic s-vaultwarden-api-webhook-credentials \
  -n external-secrets \
  --from-literal=api-key="$API_KEY" \
  --dry-run=client -o yaml | \
  kubectl label --local -f - \
    external-secrets.io/type=webhook -o yaml | \
  kubectl apply -f -

kubectl apply -f \
  k8s/01-store/02-eso-store-clustersecretstore-vaultwarden.yaml
```

**Critérios de sucesso:**

```bash
kubectl get secret s-vaultwarden-api-webhook-credentials -n external-secrets
kubectl get secret s-vaultwarden-api-webhook-credentials -n external-secrets -o json \
  | jq '{namespace: .metadata.namespace, type: .metadata.labels["external-secrets.io/type"], keys: (.data | keys)}'
kubectl get clustersecretstore vaultwarden-api
kubectl describe clustersecretstore vaultwarden-api
```

O Secret deve estar em `external-secrets`, ter o label `external-secrets.io/type: webhook` e conter a chave `api-key`. O Store deve apresentar condição `Ready=True`.

> `k8s/01-store/01-eso-store-secret-webhook-credentials.yaml` é apenas um modelo com placeholder. Não aplique esse arquivo diretamente em produção.

### Etapa 3 — Exemplos de consumo

Primeiro crie no Vaultwarden os itens usados pelos exemplos. Depois aplique os arquivos na ordem:

```bash
kubectl apply -f k8s/02-examples/01-eso-examples-namespace.yaml
kubectl apply -f k8s/02-examples/02-eso-examples-externalsecret-single-key.yaml
kubectl apply -f k8s/02-examples/03-eso-examples-externalsecret-multi-key.yaml
kubectl apply -f k8s/02-examples/04-eso-examples-externalsecret-templated.yaml
kubectl apply -f k8s/02-examples/05-eso-examples-deployment-consumer.yaml
```

**Critérios de sucesso:**

```bash
kubectl get externalsecret -n eso-demo
kubectl get secret -n eso-demo app-database app-config app-dsn
kubectl get pods -n eso-demo
```

Os `ExternalSecrets` devem estar sincronizados e o pod `demo-consumer` deve estar `Running`.

---

## Estrutura do repositório

```text
external-secrets/
├── images/
│   └── eso-logo.png
├── k8s/
│   ├── 00-operator/
│   │   ├── 01-eso-operator-namespace.yaml
│   │   └── 02-eso-operator-install.yaml
│   ├── 01-store/
│   │   ├── 01-eso-store-secret-webhook-credentials.yaml
│   │   ├── 02-eso-store-clustersecretstore-vaultwarden.yaml
│   │   └── 99-eso-store-secretstore-namespaced-example.yaml
│   └── 02-examples/
│       ├── 01-eso-examples-namespace.yaml
│       ├── 02-eso-examples-externalsecret-single-key.yaml
│       ├── 03-eso-examples-externalsecret-multi-key.yaml
│       ├── 04-eso-examples-externalsecret-templated.yaml
│       └── 05-eso-examples-deployment-consumer.yaml
├── LICENSE
└── README.md
```

### Inventário de manifestos

| Arquivo | Tipo | Escopo | Finalidade |
|---|---|---|---|
| `01-eso-operator-namespace.yaml` | `Namespace` | Cluster | Cria `external-secrets`. |
| `02-eso-operator-install.yaml` | Bundle ESO | Cluster | CRDs, controllers, RBAC, Services e webhooks. |
| `01-eso-store-secret-webhook-credentials.yaml` | `Secret` | `external-secrets` | Modelo da API key; não aplicar com placeholder. |
| `02-eso-store-clustersecretstore-vaultwarden.yaml` | `ClusterSecretStore` | Cluster | Configura o provider webhook para o vaultwarden-api. |
| `99-eso-store-secretstore-namespaced-example.yaml` | `SecretStore` + `Secret` | Namespace | Modelo opcional de isolamento por namespace. |
| `01-eso-examples-namespace.yaml` | `Namespace` | Cluster | Cria o namespace `eso-demo`. |
| `02-eso-examples-externalsecret-single-key.yaml` | `ExternalSecret` | `eso-demo` | Um item externo para uma chave Kubernetes. |
| `03-eso-examples-externalsecret-multi-key.yaml` | `ExternalSecret` | `eso-demo` | Vários itens em um único Secret. |
| `04-eso-examples-externalsecret-templated.yaml` | `ExternalSecret` | `eso-demo` | Template de DSN e arquivo de configuração. |
| `05-eso-examples-deployment-consumer.yaml` | `Deployment` | `eso-demo` | Demonstra o consumo via `envFrom`. |

---

## Como cadastrar segredos no Vaultwarden

O `vaultwarden-api` lê itens de login comuns do Vaultwarden. Para cada segredo, crie um item com:

| Campo do item | Conteúdo |
|---|---|
| **Name** | O mesmo valor usado em `remoteRef.key`, por exemplo `DATABASE_URL`. |
| **Password** | O valor secreto que será sincronizado. |

A API prioriza os valores nesta ordem: `password` → campo personalizado → `notes`. A correspondência do nome é case-insensitive.

### Itens usados pelos exemplos

| Item no Vaultwarden | Usado por |
|---|---|
| `DATABASE_URL` | `externalsecret-single-key` |
| `APP_DB_USER` | `externalsecret-multi-key` |
| `APP_DB_PASSWORD` | `externalsecret-multi-key` |
| `APP_JWT_SECRET` | `externalsecret-multi-key` |
| `PG_USER` | `externalsecret-templated` |
| `PG_PASSWORD` | `externalsecret-templated` |
| `PG_HOST` | `externalsecret-templated` |
| `PG_DBNAME` | `externalsecret-templated` |

> **Recomendação:** use uma conta dedicada no Vaultwarden, com apenas os itens necessários para as aplicações. Evite usar uma conta pessoal com acesso ao cofre inteiro.

O `vaultwarden-api` também suporta filtros de organização, coleção e pasta. Exemplo:

```yaml
remoteRef:
  key: "DATABASE_URL?collection_name=Project1"
```

Para evitar ambiguidades, prefira nomes exatos e habilite `STRICT_SECRET_MATCH=true` no `vaultwarden-api` quando o comportamento desejado for falhar em qualquer correspondência parcial.

---

## Referência dos recursos

### `ClusterSecretStore` usado neste projeto

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: vaultwarden-api
spec:
  provider:
    webhook:
      url: "http://svc-vaultwarden-api.vaultwarden.svc.cluster.local:8080/secret/{{ .remoteRef.key }}"
      method: GET
      timeout: 10s
      result:
        jsonPath: "$.value"
      headers:
        Accept: "application/json"
        Authorization: "Bearer {{ index .creds \"api-key\" }}"
      secrets:
        - name: creds
          secretRef:
            name: s-vaultwarden-api-webhook-credentials
            namespace: external-secrets
```

Pontos importantes:

- O host da URL é literal; apenas o caminho que contém `remoteRef.key` é templatizado.
- `api-key` tem hífen, portanto o template usa `index .creds "api-key"`.
- O Secret referenciado pelo webhook precisa do label `external-secrets.io/type: webhook`.
- Resposta HTTP 404 indica ao ESO que o segredo remoto não foi encontrado; o comportamento final depende da `deletionPolicy` do `ExternalSecret`.
- O `ClusterSecretStore` é global. Se quiser limitar o escopo, use o modelo `99-eso-store-secretstore-namespaced-example.yaml`.

### Padrões de `ExternalSecret`

| Arquivo | Padrão |
|---|---|
| `02-eso-examples-externalsecret-single-key.yaml` | Um item do Vaultwarden para uma chave do Secret. |
| `03-eso-examples-externalsecret-multi-key.yaml` | Vários itens para um único Secret, adequado para `envFrom`. |
| `04-eso-examples-externalsecret-templated.yaml` | Composição de connection string e arquivo de configuração. |

Consumo típico pela aplicação:

```yaml
envFrom:
  - secretRef:
      name: app-config
```

---

## Segurança

### Modelo de proteção

| Ativo | Proteção aplicada | Responsabilidade operacional |
|---|---|---|
| `API_KEY` | Não é versionada; copiada em runtime para o Secret do Store. | Proteger RBAC e acesso ao namespace `external-secrets`. |
| Valor dos segredos | Não é impresso pelo fluxo documentado nem gravado em arquivos do repositório. | Não usar `kubectl get secret -o yaml` em logs compartilhados. |
| Endpoint da API | Service `ClusterIP` e DNS interno. | Não expor o Service sem autenticação e TLS adequados. |
| Acesso ao cofre | Conta dedicada recomendada. | Restringir itens por organização/coleção quando possível. |
| Comunicação interna | HTTP dentro do cluster nesta configuração. | Migrar para HTTPS + `caProvider` se houver requisito de criptografia em trânsito. |

### Boas práticas obrigatórias

- Nunca substituir o placeholder por uma chave real e fazer commit.
- Restringir quem pode ler Secrets no namespace `external-secrets`.
- Usar uma API key dedicada e rotacionável para o ESO.
- Preferir API keys escopadas no `vaultwarden-api` quando a instalação suportar esse modelo.
- Não habilitar dumps HTTP, `GODEBUG=http2debug=2` ou proxies de debug em produção.
- Avaliar `refreshInterval` conforme criticidade e carga do backend.
- Usar `readOnlyRootFilesystem`, `runAsNonRoot`, `seccompProfile` e `drop: [ALL]` nos workloads consumidores.

---

## Operação

### Verificar saúde do stack

```bash
kubectl get pods -n external-secrets -o wide
kubectl get deploy -n external-secrets
kubectl get validatingwebhookconfiguration | grep -E 'externalsecret|secretstore'
kubectl get crd | grep external-secrets.io
```

Os três Deployments esperados são:

```text
external-secrets
external-secrets-webhook
external-secrets-cert-controller
```

### Forçar sincronização

```bash
kubectl -n eso-demo annotate externalsecret app-config \
  force-sync="$(date +%s)" --overwrite
```

### Inspecionar eventos e condições

```bash
kubectl describe clustersecretstore vaultwarden-api
kubectl -n eso-demo describe externalsecret app-config
kubectl -n eso-demo get events --sort-by=.lastTimestamp
```

### Rotacionar a API key

1. Gere ou altere a API key no `vaultwarden-api`.
2. Atualize o Secret de credenciais sem imprimir o valor:

   ```bash
   API_KEY="$(kubectl get secret s-vaultwarden-api-credentials \
     -n vaultwarden -o jsonpath='{.data.API_KEY}' | base64 -d)"

   kubectl create secret generic s-vaultwarden-api-webhook-credentials \
     -n external-secrets --from-literal=api-key="$API_KEY" \
     --dry-run=client -o yaml | \
     kubectl label --local -f - external-secrets.io/type=webhook -o yaml | \
     kubectl apply -f -
   ```

3. Confirme `Ready=True` no `ClusterSecretStore`.
4. Force um refresh de um `ExternalSecret` se precisar validar imediatamente.

### Logs

```bash
kubectl -n external-secrets logs deploy/external-secrets --tail=100
kubectl -n external-secrets logs deploy/external-secrets-webhook --tail=100
kubectl -n external-secrets logs deploy/external-secrets-cert-controller --tail=100
```

O provider webhook evita registrar request/response completos porque eles podem conter credenciais. Use logs do `vaultwarden-api` para diagnosticar lookups e falhas de autenticação, sem habilitar logging de valores secretos.

### Renomear a Secret TLS do webhook

A Secret TLS interna do webhook usa o nome `s-external-secrets-webhook`. O nome do Service continua sendo `external-secrets-webhook`; são recursos diferentes e o Service não deve ser renomeado.

O bundle atualizado não remove automaticamente a Secret antiga. Para fazer a migração:

```bash
kubectl apply --server-side --force-conflicts \
  -f k8s/00-operator/02-eso-operator-install.yaml

kubectl -n external-secrets rollout status \
  deploy/external-secrets-cert-controller
kubectl -n external-secrets rollout status \
  deploy/external-secrets-webhook

kubectl get secret s-external-secrets-webhook -n external-secrets
kubectl get pods -n external-secrets
```

Somente depois de confirmar que o webhook está `Ready` e que as validações estão funcionando, remova a Secret antiga, caso ela ainda exista:

```bash
kubectl delete secret external-secrets-webhook -n external-secrets
```

> Não delete a Secret antiga antes de o novo Deployment estar usando `s-external-secrets-webhook`. O novo nome é referenciado pelo cert-controller (`--secret-name`) e pelo volume TLS do webhook (`secretName`).

### Logs

### Atualizar o ESO

O arquivo `02-eso-operator-install.yaml` é um bundle estático baseado no chart v0.20.2. Para atualizar:

1. Escolha e registre a nova versão.
2. Renderize o chart em um ambiente de build.
3. Preserve as customizações locais:
   - `affinity.nodeAffinity` para `apps=services`;
   - toleration `servicesonly=true:PreferNoSchedule`;
   - remoção de `--enable-partial-cache=true` no cert-controller;
   - recursos e security contexts adotados neste bundle.
4. Revise o diff, especialmente CRDs, RBAC, webhooks e imagens.
5. Aplique com server-side apply:

```bash
kubectl apply --server-side --force-conflicts \
  -f k8s/00-operator/02-eso-operator-install.yaml
```

> Helm pode ser utilizado somente para gerar o YAML estático durante o processo de manutenção. Helm não é instalado nem executado dentro do cluster por este projeto.

### Rollback operacional

Não remova CRDs em um rollback comum: isso pode apagar ou invalidar recursos `ExternalSecret` existentes. O procedimento seguro é:

1. manter o bundle anterior versionado;
2. reaplicar o `02-eso-operator-install.yaml` anterior com server-side apply;
3. aguardar os três rollouts;
4. verificar `ClusterSecretStore` e `ExternalSecrets`;
5. remover manualmente recursos somente após avaliar impacto e retenção dos Secrets gerados.

---

## Troubleshooting

Comece sempre por estas três verificações:

```bash
kubectl get pods -n external-secrets
kubectl describe clustersecretstore vaultwarden-api
kubectl -n <namespace> describe externalsecret <nome>
```

| Sintoma | Causa provável | Ação recomendada |
|---|---|---|
| Store `Ready=False` / `InvalidProviderConfig` | URL, credencial, label ou schema inválido. | Inspecione `describe` e confirme o Secret rotulado `external-secrets.io/type: webhook`. |
| `secret does not contain needed label 'external-secrets.io/type: webhook'` | Secret usado pelo webhook não tem o label obrigatório. | Adicione o label e reaplique o Secret. |
| `ExternalSecret` com `SecretSyncedError` e HTTP 401 | API key ausente, vazia ou divergente. | Recrie `s-vaultwarden-api-webhook-credentials` a partir de `s-vaultwarden-api-credentials`. |
| HTTP 404 no `ExternalSecret` | Item não existe, está na lixeira ou o nome está diferente. | Corrija o `remoteRef.key` ou crie o item no Vaultwarden. |
| HTTP 403 / IP bloqueado | `ALLOWED_IPS` não contempla a rede de pods. | Confirme o CIDR usado pelo cluster, por exemplo `10.244.0.0/16`. |
| HTTP 429 | Rate limit do `vaultwarden-api` atingido. | Aumente `refreshInterval` ou revise `RATE_LIMIT_MAX`. |
| `dial tcp :443: connect: connection refused` durante validação | Host foi colocado dentro de um template. | Mantenha o host literal e templatize somente path/query. |
| Store `ProviderNotFound` | CRDs ou controllers ausentes/não disponíveis. | Reaplique o Operator com server-side apply e confira os rollouts. |
| `cert-controller` `0/1`, `/readyz` 500, `crd-inject failed` | CRDs grandes de Store não foram criadas por apply client-side. | Use `kubectl apply --server-side --force-conflicts -f k8s/00-operator/02-eso-operator-install.yaml` e confirme `kubectl get crd | grep secretstores`. |
| `cert-controller` não agenda nos nodes esperados | Label ou taint dos nodes divergiu do bundle. | Confirme `kubectl get nodes -l apps=services` e o taint `servicesonly=true:PreferNoSchedule`. |
| Webhook usa Secret antiga ou falha ao montar `/tmp/certs` | Bundle foi atualizado parcialmente ou a Secret antiga foi removida antes do rollout. | Reaplique o bundle com server-side apply, aguarde os dois rollouts e só então remova `external-secrets-webhook`. |
| Valor do header inválido | Uso de `.creds.api-key`, que não é sintaxe válida para chave com hífen. | Use `{{ index .creds "api-key" }}`. |
| Secret Kubernetes não muda no pod | O Secret mudou, mas o workload não reinicia automaticamente. | Use Reloader ou faça rollout controlado da aplicação. |

### Diagnóstico de rede

Verifique DNS e saúde do endpoint a partir do cluster sem expor a API:

```bash
kubectl -n vaultwarden get svc svc-vaultwarden-api
kubectl -n vaultwarden get endpoints svc-vaultwarden-api
kubectl -n external-secrets run network-debug --rm -it --restart=Never \
  --image=curlimages/curl:8.10.1 -- \
  curl -fsS http://svc-vaultwarden-api.vaultwarden.svc.cluster.local:8080/health
```

O comando acima testa apenas `/health`, que não exige a API key. Nunca coloque a API key na linha de comando de um diagnóstico compartilhado.

---

## Decisões de implementação

| Decisão | Motivo |
|---|---|
| Provider `webhook` | O backend disponível é uma REST API customizada, não o `bitwarden-sdk-server`. |
| `ClusterSecretStore` | Permite que aplicações em múltiplos namespaces reutilizem a integração. |
| YAML estático | O cluster recebe apenas manifests Kubernetes; não existe dependência de Helm em runtime. |
| Server-side apply | As CRDs do ESO podem ultrapassar o limite da anotação client-side `last-applied-configuration`. |
| Node affinity obrigatória | Os controllers devem rodar em nodes com `apps=services`. |
| Sem partial cache no cert-controller | Evita que CRDs sem o label esperado fiquem invisíveis ao check `crd-inject`. |
| Secret de credenciais fora do Git | Evita persistir a API key no repositório. |

---

## Referências

- [External Secrets Operator](https://external-secrets.io/)
- [ESO — Webhook provider](https://external-secrets.io/latest/provider/webhook/)
- [ESO — Getting started](https://external-secrets.io/latest/introduction/getting-started/)
- [ESO — Controller options](https://external-secrets.io/latest/api/controller-options/)
- [Turbootzz/Vaultwarden-API](https://github.com/Turbootzz/Vaultwarden-API)
- [Vaultwarden](https://github.com/dani-garcia/vaultwarden)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes server-side apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/)

---

## Licença

Distribuído sob a licença MIT. Consulte [`LICENSE`](LICENSE) para os termos completos.
