# Day09-ingress
本日はIngressを用いて外部から内部のServiceにアクセスする方法を紹介します。

Ingressとは、クラスター内のServiceに外部からのアクセスを許可するルールを定義するAPIオブジェクトです。

IngressコントローラーはIngress内に設定されたルールを満たすように動作します。

## 事前作業
minikubeを起動します。
```
$ minikube start
```

NGINX Ingressコントローラーを有効化します。
```
$ minikube addons enable ingress
```

NGINX Ingressコントローラーの起動状態を確認します。
```
$ kubectl get pods -n kube-system
```

前回のDay08で扱った`loki-stack`を利用するため、コンテキストを設定します。
```
$ kubectl config set-context minikube --namespace loki
$ kubectl config use-context minikube
```
## Ingress概要
![](https://raw.githubusercontent.com/NakamuraYosuke/Day09-ingress/main/images/ingress-nodeport.png)
各物理ノードの NodePort に対してトラフィック振り分けを行なっています。

## NodePort Serviceの状態確認
前回の演習の最後に、下記のNodePortのServiceを作成しました。
```
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/instance: loki-stack-1618807512
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: grafana
    app.kubernetes.io/version: 6.7.0
    helm.sh/chart: grafana-5.7.10
  name: loki-stack-grafana-nodeport
  namespace: loki
spec:
  ports:
  - name: service
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app.kubernetes.io/instance: loki-stack-1618807512
    app.kubernetes.io/name: grafana
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```
この`loki-stack-grafana-nodeport` Serviceを確認します。
```
$ kubectl get svc loki-stack-grafana-nodeport
NAME                          TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
loki-stack-grafana-nodeport   NodePort   10.107.169.67   <none>        80:31970/TCP   7h43m
```

## Ingress作成
以下のマニフェストを`loki-stack-grafana-ingress.yaml`という名前で保存します。
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: loki-stack-grafana-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: minikube.grafana
      http:
        paths:
          - path: /
            backend:
              serviceName: loki-stack-grafana-nodeport
              servicePort: 80
```

`.spec.rules.host`に記載するホスト名称は任意の名前を指定してください。（世の中にないドメイン名で・・・）
`serviceName`には`NodePortのService名`を、`servicePort`には`NodePortのPort番号`を設定します。

保存し、Ingressリソースを作成します。
```
$ kubectl apply -f loki-stack-grafana-ingress.yaml
ingress.networking.k8s.io/loki-stack-grafana-ingress created
```

Ingressリソースを確認します。
```
$ kubectl get ingress
NAME                         CLASS    HOSTS              ADDRESS        PORTS   AGE
loki-stack-grafana-ingress   <none>   minikube.grafana   192.168.64.4   80      35m
```

HOSTS列に先ほどのマニフェストで指定したホスト名が、ADDRESS列にはminikubeのIPアドレスが設定されていることを確認します。

このホスト名はDNSに登録されているわけではありませんので、ブラウザからアクセス可能とするためhostsファイルに定義してあげます。

```
$ sudo vim /etc/hosts
```
以下を追記します。
```
192.168.64.4 minikube.grafana
```

ブラウザを起動し、アドレス欄に`http://minikube.grafana`と入力し、下記の画面が出れば成功です。

![](https://raw.githubusercontent.com/NakamuraYosuke/Day08-helm/main/images/login.png)

## TLSによるアクセス
TLSでアクセスするためには、鍵、証明書署名リクエスト、及び証明書を作成します。

まずは`loki-stack-grafana-ingress.key`という名前の鍵を作成します。
```
$ openssl genrsa -out loki-stack-grafana-ingress.key 2048
```

次に、`loki-stack-grafana-ingress.csr`という名前の証明書署名リクエストを作成します。

CNには、Ingressリソースに設定したホスト名を設定します。
```
$ openssl req -new -key loki-stack-grafana-ingress.key -out loki-stack-grafana-ingress.csr -subj "/CN=minikube.grafana"
```

最後に証明書を作成します。
上記で作成した鍵ファイルと証明書署名用リクエストを指定します。
```
$ openssl x509 -req -days 365 -in loki-stack-grafana-ingress.csr -signkey loki-stack-grafana-ingress.key -out loki-stack-grafana-ingress.crt
```

### TLS Secretの作成
上記で作成した証明書と鍵をもとに、Secretを作成します。
```
$ kubectl create secret tls loki-stack-grafana-ingress-tls --cert loki-stack-grafana-ingress.crt --key loki-stack-grafana-ingress.key
```

### Ingressマニフェスト修正
前段で作成した`loki-stack-grafana-ingress.yaml`を修正します。
以下のように、`.spec.tls`に上記で作成したSecretを定義します。
```
  tls:
  - secretName: loki-stack-grafana-ingress-tls
```

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: loki-stack-grafana-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - secretName: loki-stack-grafana-ingress-tls
  rules:
    - host: minikube.grafana
      http:
        paths:
          - path: /
            backend:
              serviceName: loki-stack-grafana-nodeport
              servicePort: 80
```

Ingressリソースを更新します。
```
$ kubectl apply -f loki-stack-grafana-ingress.yaml
ingress.networking.k8s.io/loki-stack-grafana-ingress configured
```

Ingressリソースを確認します。
```
$ kubectl get ingress                                 
NAME                         CLASS    HOSTS              ADDRESS        PORTS     AGE
loki-stack-grafana-ingress   <none>   minikube.grafana   192.168.64.4   80, 443   89s
```
443ポートでアクセスできるようになりました。

ブラウザを起動し、アドレス欄に`https://minikube.grafana`と入力しログイン画面が表示されればOKです。