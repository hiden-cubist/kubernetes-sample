# kubernetes sample

以下 docker-desktop for windows で動かす場合を想定

## hello-world

deployment/service/ingress のサンプル

### ingress controller デプロイ

ingress の動作に必要

```
> kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.48.1/deploy/static/provider/cloud/deploy.yaml
```

- https://kubernetes.io/ja/docs/concepts/services-networking/ingress/
- https://kubernetes.github.io/ingress-nginx/deploy/

### 動作確認

```
> kubectl apply -f hello-world/
> curl localhost/hello
Hello Kubernetes!
```

## kubernetes dashboard 導入

### dashboard デプロイ

```
> kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

- https://kubernetes.io/ja/docs/tasks/access-application-cluster/web-ui-dashboard/
- https://github.com/kubernetes/dashboard

### metrics server デプロイ

dashboard で cpu やメモリの使用率が表示されるようになる。（kubectl top pod 等も動作するようになる）

```
> kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

このままだと tls のエラーがでてしまう

```
> kubectl logs -n kube-system metrics-server-[pod-name]
-- 省略 --
x509: cannot validate certificate for 192.168.65.4 because it doesn't contain any IP SANs" node="docker-desktop"
```

そこで、deployment に一行追加する

```
> kubectl patch deployment metrics-server -n kube-system --type 'json' -p '[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

> kubectl get deployment -n kube-system metrics-server -o yaml
-- 省略 --
spec:
    containers:
    - args:
    - --cert-dir=/tmp
    - --secure-port=443
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --kubelet-use-node-status-port
    - --metric-resolution=15s
    # 以下の行が追加される
    - --kubelet-insecure-tls

# deploymentの再起動
> kubectl rollout -n kube-system restart deployment metrics-server
```

- https://github.com/kubernetes-sigs/metrics-server
- https://andrewlock.net/running-kubernetes-and-the-dashboard-with-docker-desktop/

### 動作確認

```
> kubectl proxy

# 以下にアクセスしてdashboardを起動
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

ここで token を入力する必要があり、本来は新しく cluster-admin ロール等を持つ service account を作成して token を取得する必要があるが、docker desktop の場合は、default の service account にも cluster-admin ロールがついている。

```
> kubectl get clusterrolebinding docker-for-desktop-binding -o yaml
-- 省略 --
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts
  namespace: kube-system
```

よって、default ネームスペースの default service account の token でもリソース情報の閲覧・変更等が可能。

```
> kubectl describe secrets
# 表示されるdefault-tokenをコピーして入力する
```

- https://github.com/docker/for-mac/issues/4774
- https://qiita.com/paper2/items/1cd3d6cf53cb2d085a60

## rbac

ユーザ作成と認証・認可

### ユーザ作成

```
# 秘密鍵作成
> openssl genrsa -out test-user.key 2048

# 証明書署名リクエスト作成
> openssl req -new -key test-user.key -out test-user.csr
Organization Name (eg, company) [Internet Widgits Pty Ltd]:test-group
Common Name (e.g. server FQDN or YOUR name) []:test-user

# base64エンコード
# certificate-signing-request.yamlのspec.requestを、以下で出力された値に置き換える
> cat test-user.csr | base64 | tr -d "\n"

# デプロイ
> kubectl apply -f certificate-signing-request.yaml

# 確認
> kubectl get csr
NAME        AGE   SIGNERNAME                            REQUESTOR            CONDITION
test-user   28s   kubernetes.io/kube-apiserver-client   docker-for-desktop   Pending

# 承認
> kubectl certificate approve test-user

# 確認
> kubectl get csr
NAME        AGE   SIGNERNAME                            REQUESTOR            CONDITION
test-user   94s   kubernetes.io/kube-apiserver-client   docker-for-desktop   Approved,Issued

# 証明書取得
> kubectl get csr test-user -o jsonpath='{.status.certificate}'| base64 -d > test-user.crt

# 資格情報のセット
> kubectl config set-credentials test-user --client-key=test-user.key --client-certificate=test-user.crt --embed-certs=true

# コンテキスト作成
> kubectl config set-context test-user --cluster=docker-desktop --user=test-user

# コンテキスト切り替え
> kubectl config use-context test-user

# 確認
# 認可設定をしていないのでpodへのアクセス権限がない
> kubectl get pods
Error from server (Forbidden): pods is forbidden: User "test-user" cannot list resource "pods" in API group "" in the namespace "default"
```

- https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/

### 認証・認可

```
# test-userにpodの読み取り権限の付与
> kubectl config use-context docker-desktop
> k apply -f rbac/role.yaml
> k apply -f rbac/role-binding.yaml

# 確認
> kubectl config use-context test-user
> kubectl get pods
No resources found in default namespace.
```

- https://kubernetes.io/ja/docs/reference/access-authn-authz/rbac/

## Redis を使用した PHP のゲストブックアプリケーション

frontend アプリケーション 3 + redis master 1 + redis slave 2 の冗長構成

### 動作確認

```
> kubectl apply -f guestbook/
# localhost:80でアプリケーションが起動する
```

- https://kubernetes.io/ja/docs/tutorials/stateless-application/guestbook/
