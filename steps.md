# FLUX V1 -> FLUX V2

## Instalação do Flux v1

`export GHUSER=YOURUSER`

`kubectl create ns flux`

```
fluxctl install \
--git-user=${GHUSER} \
--git-email=${GHUSER}@users.noreply.github.com \
--git-url=git@github.com:${GHUSER}/fluxMigration \
--git-path=namespaces,workloads \
--git-branch=main \
--namespace=flux
```

`kubectl -n flux rollout status deployment/flux`

`fluxctl identity --k8s-fwd-ns flux`

## install flux v2:

Com componentes extras para automação de imagem!

**repositório pessoal**
```
flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=$GITHUB_USER \
  --repository=flux-image-updates \
  --branch=main \
  --path=clusters/my-cluster \
  --token-auth \
  --personal
```

**repositório público**
```
flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=winecombr \
  --repository=devops \
  --branch=master \
  --path=./kubernetes/clusters/hmg-dev
```

> `flux get source git -A`

> `flux get kustomization -A`

## kustomizae controller

> `flux reconcile kustomization flux-system --with-source`

> `flux suspend kustomization flux-system`

> `flux resume kustomization flux-system`

ex de kustomization com healhChecks

https://toolkit.fluxcd.io/components/kustomize/kustomization/#health-assessment


O kustomization não vai ficar "ready" se não passar por esses healthChecks no `timeout` especificado


### Deploy de uma aplicação

Realizar o deployment de uma aplicação podinfo (por exemplo):

```
flux create source git podinfo \
  --url=https://github.com/stefanprodan/podinfo \
  --branch=master \
  --interval=30s \
  --export > ./kubernetes/clusters/bifrost/releases/podinfo-4/podinfo-source.yaml
```

```
flux create kustomization podinfo \
  --source=podinfo \
  --path="./kustomize" \
  --prune=true \
  --validation=client \
  --interval=1m \
  --export > ./kubernetes/clusters/bifrost/releases/podinfo-4/podinfo-kustomization.yaml
```

## install KUTOMIZE CLI 

```
curl --silent --location --remote-name \
 "https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize/v3.2.3/kustomize_kustomize.v3.2.3_linux_amd64" && \
 chmod a+x kustomize_kustomize.v3.2.3_linux_amd64 && \
 sudo mv kustomize_kustomize.v3.2.3_linux_amd64 /usr/local/bin/kustomize
```
## Gerar kutomization.yaml automaticamente

`kustomize create --autodetect --recursive`

Obs: É mais indicado criar um kustomization para cada aplicação, para meios de apresentação do flux2, serve.

- O seguinte comando utiliza o kustomization para sincronizar as mudanças, você pode rodá-lo ou esperar o tempo do fluz para sincronizar:

`flux reconcile kustomization flux-system --with-source`


O flux 2 será instalado e após isso deve realizar a migração gradual do helm-operator para o helm-controller através dos arquivos

exemplo da configuração anterior:


```
apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: podinfo-4
  namespace: app
  annotations:
    fluxcd.io/automated: "true"
    fluxcd.io/locked: "true"
spec:
  chart:
    repository: https://stefanprodan.github.io/podinfo
    name: podinfo
    version: 3.2.0 

```

No helm controller, você tem, além do helmRelease, o HelmRepository

```
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: podinfo-2
  namespace: app
spec:
  # The interval at which to reconcile the Helm release
  interval: 2m
  chart:
    spec:
      # The name of the chart as made available by the HelmRepository
      # (without any aliases)
      chart: podinfo
      # A fixed SemVer, or any SemVer range
      # (i.e. >=4.0.0 <5.0.0)
      version: 3.2.0
      # The reference to the HelmRepository
      sourceRef:
        kind: HelmRepository
        name: podinfo-h3
        # Optional, defaults to the namespace of the HelmRelease
        namespace: default
---
apiVersion: v1
kind: Secret
metadata:
  name: my-repository-creds
  namespace: default
data:
  # HTTP/S basic auth credentials
  username: <base64 encoded username>
  password: <base64 encoded password>
  # TLS credentials (certFile and keyFile, and/or caCert)
  certFile: <base64 encoded certificate>
  keyFile: <base64 encoded key>
  caCert: <base64 encoded CA certificate>
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: my-repository
  namespace: default
spec:
  # ...omitted for brevity
  secretRef:
    name: my-repository-creds
```
No Helm-operator o repositório Git era configurado da seguinte maneira:

