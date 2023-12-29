---
title: "Cloud Run のマルチコンテナで実現する Envoy & Open Policy Agentによる認証認可"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GoogleCloud","CloudRun","Envoy","opa","Keycloak"]
published: true
---

## はじめに

今回は先日紹介したCloud Run のマルチコンテナデプロイ機能に関連してサイドカーコンテナとして Envoy と Open Policy Agent (以下、OPA)
をデプロイして認証認可を行う方法を紹介します。
また、認証については今回は Keycloak を使って Open ID Connect (以下、OIDC) による認証を行います。

Cloud Run のマルチコンテナデプロイ機能については以下の記事で紹介しています。
現在ご覧いただいている記事執筆時点においてもマルチコンテナデプロイ機能は Public Preview なのでご注意ください。

https://zenn.dev/cloud_ace/articles/036f6056a620fe

## 検証環境の構成

今回紹介させていただく内容は以下の構成で検証しています。

![全体構成](/images/7de5d77ffbf350/system-achitecture.png)

上記、図上の右下の Cloud Run についてマルチコンテナでデプロイしています。
以下の順に処理を行っていきます。

1. ブラウザからフロントエンドへアクセス
2. フロントエンドから Keycloak にて認証
3. 認証後のフロントエンド画面において Backend API へアクセス
   ※リクエストにはKeycloakが発行した Json Web Token (以下、JWT)が設定されます。
4. Cloud Run 上の Envoy コンテナにてJWTを検証して認証されたリクエストかチェック
5. OPA コンテナにて Envoy コンテナより連携された JWT上のペイロードを基に Backend API へアクセス可能なリクエストかチェック
6. 上記 4. 及び 5. にてパスされたリクエストのみ Envoy から Backend API へルーティング
7. あいうえお

各構成要素について簡単に説明させていただきます。

### FrontendApp

今回は React のサンプルアプリをベースにKeycloakでの認証機能、API アクセス機能を追加しています。
Keyclokとの連携についてはkeycloak-jsという package を利用しています。
※フロントエンドの実装が苦手なので、たいしたコード書けていません。。
いやー本当苦手なんですよね。

https://www.npmjs.com/package/keycloak-js

:::message
今回は検証である為、Cloud Run へホスティングしていますが、実運用ではSPAであれば Cloud Storage等で静的 Web
サイトとしてCDNと組み合わせて公開する選択肢もあるので検討してください。Cloud Run のコールドスタートとコンテンツのダウンロードにかかる時間から応答時間が長期化する傾向があります。
:::

### Keycloak

Red Hat社 によって開発・運用されているOSSの Identity Providerです。
OpenID Connect, OAuth2.0, SAMLなどの認証プロトコルに対応しており、デフォルトで Google Authenticator を使ったOTP認証にも対応しています。
オンプレミスやクラウド、コンテナなど多くの展開オプションを持っています。
以前、大変お世話になり今回の検証でも利用しています。

https://www.keycloak.org/

今回は Google Compute Engine 上に Docker をインストールしてコンテナとしてデプロイしています。
Keycloakに検証用のユーザを登録して OIDC による認証を行います。ユーザにはロールを設定してこのロールを OPA で検証することで認可を行っていきます。
余談ですが、Keycloak は Terraform Provider が存在するので IaC による構築管理が可能なので、今回の検証でも利用しています。

https://registry.terraform.io/providers/mrparkers/keycloak/latest/docs

:::message
今回は検証である為、DBへの永続化や冗長化構成はしていません。
:::

### Envoy

Lyft社によって開発された後、Cloud Native Computing Foundation（以下、CNCF）へ寄贈されており、CNCFの中では現在、graduated（安定しており本番利用可）とされています。
Envoyはロードバランシング、サーキットブレーカー、今回紹介するような認証認可の機能などマイクロサービスアーキテクチャに特化しており、HTTP/1.1 に加えて、HTTP/2, gRPC
といったプロトコルに対応しています。
また、サービスメッシュである Istio でもEnvoyが使われています。

https://www.envoyproxy.io/

今回は Cloud Run のサイドカーコンテナとしてデプロイしています。
認証にあたるJWTの検証は JWT Authentication というフィルタ機能を利用していきます。

https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/jwt_authn_filter

OPA へ認可処理を委譲するために External Authorization というフィルタ機能を利用していきます。この機能によって認可処理を OPA
に限らず外部のサービスへ委譲することが可能になります。

https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_authz_filter

### Open Policy Agent

