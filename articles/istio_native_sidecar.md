---
title: "IstioのサイドカーコンテナをKubernetesのサイドカーコンテナ機能で起動する"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Kubernetes", "Istio"]
published: true
---

## はじめに
Kubernetes v1.29からサイドカーコンテナ機能が実装されました。これはメインコンテナとは別にロギングやプロキシのような周辺機能を追加するための機能です。

Istioでもネットワークプロキシとしてenvoyコンテナをメインコンテナとは別にインジェクションし、1つのPodに仕立て上げます。
しかしこれには問題があり、Jobを起動した際にメインコンテナが正常終了した後でもenvoyが終了せずにPodが残り続けてしまうといった事象がありました。

こういったIstio利用における問題点を解消するのにKubernetesネイティブなサイドカーコンテナ機能が役立ちます。

以降Kubernetesネイティブな機能でデプロイされるサイドカーを「ネイティブサイドカー」と呼び、従来のIstioのような形でデプロイされるサイドカーを単に「サイドカー」と呼びます。
ネイティブサイドカーについての詳細は以下をご覧ください。

https://kubernetes.io/ja/docs/concepts/workloads/pods/sidecar-containers/

https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/753-sidecar-containers/README.md

ここではKindを用いてローカルでenvoyのネイティブサイドカー化を検証してみます。

## Kindでクラスターを作成
今回は検証に使えれば問題ないので、クラスターは小さめで作成します。 以下の設定ファイルを使用します。
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: istio-k8s
networking:
  disableDefaultCNI: false
nodes:
- role: control-plane
- role: worker
- role: worker
```

これを使ってクラスターを作成します。
```shell
kind create cluster --config ./kind_config_1m2w.yaml
```

## Istioインストール ~ネイティブサイドカーを添えて~
Istioでネイティブサイドカーを使用するための設定は至って簡単で、コントロールプレーンとなるistiodに対して`ENABLE_NATIVE_SIDECARS=true`と環境変数を設定してあげるだけです。

今回はクイックにistioctlを使用して構築します。プロファイル設定も最小限ということでminimalを使用しますので、以下のようなコマンドになります。
```shell
istioctl install --set profile=minimal --set values.pilot.env.ENABLE_NATIVE_SIDECARS=true
```

## 動作確認
前述のようなJobから作成されるPodが削除されない問題が発生しないことを確認してみます。

default Namespaceを使用して検証するので、まずはデプロイされるPodに対してサイドカーコンテナがデプロイされるようにラベル付けします。
```shell
kubectl label ns default istio.io/rev=default
```

そして、以下のJobをデプロイしてみます。
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: native-sidecar-test
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: busybox
          image: busybox
          command:
            - /bin/sh
            - -c
            - sleep 3
          resources:
            requests:
              cpu: 200m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 128Mi
```

```shell
kubectl apply -f sidecar-test-job.yaml
```

同時にPodの挙動を確認します。
```shell
kubectl get pod -w

NAME                        READY   STATUS    RESTARTS   AGE
native-sidecar-test-j7mk7   0/2     Pending   0          0s
native-sidecar-test-j7mk7   0/2     Pending   0          0s
native-sidecar-test-j7mk7   0/2     Init:0/2   0          0s
native-sidecar-test-j7mk7   0/2     Init:1/2   0          1s
native-sidecar-test-j7mk7   0/2     Init:1/2   0          2s
native-sidecar-test-j7mk7   0/2     PodInitializing   0          2s
native-sidecar-test-j7mk7   0/2     PodInitializing   0          3s
native-sidecar-test-j7mk7   1/2     PodInitializing   0          3s
native-sidecar-test-j7mk7   2/2     Running           0          6s
native-sidecar-test-j7mk7   1/2     Completed         0          9s
native-sidecar-test-j7mk7   0/2     Completed         0          10s
native-sidecar-test-j7mk7   0/2     Completed         0          12s
native-sidecar-test-j7mk7   0/2     Completed         0          12s
native-sidecar-test-j7mk7   0/2     Completed         0          13s
```
想定通りしっかりCompletedになりましたね。

## 最後に
Istioに限らずサイドカーコンテナを期待通りにハンドリングするのには一定の煩わしさがあったと思いますが、ここもだいぶ改善されます。
ぜひ移行に踏み切ってみてはいかがでしょうか？

ちなみに実装上では以下の部分でネイティブサイドカーの利用有無に関する切り替えが行われているようですね。面白いと思うのでコードを追ってみるのも一興かもしれないですね。
https://github.com/istio/istio/blob/1.23.0/pkg/kube/inject/inject.go#L461-L471

## おまけ
おまけとして、Istioに限らずネイティブサイドカーを使うときには確認した方がよい点を上げてみたいと思います。

### startupProbeとpostStart hookが設定されているか
kubeletはネイティブサイドカーコンテナの起動完了を`startupProbe`と`postStart hook`で判別します。これらを設定しないと、ネイティブサイドカーコンテナよりもメインコンテナの起動が先行してしまう可能性があります。

例えばGoogle Cloudの[Cloud SQL Auth Proxy](https://cloud.google.com/sql/docs/mysql/sql-proxy?hl=ja)を使うようなユースケースを考えると、auth-proxyコンテナよりもメインコンテナが先に立ち上がりCloud SQLにアクセスしてしまうかもしれません。

しかしauth-proxyはまだ起動できていませんから、当然失敗してしまいます。

こういったケースに対応するためにも`startupProbe`と`postStart hook`が適切に設定できているかを確認しましょう。

### terminationGracePeriodSecondsが適切な値に設定されているか
ネイティブサイドカーはinitContainerの1種ではありますが、`terminationGracePeriodSeconds`のカウントに含まれます。

メインコンテナだけではなくネイティブサイドカーもスコープに入れて十分に足りる秒数を確保できているかを確認しましょう。