```
---
apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: my-release
  namespace: default
spec:
  chart:
    # The URL of the Git repository
    git: https://example.com/org/repo
    # The Git branch (or other Git reference)
    ref: master
    # The path of the chart relative to the repository root
    path: ./charts/my-chart
```
Agora existe um recurso especial para isso: GitRepository

```
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: my-repository
  namespace: default
spec:
  # The interval at which to check the upstream for updates
  interval: 10m
  # The repository URL, can be a HTTP/S or SSH address
  url: https://example.com/org/repo
  # The Git reference to checkout and monitor for changes
  # (defaults to master)
  # For all available options, see:
  # https://toolkit.fluxcd.io/components/source/api/#source.toolkit.fluxcd.io/v1beta1.GitRepositoryRef
  ref:
    branch: master
---
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: my-release
  namespace: default
spec:
  # The interval at which to reconcile the Helm release
  interval: 10m
  chart:
    spec:
      # The path of the chart relative to the repository root
      chart: ./charts/my-chart
      # The reference to the GitRepository
      sourceRef:
        kind: GitRepository
        name: my-repository
        # Optional, defaults to the namespace of the HelmRelease
        namespace: default
```


## Migração da automação de image

### verificar pre-requisitos:

flux check --components-extra=image-reflector-controller,image-automation-controller

### Criando um objeto de automação:

[Neste](https://toolkit.fluxcd.io/guides/flux-v1-automation-migration/#making-an-automation-object) link explica como criar esse objeto.

flux create image update flux-system \
    --git-repo-ref=flux-system \
    --git-repo-path="./kubernetes/clusters/bifrost/releases/automation" \
    --checkout-branch=main \
    --push-branch=master \
    --author-name=fluxcdbot \
    --author-email=fluxcdbot@users.noreply.github.com \
    --commit-template="{{range .Updated.Images}}{{println .}}{{end}}" \
    --export > ./Documentos/github/beatrizafonso/fluxMigration/kubernetes/clusters/bifrost/releases/automation/my-app-auto.yaml


### migrando cada manifesto:

- imageRepository

FLUX 1:

```
  annotations:
    fluxcd.io/automated: "true"
    fluxcd.io/tag.app: semver:^5.0
```

FLUX 2

```
flux create image repository redis \
--image=ghcr.io/beatrizafonso/redis \
--interval=1m \
--export > ./Documentos/github/beatrizafonso/fluxMigration/kubernetes/clusters/bifrost/releases/automation/podinfo-registry.yaml
```

image Policy:

```
flux create image policy my-app-policy \
    --image-ref podinfo-image \
    --semver '^5.0' \
    --export > ./$AUTO_PATH/my-app-policy.yaml
```
verificar:

`flux get image policy flux-system`

## Notification-Controller

-criar um segredo com o webhook do seu canal:

`kubectl -n flux-system create secret generic slack-url \
--from-literal=address=https://discordapp.com/api/webhooks/825038165424996362/vwbJiy4ckG9_2aTkG_lQ3-xCEo0wTHzo7c2lWfGEwkhblGaSWpvlJNef-aUC-wc7_7EG`

- Criar um provedor e o alerta:

```
apiVersion: notification.toolkit.fluxcd.io/v1beta1
kind: Provider
metadata:
  name: discord
  namespace: flux-system
spec:
  type: discord
  channel: general
  secretRef:
    name: https://discordapp.com/api/webhooks/825038165424996362/vwbJiy4ckG9_2aTkG_lQ3-xCEo0wTHzo7c2lWfGEwkhblGaSWpvlJNef-aUC-wc7_7EG
---
apiVersion: notification.toolkit.fluxcd.io/v1beta1
kind: Alert
metadata:
  name: discord
  namespace: flux-system
spec:
  providerRef:
    name: discord
  eventSeverity: info
  eventSources:
    - kind: GitRepository
      name: '*'
    - kind: Kustomization
      name: '*'
```