Styra社によって開発された後、CNCFへ寄贈されており現在では Envoy と同様、graduated となっています。
セキュリティポリシーやアクセス制御ポリシーを管理する用途で利用されており、Kubernetes のリソース変更におけるパラメータチェックや、今回紹介するようなマイクロサービスにおける認可制御で用いられます。
ポリシーは Rego という言語で実装する必要がありますが、その分柔軟な制御が可能です。

https://www.openpolicyagent.org/

今回は Envoy と同様にサイドカーコンテナとしてデプロイしています。
Envoy から OPA に連携される JWT に含まれるKeycloakに登録したユーザのロールをチェックすることで認可します。
また、OPAで認可を行う上でポリシーが書かれたファイルを渡してあげる必要がありますが、OPA の機能として Cloud Storage
に格納されたファイルを読み込むことが可能なので、こちらの機能を利用しています。ちなみに AWS の S3 や Azure Blob Storage もサポートしているようです。

https://www.openpolicyagent.org/docs/latest/management-bundles/#google-cloud-storage

### Backend API

冒頭で挙げたマルチコンテナデプロイ機能の紹介記事でも利用している簡単な AP になります。

## 検証におけるポイント

検証内容において最もポイントとなるところをピックアップして紹介させていただきます。

### Envoy の Config ファイル (envoy.yaml)

実際に利用した yaml ファイルにコメントでポイントを記載しています。
※デプロイ時において本ファイルはEnvoyのコンテナイメージに格納しています。

```
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 51051
    filter_chains:
    - filters:
# ---------- リクエストに対するルーティング定義 --------
      - name: envoy.filters.network.http_connection_manager 
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          access_log:
          - name: envoy.access_loggers.stdout
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: backend-app

# ---------- JWT Authentication のフィルタ定義 --------
          http_filters:
          - name: envoy.filters.http.jwt_authn
            typed_config:
              '@type': type.googleapis.com/envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication
              providers:
                keycloak:
                  issuer: https://{KEYCLOAK_HOST}/auth/realms/consumer
                  forward_payload_header: payload
                  remote_jwks:
                    http_uri:
                      uri: "https://{KEYCLOAK_HOST}/auth/realms/consumer/protocol/openid-connect/certs"
                      cluster: jwks
                      timeout: 5s
                    cache_duration: 600s
              rules:
                - match:
                    prefix: "/"
                  requires:
                    provider_name: keycloak
              bypass_cors_preflight: true

# ---------- External Authorization のフィルタ定義 --------

          - name: envoy.filters.http.ext_authz
            typed_config:
              '@type': type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthz
              transport_api_version: V3
              failure_mode_allow: false
              grpc_service:
                envoy_grpc:
                  cluster_name: opa
              with_request_body:
                max_request_bytes: 8192
                allow_partial_message: true

# ---------- フィルタ処理後のルーティング --------

          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
# -------- Backend APIのエンドポイント。Cloud Run上の別コンテナへ連携するので localhost を指定 --------
  - name: backend-app
    type: STRICT_DNS
    dns_lookup_family: V4_ONLY
    load_assignment:
      cluster_name: backend-app
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: localhost
                port_value: 8090

# -------- OPA のエンドポイント。Cloud Run上の別コンテナへ連携するので localhost を指定 --------
  - name: opa
    type: STRICT_DNS
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options: {}
    load_assignment:
      cluster_name: opa
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: localhost
                port_value: 9191

# -------- Keycloak のエンドポイント。Public にアクセスするので https の設定が必要。 --------
  - name: jwks
    type: STRICT_DNS
    load_assignment:
      cluster_name: jwks
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: {KEYCLOAK_HOST}
                    port_value: 443

# -------- https で通信するための定義。サーバ証明書の検証のみ。 --------
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        common_tls_context:
          validation_context:
            trusted_ca:
              filename: '/etc/ssl/certs/ca-certificates.crt'
```

### OPA の Config ファイル (opa.yaml)

OPAを起動する際に必要な Config ファイルになります。
上記で説明した Cloud Storage からポリシーファイルを読み込む設定もこちらで行います。
Cloud Run から Cloud Storage へアクセスするためのサービスアカウントが必要になります。
※本ファイルはOPAのコンテナイメージに格納しています。

