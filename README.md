# kubernetes実践

## ダッシュボードのインストール
ダッシュボードからKubernetesにデプロイされているコンテナ等を確認できるようにします。

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/alternative/kubernetes-dashboard.yaml
```

参考：[kubernetes dashboard](https://github.com/kubernetes/dashboard/wiki/Installation)

## deployされているか確認
```
$ kubectl get pod --namespace=kube-system -l k8s-app=kubernetes-dashboard

NAME                                    READY     STATUS    RESTARTS   AGE
kubernetes-dashboard-585d8849d6-9cscc   1/1       Running   0          1m
```
STATUSがRunningとなってればOK

## ダッシュボードを閲覧する為にプロキシサーバを立てる
```
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
```

`http://localhost:8001/api/v1/namespaces/kube-system/services/kubernetes-dashboard:/proxy/`
にアクセスするとダッシュボードが表示される

# Kubernetes architecture
Kubernetesの構成はこちら→[Kubernetes architecture](https://github.com/eBay/Kubernetes/blob/master/docs/design/architecture.md)


## node確認
```
kubectl get nodes
```

## ネームスペース確認

```
kubectl get namespace
```

# Podを作成してデプロイする

```simple-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-echo
spec:
  containers:
  - name: nginx
    image: gihyodocker/nginx-proxy:latest
    env:
    - name: BACKEND_HOST
      value: localhost:8080
    ports:
    - containerPort: 80
  - name: echo
    image: gihyodocker/echo:latest
    ports:
    - containerPort: 8080
```

```
$ kubectl apply -f simple-pod.yaml
pod "simple-echo" created
```

# podを操作する
```
$ # pod確認
$ kubectl get pod

$ # pod内でコマンド実行(-c でコンテナ指定。今回は、nginxを指定)
$ kubectl exec -it simple-echo sh -c nginx

$ # pod内のコンテナ(echo)の標準出力を確認
$ kubectl logs -f simple-echo -c echo

$ # podを削除
$ kubectl delete pod simple-echo

$ # マニフェストベースでpod削除
$ kubectl delete -f simple-pod.yaml
```

# ReplicaSet
Podの複数生成・管理する

```
$ # 複数podの作成
$ kubectl apply -f simple-replicaset.yaml
$ kubectl get pod

# マニフェストベースでpod削除
$ kubectl delete -f simple-replicaset.yaml
```

# Deployment

```simple-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  labels:
    app: echo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: nginx
        image: gihyodocker/nginx-proxy:latest
        env:
        - name: BACKEND_HOST
          value: localhost:8080
        ports:
        - containerPort: 80
      - name: echo
        image: gihyodocker/echo:latest
        ports:
        - containerPort: 8080

```


```
# Deploymentマニフェストファイルを実行
$ kubectl apply -f simple-deployment.yaml --record

$ # 確認
$ kubectl get pod,replicaset,deployment --selector app=echo

$ # リビジョン確認
$ kubrctl rollout history deployment echo
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=simple-deployment.yaml --record=true
```

## replica数を3→4に変更して実行

yamlを修正して、 `replicas: 3` →　`replicas: 4` 　にする。

```
$ kubectl apply -f simple-deployment.yaml --record

$ # 確認
$ kubectl get pod --selector app=echo
```
または、ダッシュボードなどでpodの数が４になっていることを確認


# コンテナの定義をecho:latest→echo:patchedに変更して実行

yamlを修正して、 `image: gihyodocker/echo:latest` →　`image: gihyodocker/echo:patched` 　にする。

```
kubectl apply -f simple-deployment.yaml --record
```
latestで作成されていたPodが削除され、新たにpatchedのpodが作成されていきます。

```
# リビジョン確認
$ kubrctl rollout history deployment echo

REVISION  CHANGE-CAUSE
1         kubectl apply --filename=simple-deployment.yaml --record=true
2         kubectl apply --filename=simple-deployment.yaml --record=true
```
リビジョンは、2になってます。


# ロールバックの実行(リビジョン１に戻す)
```
$ # 特定のリビジョンの内容確認
$ kubectl rollout history deployment echo --revision=1

deployments "echo" with revision #1
Pod Template:
  Labels:	app=echo
	pod-template-hash=1239334866
  Annotations:	kubernetes.io/change-cause=kubectl apply --filename=simple-deployment.yaml --record=true
  Containers:
   nginx:
    Image:	gihyodocker/nginx-proxy:latest
    Port:	80/TCP
    Host Port:	0/TCP
    Environment:
      BACKEND_HOST:	localhost:8080
    Mounts:	<none>
   echo:
    Image:	gihyodocker/echo:latest
    Port:	8080/TCP
    Host Port:	0/TCP
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```

```
$ # undoで直前のリビジョンにロールバックしてみる
$ kubectl rollout undo deployment echo
```


# deployment削除（関連するreplicasetとpodも削除）
```
kubectl delete -f simple-deployment.yaml
```

# Service
Serviceはクラスタ内に置いて、Podの集合（主にreplicaset）に対する経路や
サービスディスカバリ(クライアントから一貫した名前でアクセスできる仕組み)を提供する為のリソース。

```simple-replicaset-with-label.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: echo-spring
  labels:
    app: echo
    release: spring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
      release: spring
  template:
    metadata:
      labels:
        app: echo
        release: spring
    spec:
      containers:
      - name: nginx
        image: gihyodocker/nginx-proxy:latest
        env:
        - name: BACKEND_HOST
          value: localhost:8080
        ports:
        - containerPort: 80
      - name: echo
        image: gihyodocker/echo:patched
        ports:
        - containerPort: 8080

---
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: echo-summer
  labels:
    app: echo
    release: summer
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo
      release: summer
  template:
    metadata:
      labels:
        app: echo
        release: summer
    spec:
      containers:
      - name: nginx
        image: gihyodocker/nginx-proxy:latest
        env:
        - name: BACKEND_HOST
          value: localhost:8080
        ports:
        - containerPort: 80
      - name: echo
        image: gihyodocker/echo:patched
        ports:
        - containerPort: 8080

```

# マニフェストファイルをaaplyし作成されたPodを確認
```
$ # apply
$ kubectl apply -f simple-replicaset-with-label.yaml

$ # 確認(spring)
$ kubectl get pod -l app=echo -l release=spring
NAME                READY     STATUS    RESTARTS   AGE
echo-spring-tq9rs   2/2       Running   0          3m

$ # 確認(summer)
$ kubectl get pod -l app=echo -l release=summer
AME                READY     STATUS    RESTARTS   AGE
echo-summer-8ds9s   2/2       Running   0          4m
echo-summer-8spqp   2/2       Running   0          4m
```

# release=summerをもつpodのみにアクセスできるようなServiceを作成してみる

```simple-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  selector:
    app: echo
    release: summer
  ports:
    - name: http
      port: 80
```

```
$ # simple-service.yamlをapply
$ kubectl apply -f simple-service.yaml

$ #確認
$ kubectl get svc echo
NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
echo      ClusterIP   10.100.200.186   <none>        80/TCP    53s
```

# release=summerをもつPodのみにトラフィックが流れるか確認してみる
ServiceはKubernetesクラスタの中からしかアクセスできないので、
一時的なデバックコンテナをデプロイ、curlコマンドでHTTPリクエストを送って確認します。

```
$ kubectl run -i --rm --tty debug \
--image=gihyodocker/fundamental:0.1.0 \
--restart=Never bash -il

# curlでhttpリクエストを送信（複数回実行してみましょう）
debug:/# curl http://echo/
Hello Docker!!

# exitでログアウト
debug:/# exit
logout

$ # logの確認（release=summerにトラフィックが流れているか）
kubectl logs -f echo-summer-8ds9s -c echo
2018/10/12 07:35:42 start server
2018/10/12 07:35:42 image changed
2018/10/12 07:55:14 received request　←これ
:
:


$ kubectl logs -f echo-summer-8spqp -c echo
2018/10/12 07:35:45 start server
2018/10/12 07:35:45 image changed
2018/10/12 07:54:34 received request　←これ
:
:


$ kubectl logs -f echo-spring-tq9rs -c echo
2018/10/12 07:35:40 start server
2018/10/12 07:35:40 image changed
```
