---
title: "Kube-schedulerプラグインCoschedulingを体験してみた"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Kubernetes"]
published: true
published_at: 2024-12-11 10:00
---

本記事は [3-shake Advent Calendar 2024](https://qiita.com/advent-calendar/2024/3-shake) シリーズ 1 の 11 日目の記事です。

## はじめに
ここ最近Kubernetesのスケジューリングについて調査する機会があり、その一環でスケジューラープラグインの1つである[Coscheduling](https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/pkg/coscheduling)についても調査しました。この時の調査と簡単なハンズオンについてこの記事でまとめてみたいと思います。

Kubernetesのコントロールプレーンの1コンポーネントであるスケジューラはpluginによる機能拡張が可能です。プラグインは以下のリポジトリにまとまっています。
https://github.com/kubernetes-sigs/scheduler-plugins
今回はこのプラグインの1つであるCoschedulingを取り上げます。

## Coschedulingについて
まずは以下のKEPをベースにCoschedulingについて調べました。
https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/kep/42-podgroup-coscheduling

Coschedulingは複数のPodをグルーピングしてまとめてスケジュールすることを意味しており、Coschedulingプラグインではこれを実現するための機能を提供しています。

KEPで挙げられているユースケースとしてSparkを使った大規模な分散データ処理やTensorFlowを用いたモデルトレーニングジョブがあります。これらは複数のPodが協調しながら動作するため一部だけが動いていても単にリソースを食い潰してしまうことになります。
こういった問題がモチベーションとなってCoschedulingが実装されました。

:::
Coschedulingに似たような概念にgang-schedulingがあります。
gang-schedulingも複数のPodをグルーピングしてスケジュールすることには変わりないのですが、グルーピングされたPod全てがスケジュールされる or 全くスケジュールされないの2択です。
一方でCoschedulingでは言うなれば「できるだけまとめてスケジューリングする」と表現することができます。
:::

## ハンズオン
ハンズオンは公式がドキュメントを残してくださっていますので、これにしたがって実施してみます。有り難し...
https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/doc/install.md

### kindで検証のためのクラスターを作成する
まずは検証用のクラスターを作成します。
今回はコントロールプレーン用1台、データプレーン用2台構成です。

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: coscheduling
networking:
  disableDefaultCNI: false
nodes:
  - role: control-plane
  - role: worker
  - role: worker

```

### scheduler-pluginをインストールする
scheduler-pluginをインストールするのには2つの方法があるとのことです。それぞれpros/consが挙げられています。

**【セカンドスケジューラーとしてインストールする】**

**pros**: [Helm Chart](https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/manifests/install/charts/as-a-second-scheduler)により簡単にインストールできる

**cons**: マルチスケジューラではクラスターのリソースが少なくなったときにコンフリクトが発生する

これについてはさらに具体例をもとに説明されています。ノードは1つのPodがフィットするリソース量として、複数のスケジューラがそれぞれPodをスケジュールしようと試みたケースでの挙動を考えます。
1. 先にスケジュールされたPodがノードにバインドされ、ノードリソースを予約する
2. 後続でスケジュールされるPodは先にPodがバインドされているためリソース不足でKubeletによってevictされる 
3. しかし、後続のKubeletにevictされたPodは`.Spec.nodeName`フィールドが残り続ける。つまり、Podが既にノードに割り当てられているが、実際には実行されていない状態となる
4. kubeletにevictされたがスケジューラはPodが既にノードに割り当てられていると認識しているため、再スケジュールを行わずハング状態となる

:::message alert
このような事象があるため、**本番環境では推奨されていません。**
:::

**【シングルスケジューラとしてインストールする】**

**pros**: 統合されたスケジューラを使用するためリソースの競合がなくなる。本番環境に推奨される

**cons**: コントロールプレーンを操作する権限が必要で、現時点ではインストールは完全には自動化されていない（Helmチャートはまだない）


上記のような説明を踏まえて今回はシングルスケジューラとしてインストールすることとします。

まずは、コントロールプレーンノードのコンテナでシェルを拾います。
```shell
podman ps -a
CONTAINER ID  IMAGE                                                                                           COMMAND     CREATED         STATUS         PORTS                      NAMES
8f0268eca4fa  docker.io/kindest/node@sha256:047357ac0cfea04663786a612ba1eaba9702bef25227a794b52890dd8bcd692e              35 minutes ago  Up 35 minutes                             coscheduling-cluster-worker2
47f18af0cb59  docker.io/kindest/node@sha256:047357ac0cfea04663786a612ba1eaba9702bef25227a794b52890dd8bcd692e              35 minutes ago  Up 35 minutes  127.0.0.1:64912->6443/tcp  coscheduling-cluster-control-plane
b313929f385d  docker.io/kindest/node@sha256:047357ac0cfea04663786a612ba1eaba9702bef25227a794b52890dd8bcd692e              35 minutes ago  Up 35 minutes                             coscheduling-cluster-worker

podman exec -it coscheduling-cluster-control-plane bash
root@coscheduling-cluster-control-plane:/#
```

この後`/etc/kubernetes/manifests/kube-scheduler.yaml`を変更するため、事前にバックアップを作成しておきます。このファイルはkube-schedulerのPod定義を書いたマニフェストファイルです。
```shell
root@coscheduling-cluster-control-plane:/# cp /etc/kubernetes/manifests/kube-scheduler.yaml /etc/kubernetes/kube-scheduler.yaml
```

次にschedulerの設定を定義したマニフェストファイルを作成します。
```yaml
cat <<EOF > /etc/kubernetes/sched-cc.yaml
> apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
leaderElection:
  # (Optional) Change true to false if you are not running a HA control-plane.
  leaderElect: false
clientConnection:
  kubeconfig: /etc/kubernetes/scheduler.conf
profiles:
- schedulerName: default-scheduler
  plugins:
    multiPoint:
      enabled:
      - name: Coscheduling
      disabled:
      - name: PrioritySort
> EOF
```

scheduler-pluginのためのdeploymentやRBAC設定が書かれたマニフェストが提供されているのでこれをapplyします。
```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/scheduler-plugins/refs/heads/master/manifests/install/all-in-one.yaml
```

さらに使用するscheduler-pluginに合わせてCRDをインストールします。今回はCoschedulingを利用するため、PodGroupのCRDをインストールです。カスタムリソースのPodGroupについては後述します。
```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/scheduler-plugins/refs/heads/master/manifests/coscheduling/crd.yaml
```

最後にkube-schedulerがCoschedulingとともに動くようにマニフェストを書き換えます。ファイル変更後は以下のようになればOKです。
```
# cat kube-scheduler.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --config=/etc/kubernetes/sched-cc.yaml
    image: registry.k8s.io/scheduler-plugins/kube-scheduler:v0.30.6
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10259
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    name: kube-scheduler
    resources:
      requests:
        cpu: 100m
    startupProbe:
      failureThreshold: 24
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10259
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    volumeMounts:
    - mountPath: /etc/kubernetes/scheduler.conf
      name: kubeconfig
      readOnly: true
    - mountPath: /etc/kubernetes/sched-cc.yaml
      name: sched-cc
      readOnly: true
  hostNetwork: true
  priority: 2000001000
  priorityClassName: system-node-critical
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  volumes:
  - hostPath:
      path: /etc/kubernetes/scheduler.conf
      type: FileOrCreate
    name: kubeconfig
  - hostPath:
      path: /etc/kubernetes/sched-cc.yaml
      type: FileOrCreate
    name: sched-cc
status: {}
```

一応手元で事前に取っておいてバックアップファイルとdiffを取ってみたのでご参考までに。
```
# diff kube-scheduler.yaml kube-scheduler.yaml.bak
17,18c17,19
<     - --config=/etc/kubernetes/sched-cc.yaml
<     image: registry.k8s.io/scheduler-plugins/kube-scheduler:v0.30.6
---
>     - --kubeconfig=/etc/kubernetes/scheduler.conf
>     - --leader-elect=true
>     image: registry.k8s.io/kube-scheduler:v1.30.0
48,50d48
<     - mountPath: /etc/kubernetes/sched-cc.yaml
<       name: sched-cc
<       readOnly: true
62,65d59
<   - hostPath:
<       path: /etc/kubernetes/sched-cc.yaml
<       type: FileOrCreate
<     name: sched-cc
```

### 動作確認
CoschedulingではPodをグルーピングしてスケジュールするため、まずはPodGroupというカスタムリソースを利用してどのようなグルーピングを行うのかをあらかじめ定義しておきます。

以下のようなマニフェストをデプロイしておきます。
```yaml
# podgroup.yaml
apiVersion: scheduling.x-k8s.io/v1alpha1
kind: PodGroup
metadata:
  name: pg1
spec:
  scheduleTimeoutSeconds: 10
  minMember: 3
```
specでは2つのパラメータを設定しています。
* scheduleTimeoutSeconds: ノードにバインドされるのを待つ際のタイムアウトの時間
* minMember: PodGroupとしてスケジュール可能な最小のPod数

次にdeploymentをデプロイしますが、このPodGroupに所属することを表現するために`spec.template.metadata.labels`に`scheduling.x-k8s.io/pod-group: pg1`というラベルを貼ります。
```yaml
# deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pause
spec:
  replicas: 2
  selector:
    matchLabels:
      app: pause
  template:
    metadata:
      labels:
        app: pause
        scheduling.x-k8s.io/pod-group: pg1
    spec:
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.6
```

この時点ではまだレプリカ数は2のためスケジュールされずにPending statusとなっているのが期待値です。
```
❯ kubectl get pod -w
NAME                    READY   STATUS    RESTARTS   AGE
pause-897896f6b-2zklq   0/1     Pending   0          84s
pause-897896f6b-nkc7p   0/1     Pending   0          84s
```

さらにPodのEventsを見てみます。
```
❯ kubectl describe po pause-897896f6b-nkc7p

...

Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  32s   default-scheduler  0/3 nodes are available: pre-filter pod pause-897896f6b-nkc7p cannot find enough sibling pods, current pods number: 2, minMember of group: 3. preemption: 0/3 nodes are available: 3 Preemption is not helpful for scheduling., PodGroup default/pg1 gets rejected due to Pod pause-897896f6b-nkc7p is unschedulable even after PostFilter
```
ちゃんとMessageでPodGroupの制約で止められている旨が記載されています。

次にdeploymentをスケールさせてPod数を3つにしてみます。
```shell
❯ kubectl scale deploy pause --replicas=3
deployment.apps/pause scaled

❯ kpod -w
NAME                    READY   STATUS    RESTARTS   AGE
pause-897896f6b-2zklq   0/1     Pending   0          84s
pause-897896f6b-nkc7p   0/1     Pending   0          84s
pause-897896f6b-njsfn   0/1     Pending   0          0s
pause-897896f6b-njsfn   0/1     Pending   0          0s
pause-897896f6b-2zklq   0/1     Pending   0          2m7s
pause-897896f6b-nkc7p   0/1     Pending   0          2m7s
pause-897896f6b-2zklq   0/1     ContainerCreating   0          2m7s
pause-897896f6b-njsfn   0/1     ContainerCreating   0          0s
pause-897896f6b-nkc7p   0/1     ContainerCreating   0          2m7s
pause-897896f6b-2zklq   1/1     Running             0          2m11s
pause-897896f6b-njsfn   1/1     Running             0          4s
pause-897896f6b-nkc7p   1/1     Running             0          2m11s
```

再びPodのEventsを見てみます。
```
❯ kubectl describe po pause-897896f6b-nkc7p

...

Events:
  Type     Reason            Age    From               Message
  ----     ------            ----   ----               -------
  Warning  FailedScheduling  2m21s  default-scheduler  0/3 nodes are available: pre-filter pod pause-897896f6b-nkc7p cannot find enough sibling pods, current pods number: 2, minMember of group: 3. preemption: 0/3 nodes are available: 3 Preemption is not helpful for scheduling., PodGroup default/pg1 gets rejected due to Pod pause-897896f6b-nkc7p is unschedulable even after PostFilter
  Normal   Scheduled         14s    default-scheduler  Successfully assigned default/pause-897896f6b-nkc7p to coscheduling-cluster-worker2
  Normal   Pulling           14s    kubelet            Pulling image "registry.k8s.io/pause:3.6"
  Normal   Pulled            11s    kubelet            Successfully pulled image "registry.k8s.io/pause:3.6" in 2.442s (2.442s including waiting). Image size: 253553 bytes.
  Normal   Created           11s    kubelet            Created container pause
  Normal   Started           11s    kubelet            Started container pause
```

問題なくスケジュールされたことが確認できました。

## 最後に
今回はCoschedulingについて説明しましたが、具体的な実装のお話などは省きました。

CoschedulingはScheduling Frameworkという機構に則って実装されています。しかしschedulerを拡張するもう一つの方法としてWebhookを活用する方式があります。

こういった実装上の違いも含めてKubernetesのスケジューラは奥が深くて、パフォーマンス面や複雑なケースに対しての対策など様々な工夫がコードから読み取れてとても好奇心をくすぐられました。

ぜひ皆様も一度スケジューラについて勉強してみてはいかがでしょうか？