```
plugins:
  envoy_ext_authz_grpc:
    addr: ":9191"
    enable-reflection: true
decision_logs:
  console: true
services:
  gcs:
    url: https://storage.googleapis.com/storage/v1/b/{BUCKET_NAME}/o
    credentials:
      gcp_metadata:
        scopes:
          - https://www.googleapis.com/auth/devstorage.read_only
bundles:
  authz:
    service: gcs
    # NOTE ?alt=media is required
    resource: 'bundle.tar.gz?alt=media'
```

### OPA の Policy ファイル (policy.rego)

認可を行うためのポリシーを実装しています。
今回の検証では /backend というパスに対して CORS による OPTIONS メソッドか GET メソッドにアクセスにおいて"extended"というロールを持つユーザのみパスするようにしています。

```

package envoy.authz

import input.attributes.request.http as http_request

default allow = false

allow {
    path
    method
}

path {
    http_request.path == "/backend"
}

method {
    http_request.method == "OPTIONS"
}

method {
    http_request.method == "GET"
    claims.resource_access.web.roles[_] == "extended"
}

claims = payload {
    payload := json.unmarshal(base64url.decode(http_request.headers.payload))
}

```

Cloud Storage にポリシーを格納する際は policy.rego というファイルを直接格納するのではなく、bundle というビルドされたファイルで格納する必要があります。
公式 doc でも説明されていますが、以下のコマンドでポリシーファイルの bundle 化が可能です。※tar.gzで出力されます。

https://www.openpolicyagent.org/docs/latest/management-bundles/#bundle-build

```
opa build -b {policyファイルが格納されたフォルダ}
```

### Cloud Run でのマルチコンテナデプロイ

上記で説明している Envoy, OPA, Backend API を一つの Cloud Run サービスとしてデプロイしていきます。
外部対してリッスンする ingress ポートは Envoy のポートとしています。
デプロイするための yaml ファイルは以下の通りです。

```
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: demo-sidecar-auth-app
  labels:
    cloud.googleapis.com/location: asia-northeast1
  annotations:
    run.googleapis.com/launch-stage: BETA
spec:
  template:
    metadata:
      annotations:
        run.googleapis.com/execution-environment: gen2
    spec:
      containerConcurrency: 80
      timeoutSeconds: 30
      containers:
        - image: asia-northeast1-docker.pkg.dev/{PROJECT_ID}/{REPOSITORY_NAME}/demo-sidecar-app:v0.0.5
          name: backend-app
          env:
            - name: APP_NAME
              value: 'backend'
            - name: APP_PORT
              value: '8090'
          resources:
            limits:
              cpu: 1000m
              memory: 256Mi
        - image: asia-northeast1-docker.pkg.dev/{PROJECT_ID}/{REPOSITORY_NAME}/envoy:1.26.1-u5
          name: envoy-proxy
          ports:
            containerPort: 51051
          args:
          - "-c"
          - "/etc/envoy/envoy.yaml"
          - "--service-cluster"
          - "front-proxy"
          resources:
            limits:
              cpu: 1000m
              memory: 256Mi
        - image: asia-northeast1-docker.pkg.dev/{PROJECT_ID}/{REPOSITORY_NAME}/opa:envoy-u2
          name: opa
          args:
          - "run"
          - "--server"
          - "--config-file=/opa.yaml"
          - "--log-level=debug"
          resources:
            limits:
              cpu: 1000m
              memory: 256Mi
  traffic:
    - percent: 100
      latestRevision: true
```

記事上での説明は割愛しますが、上記に加えて FrontendApp と Keycloak のセットアップすることで上記 Cloud Run にて未認証ユーザには ```401 Unauthorized```
を応答し、更に特定ロールを持たないユーザからのアクセスには ```403 Forbidden``` が応答されるようになります。

## まとめ

認証認可の処理についてEnvoy, OPA のコンテナを Cloud Run に適用することで API(今回ではBackend API) にて実装することなく実現する方法を紹介させていただきました。
マルチコンテナとすることで新たに API を追加した際においても同じコンテナをデプロイすることで認証認可の処理を横断的に展開することが可能です。もちろん認可の制御を変えたい場合は OPA
のポリシーファイルを個別に用意して適用することも可能です。

繰り返しにはなりますが、Cloud Run のマルチコンテナデプロイ機能は本記事執筆時点では Public Preview となりますのでご注意ください。

参考までに今回検証で実装した各種資材は以下のリポジトリにて公開しています。localブランチにおいては docker-compose.yaml
で各種コンテナを立ち上げてローカルで検証できるようになっています。

[GitHub Repository](https://github.com/cloud-ace/zenn-opa-envoy)