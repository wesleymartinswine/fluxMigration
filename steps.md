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

- image-reflector-controller: reponsável por ver qual tag é a mais recente;
- image-automation-controller: reponsável por impor as policies (políticas de imagem)
- Esses dois controllers não são instalados por default, devem ser instalados junto com o bootstrap,


### Criando um objeto de automação:

[Neste](https://toolkit.fluxcd.io/guides/flux-v1-automation-migration/#making-an-automation-object) link explica como criar esse objeto.

OBS: nao consegui criar um objeto ImageUpdateAutomation -> ainda nao existe as flags que eles disponibilizaram no comando,
acredito que seja porque essa parte ainda (de automação de imagem) ainda está em construção.

### migrando cada manifesto:

Antes a automação de imagem era dada por annotations no deploy da aplicação:

```
  annotations:
    fluxcd.io/automated: "true"
    fluxcd.io/tag.app: semver:^5.0
```

Agora existe um objeto separado para isso: imagePolicy. O seguinte comando cria o seu manifesto no caminho especificado:

```
flux create image repository podinfo \
--image=ghcr.io/stefanprodan/podinfo \
--interval=1m \
--export > ./kubernetes/clusters/bifrost/releases/automation/podinfo-registry.yaml
```
Esse comando vai criar um manifesto no caminho especificado.

Após isso commitar mudanças, sincronizar o flux (com flux reconcile) e checar se está funcionando:

`flux reconcile kustomization --with-source flux-system`

[Aqui](https://toolkit.fluxcd.io/guides/flux-v1-automation-migration/#how-to-use-sortable-image-tags) vc consegue ler melhor sobre os prefixos utilizados no flux1 e o que utilizar no flux2.

O seguinte comando cria os manifestos da image Policy:

```
flux create image policy my-app-policy \
    --image-ref podinfo-image \
    --semver '^5.0' \
    --export > ./$AUTO_PATH/my-app-policy.yaml
```
verificar:

`flux get image policy flux-system`

### Troubleshooting
- Caso algo não funcione, por esse comando vc olha os logs:

`kubectl logs -n flux-system deploy/image-automation-controller`
