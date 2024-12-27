# flux2-kustomize-helm-example

[![test](https://github.com/fluxcd/flux2-kustomize-helm-example/workflows/test/badge.svg)](https://github.com/fluxcd/flux2-kustomize-helm-example/actions)
[![e2e](https://github.com/fluxcd/flux2-kustomize-helm-example/workflows/e2e/badge.svg)](https://github.com/fluxcd/flux2-kustomize-helm-example/actions)
[![license](https://img.shields.io/github/license/fluxcd/flux2-kustomize-helm-example.svg)](https://github.com/fluxcd/flux2-kustomize-helm-example/blob/main/LICENSE)

この例では、ステージングとプロダクションの2つのクラスターを持つシナリオを想定しています。最終的な目標は、FluxとKustomizeを活用して、重複した宣言を最小限に抑えながら両方のクラスターを管理することです。

Fluxを設定して、`HelmRepository` と `HelmRelease` のカスタムリソースを使用してデモアプリをインストール、テスト、アップグレードします。FluxはHelmリポジトリを監視し、セマンティックバージョン範囲に基づいてHelmリリースを自動的に最新のチャートバージョンにアップグレードします。

## 前提条件

Kubernetesクラスターのバージョン1.28以上が必要です。ローカルでのクイックテストには、[Kubernetes kind](https://kind.sigs.k8s.io/docs/user/quick-start/)を使用できます。他のKubernetesセットアップでも動作します。

このガイドに従うには、GitHubアカウントとリポジトリを作成できる[個人アクセストークン](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)が必要です（`repo`のすべての権限をチェックしてください）。

macOSまたはLinuxでHomebrewを使用してFlux CLIをインストールします：

```sh
brew install fluxcd/tap/flux
```

または、Bashスクリプトを使用して事前コンパイル済みバイナリをダウンロードしてCLIをインストールします：

```sh
curl -s https://fluxcd.io/install.sh | sudo bash
```

## リポジトリ構造

Gitリポジトリには、次のトップディレクトリが含まれています：

- **apps** ディレクトリには、クラスターごとにカスタム設定されたHelmリリースが含まれています
- **infrastructure** ディレクトリには、ingress-nginxやcert-managerなどの共通インフラツールが含まれています
- **clusters** ディレクトリには、クラスターごとのFlux設定が含まれています

```
├── apps
│   ├── base
│   ├── production 
│   └── staging
├── infrastructure
│   ├── configs
│   └── controllers
└── clusters
    ├── production
    └── staging
```

### アプリケーション

アプリの設定は次のように構成されています：

- **apps/base/** ディレクトリには、名前空間とHelmリリース定義が含まれています
- **apps/production/** ディレクトリには、プロダクションのHelmリリース値が含まれています
- **apps/staging/** ディレクトリには、ステージングの値が含まれています

```
./apps/
├── base
│   └── podinfo
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       ├── release.yaml
│       └── repository.yaml
├── production
│   ├── kustomization.yaml
│   └── podinfo-patch.yaml
└── staging
    ├── kustomization.yaml
    └── podinfo-patch.yaml
```

**apps/base/podinfo/** ディレクトリには、両方のクラスターに共通の値を持つFlux `HelmRelease` があります：

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo
spec:
  releaseName: podinfo
  chart:
    spec:
      chart: podinfo
      sourceRef:
        kind: HelmRepository
        name: podinfo
        namespace: flux-system
  interval: 50m
  values:
    ingress:
      enabled: true
      className: nginx
```

**apps/staging/** ディレクトリには、ステージング固有の値を持つKustomizeパッチがあります：

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: podinfo
spec:
  chart:
    spec:
      version: ">=1.0.0-alpha"
  test:
    enable: true
  values:
    ingress:
      hosts:
        - host: podinfo.staging
```

`version: ">=1.0.0-alpha"` を使用して、Fluxが`HelmRelease`を自動的に最新のチャートバージョンにアップグレードするように設定しています（アルファ、ベータ、プレリリースを含む）。

**apps/production/** ディレクトリには、プロダクション固有の値を持つKustomizeパッチがあります：

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: podinfo
  namespace: podinfo
spec:
  chart:
    spec:
      version: ">=1.0.0"
  values:
    ingress:
      hosts:
        - host: podinfo.production
```

`version: ">=1.0.0"` を使用して、Fluxが`HelmRelease`を自動的に最新の安定版チャートバージョンにアップグレードするように設定しています（アルファ、ベータ、プレリリースは無視されます）。

### インフラストラクチャ

インフラストラクチャは次のように構成されています：

- **infrastructure/controllers/** ディレクトリには、Kubernetesコントローラーの名前空間とHelmリリース定義が含まれています
- **infrastructure/configs/** ディレクトリには、cert issuersやネットワークポリシーなどのKubernetesカスタムリソースが含まれています

```
./infrastructure/
├── configs
│   ├── cluster-issuers.yaml
│   └── kustomization.yaml
└── controllers
    ├── cert-manager.yaml
    ├── ingress-nginx.yaml
    └── kustomization.yaml
```

**infrastructure/controllers/** ディレクトリには、Flux `HelmRepository` と `HelmRelease` の定義が含まれています：

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cert-manager
  namespace: cert-manager
spec:
  interval: 30m
  chart:
    spec:
      chart: cert-manager
      version: "1.x"
      sourceRef:
        kind: HelmRepository
        name: cert-manager
        namespace: cert-manager
      interval: 12h
  values:
    installCRDs: true
```

`interval: 12h` を使用して、FluxがHelmリポジトリインデックスを12時間ごとにプルして更新を確認するように設定しています。`1.x` セマンティックバージョン範囲に一致する新しいチャートバージョンが見つかった場合、Fluxはリリースをアップグレードします。

**infrastructure/configs/** ディレクトリには、Let's Encrypt issuerなどのKubernetesカスタムリソースが含まれています：

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    # 自分の連絡用メールアドレスに置き換えてください
    email: fluxcdbot@users.noreply.github.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-nginx
    solvers:
      - http01:
          ingress:
            class: nginx
```

**clusters/production/infrastructure.yaml** では、Let's Encryptサーバーの値をプロダクションAPIに置き換えます：

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infra-configs
  namespace: flux-system
spec:
  # ...省略
  dependsOn:
    - name: infra-controllers
  patches:
    - patch: |
        - op: replace
          path: /spec/acme/server
          value: https://acme-v02.api.letsencrypt.org/directory
      target:
        kind: ClusterIssuer
        name: letsencrypt
```

`dependsOn` を使用して、Fluxが最初にコントローラーをインストールまたはアップグレードし、その後に設定を行うように指示しています。これにより、Kubernetes CRDがクラスターに登録されてから、Fluxがカスタムリソースを適用することが保証されます。

## ステージングとプロダクションのブートストラップ

クラスターのディレクトリには、Fluxの設定が含まれています：

```
./clusters/
├── production
│   ├── apps.yaml
│   └── infrastructure.yaml
└── staging
    ├── apps.yaml
    └── infrastructure.yaml
```

**clusters/staging/** ディレクトリには、FluxのKustomization定義があります。例えば：

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m0s
  dependsOn:
    - name: infra-configs
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/staging
  prune: true
  wait: true
```

`path: ./apps/staging` を使用して、FluxがステージングのKustomizeオーバーレイを同期するように設定し、`dependsOn` を使用して、アプリをデプロイする前にインフラストラクチャ項目を作成するように指示しています。

このリポジトリを個人のGitHubアカウントにフォークし、GitHubアクセストークン、ユーザー名、リポジトリ名をエクスポートします：

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=<repository-name>
```

ステージングクラスターが前提条件を満たしていることを確認します：

```sh
flux check --pre
```

kubectlコンテキストをステージングクラスターに設定し、Fluxをブートストラップします：

```sh
flux bootstrap github \
    --context=staging \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/staging
```

ブートストラップコマンドは、`clusters/staging/flux-system` ディレクトリにFluxコンポーネントのマニフェストをコミットし、GitHubに読み取り専用アクセスのデプロイキーを作成して、クラスター内で変更をプルできるようにします。

ステージングでインストールされるHelmリリースを監視します：

```console
$ watch flux get helmreleases --all-namespaces

NAMESPACE     NAME          REVISION SUSPENDED READY MESSAGE 
cert-manager  cert-manager  v1.11.0  False     True  Release reconciliation succeeded
ingress-nginx ingress-nginx 4.4.2    False     True  Release reconciliation succeeded
podinfo       podinfo       6.3.0    False     True  Release reconciliation succeeded
```

デモアプリがingressを介してアクセス可能であることを確認します：

```console
$ kubectl -n ingress-nginx port-forward svc/ingress-nginx-controller 8080:80 &

$ curl -H "Host: podinfo.staging" http://localhost:8080
{
  "hostname": "podinfo-59489db7b5-lmwpn",
  "version": "6.2.3"
}
```

プロダクションクラスターのコンテキストとパスを設定して、Fluxをブートストラップします：

```sh
flux bootstrap github \
    --context=production \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/production
```

プロダクションの調整を監視します：

```console
$ flux get kustomizations --watch

NAME              REVISION      SUSPENDED READY MESSAGE                         
apps              main/696182e False     True  Applied revision: main/696182e 
flux-system       main/696182e False     True  Applied revision: main/696182e 
infra-configs     main/696182e False     True  Applied revision: main/696182e 
infra-controllers main/696182e False     True  Applied revision: main/696182e 
```

## クラスターの追加

フリートにクラスターを追加したい場合は、まずローカルにリポジトリをクローンします：

```sh
git clone https://github.com/${GITHUB_USER}/${GITHUB_REPO}.git
cd ${GITHUB_REPO}
```

`clusters` 内にクラスター名のディレクトリを作成します：

```sh
mkdir -p clusters/dev
```

ステージングから同期マニフェストをコピーします：

```sh
cp clusters/staging/infrastructure.yaml clusters/dev
cp clusters/staging/apps.yaml clusters/dev
```

`apps` 内にdevオーバーレイを作成し、`clusters/dev/apps.yaml` 内の `spec.path` を `path: ./apps/dev` に変更してください。

変更をメインブランチにプッシュします：

```sh
git add -A && git commit -m "add dev cluster" && git push
```

kubectlコンテキストとパスをdevクラスターに設定し、Fluxをブートストラップします：

```sh
flux bootstrap github \
    --context=dev \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/dev
```

## 同一環境

同一の環境を立ち上げたい場合は、`production-clone` などのクラスターをブートストラップし、`production` 定義を再利用できます。

`production-clone` クラスターをブートストラップします：

```sh
flux bootstrap github \
    --context=production-clone \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --path=clusters/production-clone
```

ローカルに変更をプルします：

```sh
git pull origin main
```

`clusters/production-clone` ディレクトリ内に `kustomization.yaml` を作成します：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - flux-system
  - ../production/infrastructure.yaml
  - ../production/apps.yaml
```

`flux-system` のkustomizeオーバーレイに加えて、`production` ディレクトリから `infrastructure` と `apps` のマニフェストも含めています。

変更をメインブランチにプッシュします：

```sh
git add -A && git commit -m "add production clone" && git push
```

Fluxに `production-clone` クラスターでプロダクションワークロードをデプロイするように指示します：

```sh
flux reconcile kustomization flux-system \
    --context=production-clone \
    --with-source 
```

## テスト

Kubernetesマニフェストやリポジトリ構造への変更は、プルリクエストがメインブランチにマージされ、クラスターで同期される前にCIで検証されるべきです。

このリポジトリには、次のGitHub CIワークフローが含まれています：

- [test](./.github/workflows/test.yaml) ワークフローは、[kubeconform](https://github.com/yannh/kubeconform)を使用してKubernetesマニフェストとKustomizeオーバーレイを検証します
- [e2e](./.github/workflows/e2e.yaml) ワークフローは、CIでKubernetesクラスターを起動し、Kubernetes KindでFluxを実行してステージングセットアップをテストします
