---
title: Node上へのPodのスケジューリング
content_type: concept
weight: 20
---


<!-- overview -->

{{< glossary_tooltip text="Pod" term_id="pod" >}}が稼働する{{< glossary_tooltip text="Node" term_id="node" >}}を特定のものに指定したり、優先条件を指定して制限することができます。
これを実現するためにはいくつかの方法がありますが、推奨されている方法は[ラベルでの選択](/ja/docs/concepts/overview/working-with-objects/labels/)です。
スケジューラーが最適な配置を選択するため、一般的にはこのような制限は不要です(例えば、複数のPodを別々のNodeへデプロイしたり、Podを配置する際にリソースが不十分なNodeにはデプロイされないことが挙げられます)が、
SSDが搭載されているNodeにPodをデプロイしたり、同じアベイラビリティーゾーン内で通信する異なるサービスのPodを同じNodeにデプロイする等、柔軟な制御が必要なこともあります。

<!-- body -->

Kubernetesが特定のPodをスケジュールする場所を選択するために、以下のいずれかの方法を使用できます。

  * 合致する[Nodeラベル](#built-in-node-labels)に対してマッチングする[nodeSelector](#nodeselector)フィールド
  * [アフィニティとアンチアフィニティ](#affinity-and-anti-affinity)
  * [nodeName](#nodename)フィールド

## Nodeラベル {#built-in-node-labels}

多くのKubernetesオブジェクトと同様に、Nodeにも[labels](/docs/concepts/overview/working-with-objects/labels/)があり、明示的に[付与](/docs/tasks/configure-pod-container/assign-pods-nodes/#add-a-label-to-a-node)することが可能です。
また、Kubernetesはクラスター内のすべてのNodeにラベルの標準セットを付与します。
これらの一般的なラベルのリストは、[代表的なラベル、アノテーションとTaints](/docs/reference/kubernetes-api/labels-annotations-taints/)を参照してください。

{{< note >}}
これらのラベルは、クラウドプロバイダー固有であり、確実なものではありません。
例えば、`kubernetes.io/hostname`の値はNodeの名前と同じである環境もあれば、異なる環境もあります。
{{< /note >}}

## Nodeの隔離や制限
Nodeにラベルを付与することで、Podは特定のNodeやNodeグループにスケジュールされます。
これにより、特定のPodを、確かな隔離性や安全性、特性を持ったNodeで稼働させることができます。

この目的でラベルを使用する際に、Node上の{{<glossary_tooltip text="kubelet" term_id="kubelet">}}プロセスに上書きされないラベルキーを選択することが強く推奨されています。
これは、安全性が損なわれたNodeがkubeletの認証情報をNodeのオブジェクトに設定したり、スケジューラーがそのようなNodeにデプロイすることを防ぎます。

[`NodeRestriction`アドミッションプラグイン](/docs/reference/access-authn-authz/admission-controllers/#noderestriction)は、kubeletが`node-restriction.kubernetes.io/`プレフィックスを有するラベルの設定や上書きを防ぎます。

Nodeの隔離にラベルのプレフィックスを使用するためには、以下のようにします。

1. [Node authorizer](/docs/reference/access-authn-authz/node/)を使用していることと、`NodeRestriction`アドミッションプラグインが _有効_ になっていること。
2. Nodeに`node-restriction.kubernetes.io/` プレフィックスのラベルを付与し、そのラベルが[node selectors](#nodeselector)に指定されていること。
例えば、`example.com.node-restriction.kubernetes.io/fips=true` または `example.com.node-restriction.kubernetes.io/pci-dss=true`のようなラベルです。


## nodeSelector

`nodeSelector`は、Nodeを選択するための、最も簡単で推奨されている手法です。

Kubernetesは指定されたラベルを持つNodeにのみPodをスケジューリングします。
PodSpecに`nodeSelector`フィールドを追加することで、ターゲットとするNodeが保持している[Nodeラベル](#built-in-node-labels)を指定することができます。

詳しくは[Node上へのPodのスケジューリング](/docs/tasks/configure-pod-container/assign-pods-nodes)を参照してください。


## アフィニティとアンチアフィニティ {#affinity-and-anti-affinity}

`nodeSelector`はPodの稼働を特定のラベルが付与されたNodeに制限する最も簡単な方法です。
アフィニティ/アンチアフィニティでは、より柔軟な指定方法が提供されています。
利点は以下の通りです。

1. アフィニティ/アンチアフィニティという用語はとても表現豊かです。`nodeSelector`は指定されたラベルをすべて持つNodeしか選択できませんが、アフィニティ/アンチアフィニティでは、選択ロジックをより詳細に制御することができます。
2. ルールに*soft*か*preferred*を指定すると、一致するNodeが見つから ない場合でもスケジューラーがPodをスケジュールするようにできます。
3. Node自体のラベルではなく、Node(または他のトポロジカルドメイン)上で稼働している他のPodのラベルに対して条件を指定することができ、Node上でどのPodと共存させるかについてのルールを定義することができます。

アフィニティは"Nodeアフィニティ"と"Pod間のアフィニティ/アンチアフィニティ"の2種類から成ります。

* *Nodeアフィニティ*は`nodeSelector`フィールドと似ていますが、より表現力が豊かで、ソフトルールを指定することができます。
* *Pod間のアフィニティ/アンチアフィニティ*はNodeのラベルではなくPodのラベルに対して制限をかけます。


### Nodeアフィニティ

Nodeアフィニティは概念的には、NodeのラベルによってPodがどのNodeにスケジュールされるかを制限する`nodeSelector`と同様です。
現在は2種類のNodeアフィニティがあります。

 * `requiredDuringSchedulingIgnoredDuringExecution`:このルールを満たさない限り、スケジューラはーPodをスケジューリングできません。これは`nodeSelector`に似ていますが、より柔軟に条件を指定できます。
 * `preferredDuringSchedulingIgnoredDuringExecution`:スケジューラーはこのルールを満たすNodeを探しますが、一致するNodeが見つからない場合でも、スケジューラーはPodをスケジュールします。

{{<note>}}
`IgnoredDuringExecution`は、KubernetesがPodをスケジュール後、Nodeのラベルが変更され、Podがその条件を満たさなくなった場合でも、
PodはそのNodeで稼働し続けるということを意味します。
{{</note>}}

Nodeアフィニティは、PodSpecの`affinity`フィールドにある`nodeAffinity`フィールドで特定します。

Nodeアフィニティを使用したPodの例を以下に示します:

{{< codenew file="pods/pod-with-node-affinity.yaml" >}}

このNodeアフィニティでは、
  * Podはキーが`kubernetes.io/e2e-az-name`、値が`e2e-az1`または`e2e-az2`のラベルが付与されたNodeにしか配置されません。
  * 加えて、キーが`another-node-label-key`、値が`another-node-label-value`のラベルが付与されたNodeが優先されます。

`operator`フィールドを使用すると、Kubernetesがルールを解釈する際に使用する論理演算子を指定することができます。Nodeアフィニティでは、`In`、`NotIn`、`Exists`、`DoesNotExist`、`Gt`、`Lt`のオペレーターが使用できます。

`NotIn`と`DoesNotExist`はNodeアンチアフィニティ、またはPodを特定のNodeにスケジュールさせない場合に使われる[Taints](/ja/docs/concepts/scheduling-eviction/taint-and-toleration/)に使用します。

{{<note>}}
`nodeSelector`と`nodeAffinity`の両方を指定した場合、Podは**両方の**条件を満たすNodeにスケジュールされます。

`nodeAffinity`内で複数の`nodeSelectorTerms`を指定した場合、Podは**いずれかの**`nodeSelectorTerms`を満たしたNodeへスケジュールされます。

`nodeSelectorTerms`内で複数の`matchExpressions`を指定した場合にはPodは**全ての**`matchExpressions`を満たしたNodeへスケジュールされます。
{{</note>}}

詳しくは、[Nodeアフィニティを使用したNode上へのPodスケジューリング](/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/)をご覧ください。


#### Nodeアフィニティのweight
アフィニティタイプの`preferredDuringSchedulingIgnoredDuringExecution`の各インスタンスに対して、1から100の間の`weight`を指定することができます。スケジューラーは、Podの他のすべてのスケジューリング要件を満たすNodeを見つけると、そのNodeが満たすすべての優先ルールを繰り返し、その式の`weight`の値を合算します。

最終的な合計は、そのNodeの他の優先度関数のスコアに加算されます。スケジューラーがPodのスケジューリングを決定する際に、合計スコアが最も高いNodeが優先されます。

例えば、以下のようなPodSpecを考えてみます。

{{<codenew file="pods/pod-with-affinity-anti-affinity.yaml">}}

もし、`requiredDuringSchedulingIgnoredDuringExecution`ルールに合致するNodeが2つあり、1つは`label-1:key-1`ラベルを持ち、もう一つは `label-2:key-2`ラベルを持つ場合、スケジューラーはそれぞれのNodeの`weight`を考慮し、そのNodeの他のスコアに重みを加え、最終スコアの最も高いNodeにPodをスケジューリングします。

{{<note>}}
上記の例でKubernetesにPodを正常にスケジュールさせたい場合は、`kubernetes.io/os=linux`ラベルを持つNodeが存在する必要があります。
{{</note>}}

#### Node affinity per scheduling profile

{{< feature-state for_k8s_version="v1.20" state="beta" >}}

When configuring multiple [scheduling profiles](/docs/reference/scheduling/config/#multiple-profiles), you can associate
a profile with a node affinity, which is useful if a profile only applies to a specific set of nodes.
To do so, add an `addedAffinity` to the `args` field of the [`NodeAffinity` plugin](/docs/reference/scheduling/config/#scheduling-plugins)
in the [scheduler configuration](/docs/reference/scheduling/config/). For example:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta3
kind: KubeSchedulerConfiguration

profiles:
  - schedulerName: default-scheduler
  - schedulerName: foo-scheduler
    pluginConfig:
      - name: NodeAffinity
        args:
          addedAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: scheduler-profile
                  operator: In
                  values:
                  - foo
```

The `addedAffinity` is applied to all Pods that set `.spec.schedulerName` to `foo-scheduler`, in addition to the
NodeAffinity specified in the PodSpec.
That is, in order to match the Pod, nodes need to satisfy `addedAffinity` and
the Pod's `.spec.NodeAffinity`.

Since the `addedAffinity` is not visible to end users, its behavior might be
unexpected to them. Use node labels that have a clear correlation to the
scheduler profile name.

{{< note >}}
The DaemonSet controller, which [creates Pods for DaemonSets](/docs/concepts/workloads/controllers/daemonset/#scheduled-by-default-scheduler),
does not support scheduling profiles. When the DaemonSet controller creates
Pods, the default Kubernetes scheduler places those Pods and honors any
`nodeAffinity` rules in the DaemonSet controller.
{{< /note >}}



### Pod間アフィニティとアンチアフィニティ

Pod間アフィニティとアンチアフィニティは、Nodeのラベルではなく、すでにNodeで稼働しているPodのラベルに従ってPodがスケジュールされるNodeを制限します。
このポリシーは、"XにてルールYを満たすPodがすでに稼働している場合、このPodもXで稼働させる(アンチアフィニティの場合は稼働させない)"という形式です。
Yはnamespaceのリストで指定したLabelSelectorで表されます。
Nodeと異なり、Podはnamespaceで区切られているため(それゆえPodのラベルも暗黙的にnamespaceで区切られます)、Podのラベルを指定するlabel selectorは、どのnamespaceにselectorを適用するかを指定する必要があります。
概念的に、XはNodeや、ラック、クラウドプロバイダゾーン、クラウドプロバイダのリージョン等を表すトポロジードメインです。
これらを表すためにシステムが使用するNodeラベルのキーである`topologyKey`を使うことで、トポロジードメインを指定することができます。
先述のセクション[補足: ビルトインNodeラベル](#interlude-built-in-node-labels)にてラベルの例が紹介されています。


{{< note >}}
Pod間アフィニティとアンチアフィニティは、大規模なクラスター上で使用する際にスケジューリングを非常に遅くする恐れのある多くの処理を要します。
そのため、数百台以上のNodeから成るクラスターでは使用することを推奨されません。
{{< /note >}}

{{< note >}}
Podのアンチアフィニティは、Nodeに必ずラベルが付与されている必要があります。
言い換えると、クラスターの全てのNodeが、`topologyKey`で指定されたものに合致する適切なラベルが必要になります。
それらが付与されていないNodeが存在する場合、意図しない挙動を示すことがあります。
{{< /note >}}

#### Types of inter-pod affinity and anti-affinity

Similar to [node affinity](#node-affinity) are two types of Pod affinity and
anti-affinity as follows:

  * `requiredDuringSchedulingIgnoredDuringExecution`
  * `preferredDuringSchedulingIgnoredDuringExecution`

For example, you could use
`requiredDuringSchedulingIgnoredDuringExecution` affinity to tell the scheduler to
co-locate Pods of two services in the same cloud provider zone because they
communicate with each other a lot. Similarly, you could use
`preferredDuringSchedulingIgnoredDuringExecution` anti-affinity to spread Pods
from a service across multiple cloud provider zones.

To use inter-pod affinity, use the `affinity.podAffinity` field in the Pod spec.
For inter-pod anti-affinity, use the `affinity.podAntiAffinity` field in the Pod
spec.

Nodeアフィニティと同様に、PodアフィニティとPodアンチアフィニティにも必須条件と優先条件を示す`requiredDuringSchedulingIgnoredDuringExecution`と`preferredDuringSchedulingIgnoredDuringExecution`があります。
前述のNodeアフィニティのセクションを参照してください。
`requiredDuringSchedulingIgnoredDuringExecution`を指定するアフィニティの使用例は、"Service AのPodとService BのPodが密に通信する際、それらを同じゾーンで稼働させる場合"です。
また、`preferredDuringSchedulingIgnoredDuringExecution`を指定するアンチアフィニティの使用例は、"ゾーンをまたいでPodのサービスを稼働させる場合"(Podの数はゾーンの数よりも多いため、必須条件を指定すると合理的ではありません)です。

Pod間アフィニティは、PodSpecの`affinity`フィールド内に`podAffinity`で指定し、Pod間アンチアフィニティは、`podAntiAffinity`で指定します。

#### Podアフィニティを使用したPodの例

{{< codenew file="pods/pod-with-pod-affinity.yaml" >}}

このPodのアフィニティは、PodアフィニティとPodアンチアフィニティを1つずつ定義しています。
この例では、`podAffinity`に`requiredDuringSchedulingIgnoredDuringExecution`、`podAntiAffinity`に`preferredDuringSchedulingIgnoredDuringExecution`が設定されています。
Podアフィニティは、「キーが"security"、値が"S1"のラベルが付与されたPodが少なくとも1つは稼働しているNodeが同じゾーンにあれば、PodはそのNodeにスケジュールされる」という条件を指定しています(より正確には、キーが"security"、値が"S1"のラベルが付与されたPodが稼働しており、キーが`topology.kubernetes.io/zone`、値がVであるNodeが少なくとも1つはある状態で、
Node Nがキー`topology.kubernetes.io/zone`、値Vのラベルを持つ場合に、PodはNode Nで稼働させることができます)。
Podアンチアフィニティは、「すでにあるNode上で、キーが"security"、値が"S2"であるPodが稼働している場合に、Podを可能な限りそのNode上で稼働させない」という条件を指定しています
(`topologyKey`が`topology.kubernetes.io/zone`であった場合、キーが"security"、値が"S2"であるであるPodが稼働しているゾーンと同じゾーン内のNodeにはスケジュールされなくなります)。
PodアフィニティとPodアンチアフィニティや、`requiredDuringSchedulingIgnoredDuringExecution`と`preferredDuringSchedulingIgnoredDuringExecution`に関する他の使用例は[デザインドック](https://git.k8s.io/community/contributors/design-proposals/scheduling/podaffinity.md)を参照してください。

PodアフィニティとPodアンチアフィニティで使用できるオペレーターは、`In`、`NotIn`、 `Exists`、 `DoesNotExist`です。

原則として、`topologyKey`には任意のラベルとキーが使用できます。
しかし、パフォーマンスやセキュリティの観点から、以下の制約があります:

1. アフィニティと、`requiredDuringSchedulingIgnoredDuringExecution`を指定したPodアンチアフィニティは、`topologyKey`を指定しないことは許可されていません。
2. `requiredDuringSchedulingIgnoredDuringExecution`を指定したPodアンチアフィニティでは、`kubernetes.io/hostname`の`topologyKey`を制限するため、アドミッションコントローラー`LimitPodHardAntiAffinityTopology`が導入されました。
トポロジーをカスタマイズする場合には、アドミッションコントローラーを修正または無効化する必要があります。
3. `preferredDuringSchedulingIgnoredDuringExecution`を指定したPodアンチアフィニティでは、`topologyKey`を省略することはできません。
4. 上記の場合を除き、`topologyKey` は任意のラベルとキーを指定することができます。

`labelSelector`と`topologyKey`に加え、`labelSelector`が合致すべき`namespaces`のリストを特定することも可能です(これは`labelSelector`と`topologyKey`を定義することと同等です)。
省略した場合や空の場合は、アフィニティとアンチアフィニティが定義されたPodのnamespaceがデフォルトで設定されます。

`requiredDuringSchedulingIgnoredDuringExecution`が指定されたアフィニティとアンチアフィニティでは、`matchExpressions`に記載された全ての条件が満たされるNodeにPodがスケジュールされます。

#### Namespace selector
{{< feature-state for_k8s_version="v1.22" state="beta" >}}

You can also select matching namespaces using `namespaceSelector`, which is a label query over the set of namespaces.
The affinity term is applied to namespaces selected by both `namespaceSelector` and the `namespaces` field.
Note that an empty `namespaceSelector` ({}) matches all namespaces, while a null or empty `namespaces` list and
null `namespaceSelector` matches the namespace of the Pod where the rule is defined.

{{<note>}}
This feature is beta and enabled by default. You can disable it via the
[feature gate](/docs/reference/command-line-tools-reference/feature-gates/)
`PodAffinityNamespaceSelector` in both kube-apiserver and kube-scheduler.
{{</note>}}

#### 実際的なユースケース

Pod間アフィニティとアンチアフィニティは、ReplicaSet、StatefulSet、Deploymentなどのより高レベルなコレクションと併せて使用するとさらに有用です。
Workloadが、Node等の定義された同じトポロジーに共存させるよう、簡単に設定できます。


##### 常に同じNodeで稼働させる場合

３つのノードから成るクラスターでは、ウェブアプリケーションはredisのようにインメモリキャッシュを保持しています。
このような場合、ウェブサーバーは可能な限りキャッシュと共存させることが望ましいです。

ラベル`app=store`を付与した3つのレプリカから成るredisのdeploymentを記述したyamlファイルを示します。
Deploymentには、1つのNodeにレプリカを共存させないために`PodAntiAffinity`を付与しています。


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```

ウェブサーバーのDeploymentを記載した以下のyamlファイルには、`podAntiAffinity` と`podAffinity`が設定されています。
全てのレプリカが`app=store`のラベルが付与されたPodと同じゾーンで稼働するよう、スケジューラーに設定されます。
また、それぞれのウェブサーバーは1つのノードで稼働されないことも保証されます。


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.16-alpine
```

上記2つのDeploymentが生成されると、3つのノードは以下のようになります。

|       node-1         |       node-2        |       node-3       |
|:--------------------:|:-------------------:|:------------------:|
| *webserver-1*        |   *webserver-2*     |    *webserver-3*   |
|  *cache-1*           |     *cache-2*       |     *cache-3*      |

このように、3つの`web-server`は期待通り自動的にキャッシュと共存しています。



##### 同じNodeに共存させない場合

上記の例では `PodAntiAffinity`を`topologyKey: "kubernetes.io/hostname"`と合わせて指定することで、redisクラスター内の2つのインスタンスが同じホストにデプロイされない場合を扱いました。
同様の方法で、アンチアフィニティを用いて高可用性を実現したStatefulSetの使用例は[ZooKeeper tutorial](/docs/tutorials/stateful-application/zookeeper/#tolerating-node-failure)を参照してください。


## nodeName

`nodeName`はNodeの選択を制限する最も簡単な方法ですが、制約があることからあまり使用されません。
`nodeName`はPodSpecのフィールドです。
ここに値が設定されると、schedulerはそのPodを考慮しなくなり、その名前が付与されているNodeのkubeletはPodを稼働させようとします。
そのため、PodSpecに`nodeName`が指定されると、上述のNodeの選択方法よりも優先されます。

 `nodeName`を使用することによる制約は以下の通りです:

-   その名前のNodeが存在しない場合、Podは起動されす、自動的に削除される場合があります。
-   その名前のNodeにPodを稼働させるためのリソースがない場合、Podの起動は失敗し、理由は例えばOutOfmemoryやOutOfcpuになります。
-   クラウド上のNodeの名前は予期できず、変更される可能性があります。

`nodeName`を指定したPodの設定ファイルの例を示します:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: kube-01
```

上記のPodはkube-01という名前のNodeで稼働します。



## {{% heading "whatsnext" %}}


[Taints](/ja/docs/concepts/scheduling-eviction/taint-and-toleration/)を使うことで、NodeはPodを追い出すことができます。

[Nodeアフィニティ](https://git.k8s.io/community/contributors/design-proposals/scheduling/nodeaffinity.md)と
[Pod間アフィニティ/アンチアフィニティ](https://git.k8s.io/community/contributors/design-proposals/scheduling/podaffinity.md)
のデザインドキュメントには、これらの機能の追加のバックグラウンドの情報が記載されています。

一度PodがNodeに割り当たると、kubeletはPodを起動してノード内のリソースを確保します。
[トポロジーマネージャー](/docs/tasks/administer-cluster/topology-manager/)はNodeレベルのリソース割り当てを決定する際に関与します。

