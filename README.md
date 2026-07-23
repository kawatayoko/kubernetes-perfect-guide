# kubernetes-perfect-guide
# 第1章 Docker
## Dockerコンテナの復讐
- OCI準拠のコンテナイメージ、Dockerイメージ
    - OCI準拠のコンテナイメージを利用するケースもある
    - Dockerイメージの普及率は非常に高い 
- DockerコンテナとはDockerイメージをもとに実行される
    - 環境の影響を受けずにさまざまな環境上でコンテナを起動させることができる
        「Build Once, Run Anywhere」
## Dockerコンテナの設計
- 1コンテナにつき1プロセス
    - Dockerコンテナはアプリケーションとそのアプリケーションを実行するための実行環境をパッケージングするもの
    - それによりアプリケーションの実行を容易にする
- Immutable Infrastrcutre
    - 環境を変更する際は古い環境は廃棄して新たにつくる
        - Kubernetesがやってくれる
    - 一度作られた環境は変更されないことを徹底する
        - コンテナイメージの設計者が考慮する
        - コンテナ起動後に外部から実行バイナリを取得したり、パッケージをインストールしたりすると、外部要因によってコンテナイメージの実行結果が変わってしまう
        - コンテナイメージの中にアプリケーション実行バイナリや関連リソースを埋め込み、コンテナイメージを不変的な状態にすること
- 軽量
    - コンテナ実行時、ノード上に利用するDockerイメージがない場合、外m部からイメージをプルする
    - そのためDockerイメージはなるべく軽量な状態にするべき
        - Alpine Linux
            musl libc と busybox をベースとした軽量Linux
            Ubuntu: glibc + GNU coreutils
            Alpine: musl libc + BusyBox
            - Busy Box
                Busy Box Project
                https://busybox.net/
                複数のUNIXコマンド（ls,cp,sh,mount）を単一バイナリにまとめる設計
                ストレージやメモリが限られた環境（組み込みlinuxなど）でも標準的なコマンドを利用できる
            - musl
                musl = musl libc のこと
                musl は OSS の C標準ライブラリプロジェクト
                glibcが大きく複雑すぎることからこのプロジェクトが発足した
        - Distroless
        - scratch
            シェルすらも入っていない
            完全な空イメージ
            Linuxディストリビューションですらない
            超上級者向け
                Go界隈だと割とみるらしい。。

- 実行ユーザはroot以外
- Dockerのベストプラクティス
    https://docs.docker.jp/engine/articles/dockerfile_best-practice.html

- マルチステージビルド
    - Dockerイメージを`build`用と`実行`用とに分けて、最終imageを小さく・安全にする手法
        - 余計なコンパイルツールがないことはセキュリティ的にも望ましい
    - golang:1.14.1-alpine3.11
        - alpineイメージをベースとしてつくられた小さなイメージ
        - しかし、Goのコンパイルツールがふくまれており、イメージのサイズは大きくなる
        - マルチステージビルドを利用することでイメージを小さくできる
        ```
        # Stage1のコンテナ（アプリケーションをコンパイル）
        FROM golang:1.14.1-alpine3.11 as builder
        COPY ./main.go ./
        RUN go build -o /go-app ./main.go
        # Stage2のコンテナ（コンパイルしたバイナリを内包した実行用コンテナ）
        FROM alpine:3.11
        # Stage1でコンパイルした成果物をコピー
        COPY --from=builder /go-app .
        ENTRYPOINT ["./go-app"]
        ```
        - BuildKitを使うことで、ビルドステップの依存関係を自動判別し、ビルドステップをへいれてうに処理することも可能

- イメージレイヤの統合とイメージの縮小化
    - Dive
        https://github.com/wagoodman/dive
        Dockerイメージ

- docker hubへイメージをプッシュ
```
docker image tag sample-image:0.1 kawatayoko2/sample-image:0.1
docker image push kawatayoko2/sample-image:0.1
```

# 第2章
- Kubernetes はコンテナ化あれたアプリケーションのデプロイ、スケーリングなどの管理を自動化するためのプラットフォーム（コンテナオーケストレーションエンジン）
    - Kubernetes以外だと以下がある
        - Docker Swarm mode
        - Apatch Mesos
    - もともとはGoogle社内で利用されていたコンテナクラスタマネージャの`Borg`のアイデアをもとに作られた
    - マネージドサービス
        - GKE
            Google Kubernetes Engine
        - AKS
            Azure Container
        - EKS
            Amazon Elastic Kubernetes Service
- Kubernetesでできること
    - コンテナとは「なんらかのアプリケーションを実行するようビルドされたコンテナイメージをもとに起動されたワークロード」
    - 宣言的なコードによる管理（Infrastructure as Code）
    - スケーリング・オートスケーリング
        Kubernetesはコンテナクラスタ（Kubernetesクラスタ）を形成して、複数のKubernetes Nodeを管理する。
    - スケジューリング
        どのNodeにコンテナを配置するかを決定する
        アベイラビリティゾーンを識別する情報を不可して、マルチゾーンにコンテナを分散配置可能
    - リソース管理
        NodeのCPUやメモリの空きリソースを考慮してコンテナを配置する
    - セルフヒーリング
        標準でコンテナのプロセス監視を行っている
        プロセスの停止を検知すると自動でコンテナを再デプロイする
    - ロードバランシングとサービスディスカバリ
        - Service, Ingressといったロードバランシング機能
            条件に合致するコンテナ群に対してルーティングを行うエンドポイントを払い出すことができる
        - 個々のマイクロサービスがお互いのマイクロサービスを参照する際にサービスディスカバリ鬼王が役立つ
    - データの管理
        - バックエンドのデータストアにetcdを採用
            etcdはクラスタを組むことで冗長化できる
        - Kubernetes対応しているミドルウェア
            - Ansible
                Kubernetesへのコンテナデプロイ
            - Apache Ignite
            - Fluentd
            など
# 第３章
- Kubernetesクラスタ
    - ローカルKubernetes
        ユーザーの手元のマシン１代に構築して使用する
        個人での動作確認や開発環境としての利用い適している
        冗長化されていないためプロダクション環境には適していない
        - Minikube
            - 手元のマシン常にローカルKubernetesを簡単に構築・実行できるツール
            - 実行されるKuvernetesはシングルノード構成
            - ローカルの仮想マシン上にKubernetesをインストールするためハイパーバイザーが必要
            - ハイパーバイザー
                - Mac
                Docker/HyperKit driver/VirtualBox/Parallels/VMware/Fusion/Podman
                - Linux
                Docker/VirtualBox/Podman/KVM/Baremetal(systemd-based)
                - Windows
                Docker/VirtualBox/Hyper-V
            - minikubeを動かす
            ```
            brew intall minikube
            minikube start --driver=docker
            minikube status
            # このときKuvernetesクラスタは、Docker常に起動されたマシンの中で動作している
            # これ以降はkubectlを使用してMinikubeクラスタの操作が可能
            # 以下のコマンドでContextを切り替えると、それ以降はkubectlでMinikubeのクラスタを操作することが可能（複数のクラスタを利用している場合にはkubectlのContextを切り替えて利用する必要がある）
            kubectl config use-context minikube
            # Minikubeの削除
            minikube delete
            ```
        - Docker Desktop for Mac / Windows
        - kind (Kubernetes in Docker)
            - Kubernete自体の開発のために作られたツール
                - SIG-Testingという分科会で作られた。
            - Dockerコンテナを複数個起動し、そのコンテナをKubernetes Node　として利用することで、複数台構成のKubernetesクラスタを構築する
            - ローカル環境でマルチノードクラスタを構築するいはKindが一番よい
            - クラスタ構築はYAMLファイルで管理可能
                - kindで利用するDockerイメージはDocker Hubのkindest Organizationで管理されている
            ```
            kind create cluster --donfig kind.yaml --name kindcluster
            # contextの切り替え
            kubectl config use-context kind-kindcluster 
            # kindクラスタの削除
            kind delete cluster --name kindcluster
            ```
            - FeatureGetes
                Kubernetesの機能を有効化・無効化するための仕組み
                基本的にベータ以上のステータスになった機能についてはデフォルトで有効化される
                アルファステータスの機能については明治的に有効化する必要がある
                FeatureGateを確認するだけで、Kubernetesの新しい機能を把握することができる
    - Kubernetes構築ツール
        ツールを利用して任意の環境（オンプレ・クラウド）に裏スタを構築して使用する
        オンプレミス常にKubernetesをデプロイするケースや細かいカスタマイズを行いたい場合は、Kubernetes構築ツールを利用する
        - kubeadm
        - Rancher
    - マネージドKubernetesサービス
        パブリッククラウド常のマネージドサービスとして提供されるクラスタを利用
        基本的にマネージドKubernetesサービスが利用できる場合は、これを利用する。
        - GKE
        - AKS
        - EKS

- Kubernetesのサービスレベル目標
    - Kubernetesではスケーラビリティに関して`SIG-Scalability`という分科会で議論されている
        - サービスレベル指標（SLI：Service Level Indicator）
        - サービスレベル目標（SLO：Service Level Objectice）
        を定義している
        - https://github.com/kubernetes/community/blob/main/sig-scalability/slos/slos.md
            - APIの応答性
                - 単一オブジェクトの変更：APIリクエストにおいて、過去５分間のうち99%が1秒以内に帰ってくること
                - 非ストリーミングの取得：APIリクエストにおいて過去５分間のうち99%が下記の秒数以内にかえってくること
                    - 特定リソース 1秒
                    - Namespace 全体
                    - クラスタ全体 30秒
            - Podの起動時間
                - 過去５分間のうち99%が5秒以内に起動すること
                (イメージのPull時間やinitContainerの処理時間は含まない)
            - https://github.com/kubernetes/community/blob/main/sig-scalability/configs-and-limits/thresholds.md
    - 構築ツールを使って構築する場合、自分でKubernetes Masterのインスタンスサイズを決める必要がある
        - Kubernetes公式の自動構築スクリプト`kube-up`を利用してGCPやAWS上にクラスタを構築する場合にしようされるインスタンスサイズの目安が公開されている
            https://kubernetes.io/docs/setup/best-practices/cluster-large/
- Kubernetes構築ツール
    - kubeadm
        Kubernetesが公式に提供している構築ツール
    - Flannel
        - 基本的にDockerで起動したコンテナに付与されるIPアドレスはホストの外から疎通性のないInternalIPとなる。
            - ノードを跨いでコンテナ同士が通信することはできない
            - マルチノードのKubernetesクラスタとして成立させるには、各ホストのInternal Networkの接続性を確保する必要がある
        - それらを解決する方法としてFlannelがある
            - Flannelはノード間を繋ぐネットワークにかそうて系なトンネル（オーバーレイネットワーク）を構成する
            - それによりKubernetesクラスタ内のPod同士の通信を実現する
        - Flannel以外にもPodネットワークを実現する手段は複数ある
    - Rancher
        - オープンソースのコンテナプラットフォーム
            - Rancher Labs社
        - Rancher 2.0の特徴
            - 複数クラスタの統合管理
            - Kubernetesクラスタをさまざまなプラットフォームにデプロイ
                - AWS, OpenStack, VMware, etc
            - オンプレ環境を含む既存のKubernetesクラスタをRancherで管理（クラスタのインポート機能）
            - 複数クラスタにアプリケーションデプロイ可能
            - 中央集権的な認証、モニタリング、WebIO
            - 豊富なアプリケーションカタログ
            - Istioとの統合・連携
    - その他の構築ツール
        - Kubespray
        - kube-aws
        - kops
        - OpenStack Magnum
- パブリッククラウド上のマネージドKubernetesサービス
    - GKE(Google Kubernetes Engine)
        - 2015年8月にGA
        - Kubernetes NodeにGCE（Google Compute Engine）を採用
        - NodePool
            - Kubernetesクラスタ内のノードに対してラベル付けをしておくことでグルーピングをするような機能
            - `vCPU数の多いノード`,`メモリ容量の多いノード` のようにマシン性能が異なるノードをクラスタに混在させ、NodePoolによるラベル付をしておくことで、コンテナのスケジューリング時にデプロイ先の制約条件にすることが可能
            - NodePoolの機能はGKE以外にもさまざまなマネージドKubernetesサービスで提供されている
    - AKS(Azuru Kubernetes Service)
        - 2018年6月にGA
    - EKS(Amazon Elastic Kubernetes Service)
        - 2018年６月にGA
        - 通常はAMIから作成したEC2インスタンスをKubernetes Nodeとして利用する
        - AWS FargateをKuernetes Nodeとして利用するEKS on Fargate もある
- Kubernetes プレイグラウンド
    https://labs.play-with-k8s.com/
    最大で５台のインスタンスを起動でき、表示する手順に従ってkubeadmを使ってKubernetesクラスタを構築して使用できる
    "play with kubernetes" は終了した。
    今は、https://killercoda.com/　を利用するとよい。

# 第４章 APIリソースとkubectl
- kubectlのインストール
    - kubectlにはシェル補完機能も用意されている
        - bash 
        `source <(kubectl comletion bash)`
        - zsh
        `source <(kubectl comletion zsh)`
        - 補完を永続化するには、~/.bashrc ~/.zshrc に設定しておくこと
- Kubernetesの基礎
    - Kubernetesは以下２種類のノードから成り立っている
        - Kubernetes Master
            - APIエンドポイントの提供
                - kubectlはマニフェストファイルの情報を元にAPIにリクエストを送っている
                - RESTful APIで実装されているので、kubectlを使わずcurlなどで呼び出し可能
                - kubernetes Masterにリソースの登録を行う
            - コンテナのスケジューリング
            - コンテナのスケーリング           
        - Kubernetes Node
            - コンテナが起動するノード
- Kubernetesとリソース
    - Kubernetesが扱うリソースの一覧はAPIリファレンスに公開されている
        - https://kubernetes.io/docs/reference/kubernetes-api/?utm_source=chatgpt.com
        - `kubectl api-resources`でも確認できる
    - Workloads APIsカテゴリ
        クラスタ上にコンテナを起動させるために利用するリソース
        - Pod
        - ReplicationController
        - ReplicaSet
        - Deployment
        - DaemonSet
        - StatefulSet
        - Job
        - CronJob
    - Service APIsカテゴリ 
        コンテナのサービスディスカバリ、クラスタ外部からアクセス可能なエンドポイントの提供
        - Service
            - ClusterIP
            - ExternalIP
            - NodePort
            - LoadBalancer
            - Headless
            - ExternalName
            - Non-Selector
        - Ingress        
    - Config&Storage APIs カテゴリ
        設定や機密データをコンテナに埋め込んだり、永続ボリュームを提供する
        - Secret
        - ConfigMap
        - PersistentVolumeClaim    
    - Cluster APIs カテゴリ
        クラスタ自体の振る舞いを定義
        - Node
        - Namespace
        - PersistentVolume
        など
    - MetadataAPIs カテゴリ
        クラスタ内の他のリソースの動作を制御するためのリソース
        - LimitRange
        - HorizontalPodAutoscaler
        - PodDisruptionBudget
        - CustomResourceDefinition
- Namespace
    KubernetesにはNamespaceと呼ばれる仮想的なKubernetesクラスタの分離機能がある
    完全な分離レベルではないので使い所は限られる
    初期状態では4つのNamespaceが作成されている
    - 4つのNamespace
        - kube-system
            Kubernetes Dashboardといったシステムコンポーネントやアドオンがデプロイされる
        - kube-public
            全ユーザーが利用するConfigMapなどを配置
        - kube-node-lease
            ノードのハートビート情報
        - default
    - 目的に応じて任意のNamespaceを作成できる。1つのKubernetesクラスタを複数の目的で共用利用する予定がない場合、デフォルトをそのま利用してOK
    - 分割粒度はマイクロサービスを開発するチームごとにするのがよいが、筆者としてはクラスタごとわけるべき
        - クラスタアップグレードの際、同時に全ての環境で障害が発生する可能性がある
        - Namespaceの命名規則がprd-nsl/stg-nslのようになることでマニフェストの再利用性が低下する。クラスタを分ければ各環境で同じNamespace名を使用できるため全く同一のマニフェストを利用できる
        - Serviceの名前解決時に「SERVICE.prd-nsl.svc.cluster.local」など異なる宛先に通信を行う必要がでてくる
- CLIツール kubectl        
    - Kubernetesではクラスタ操作はすべてKubernetes MasterのAPIを通して行われる
        - kubectlを利用することが一般的
        - kubeconfig(~/.kube.config)に接続先サーバ、認証情報を保存する 
            - clusters
            接続先クラスタの情報
            - users
            認証情報(X.509クライアント証明書・トークン・パスワード・Webhook)
            - contexts
            clusterとuserのペアとnamespaceを指定したものを定義
        - kubectl は Context(current-context)を切り替えることで複数の環境を複数の権限で操作できる
            - `kubectl config get-contexts`
            Contextの一覧表示
            - `kubectl config use-context prd-admin`
            Contextの切り替え
            - `kubectl config current-context`
            現在のContextを表示
        - kubectlの代わりに`kubectx`/`kubens`などを利用できる
    - マニフェストとリソースの作成・削除更新
        - kubectl create
        - kubectl delete
        - kubectl apply
            - Client-side apply
            リソース作成にもkubectl applyを使用した方がよい
            差分適応をするする際、前回適応したマニフェストを参照するが、
            `kubectl create --save-config` のように `--save-config`オプションをつけない場合にマニフェストが保存されず、前回適応したマニフェストを参照できない
            - Server-side apply
                - `--server-side`オプションを利用する
                - kubectl.kubernetes.io/last-applied-configuration が不要
    - Podの再起動
        - DeploymentリソースのすべてのPodのリスタート
        - `kubectl rollout restart deployment sample-deployment`
    - マニフェストファイルの設計
        - 一つのマニフェストファイルの中に複数のリソースを記述
            - 実行順序を厳密にしたい（上から順に実行される）場合、リソース間の結合が強い場合はマニフェストをまとめる
            - 共通で利用する設定ファイルは別のマニフェストに分割すべき
                - ConfigMapリソース
                - Secretリソース
        - 複数のアニフェスとファイルを同時に適応する
            - ディレクトリは以下に複数ファイルを配置
            - `kubectl apply`コマンド実行時にディレクトリを指定する
            - `-R`オプションで再起的にディレクトリ内のマニフェストを適応可能
                `kubectl apply -f ./ -R`
            - 一部ファイルに文法エラーなどの問題があると、そのファイル以外が適応される
        - マニフェストファイルの設計指針 
            - マイクロサービスアーキテクチャとの親和性が高い
                - そこまで大きくない規模の場合
                    - 全てのサービスのマニフェストを一つのディレクトリにまとめる
                        - マニフェストファイルを指定して、特定のサービスをアップデート可
                        - ディレクトリを指定して、システム全体のアップデートも可能
                - 巨大なシステムの場合
                    - サブシステムや部署ごとにディレクトリを分ける
                        - 部署ごとに切れば、コンウェイの法則
                    - マイクロサービスごとにディレクトリを切る
        - アノテーション
            - システムコンポーネントが利用するメタデータ
                - metadata.annotations
                - リソースに対するメモ書き
                - 目的
                    - システムコンポーネントのためにデータを保存する
                        - kubectl.kubernetes.io/last-applied-configuration
                        - システムコンポーネントが自動で付与する
                        - ユーザーがannotationを付与していなくても様々なannotationが付与されている
                    - 全ての環境では利用できない設定を行う
                        - GKE/AKS/EKS などの環境特有の拡張機能が実装されている場合、その設定にアノテーションを使う
                            - 例：Serviceリソース
                                - GKEでは`Google Cloud Load Balancing`と連携
                                - EKSでは`AWS Classic Load Balancer` or `Network Load Balancer`と連携
                                    - さらにアノテーションを付与することでローカルIPアドレスをつかったエンドポイントの作成も可能
                                        - service.bata.kubernetes.io/   aws-load-balancer-internal: 0.0.0/0
                                    - CLB or NLBの選択
                                        - service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
                    - 正式に組み込まれる前の機能の設定を行う
        - ラベル
            - リソースの管理(分別)に利用するメタデータ
                - 開発者が利用するラベル
                - システムが利用するラベル
                    - プロジェクト
                    - アプリケーションの種類
                        - app.kubernetes.io/name
                        - app.kubernetes.io/component (役割)
                        - app.kubernetes.io/part-of  
                    - アプリケーションのバージョン
                        - app.kubernetes.io/version
                    - 環境
    - ローカルマニフェストとKubernetes上の登録情報の差分取得
        - `kubectl diff -f sample-pod.yaml`
    - 利用可能なリソース種別の一覧
        - `kubectl api-resources`
        - Namespaceレベルのリソース
            Podリソース, PersisteneVolumeClaimリソース
        - Clusterレベルのリソース
            Nodeリソース、PersistentVlumeリソース
            すべてのNamespaceから共用して利用できる
    - リソースの情報取得
        - Podの一覧を表示
        `kubectl get pods`
        - 特定のPodの情報だけを表示
        `kubectl get pod sample-pod`
        - `-l` ラベルでフィルタリングできる
        - `-o` JSON/YAML など様々な形式で出力可能
        `wide` を指定すると詳細な情報を表示
        `kubectl get pods -o wide`
        `json` `yaml` で出力した場合、より詳細な情報を出力可能
        - ノードの一覧表示
            `kubectl get nodes`
        - allを指定することでほぼすべての種類のリソースを表示
            `kubectl get all`
            Secret / ConfigMap / Ingress などよく利用するのに表示さないリソースも存在する
    - リソースの詳細情報の取得　describe
        - `kubectl describe pod sample-pod`
        - `kubectl describe node xxxxx`
    - 実際のリソースの使用量の確認 top
        `kubectl describe`コマンドで確認できるリソースの使用量はkubernetesがPodに確保した値を示している
        実際にPod内のコンテナが使用しているリソースの使用量は`kubectl top`で確認可能
        リソースの使用量はノードとPod単位で確認する
        - `kubectl top node`
            ノードのリソース使用量を確認
        - `kubectl top pod`
            Podのリソースを確認
            `--containers`オプションを使ってコンテナおとのリソース使用量を確認することも可能 
    - コンテナ上でのコマンドの実行（exec）
        - `kubectl exec　-it sample-pod --/bin/ls`
        - /bin/bash など標準入出力が必要なプログラムなどでは「-i -t」オプションを付与する
            - `-t` 擬似端末を生成
            - `-i` 標準入力をパススルー
            しながら/bin/sh を起動することで、コンテナにSSHログインしているような状態となる
        - `kubectl exec -it sample-pod -- /bin/bash`
    - Pod上にデバッグ用の一時的なコンテナの追加（debug）
        - 軽量なイメージとして軽量なDistrolessやScratchイメージなどを利用している場合、`kubectl exec`コマンドを利用してコンテナの中に入ってもデバッグは困難
        - その場合に`kubectl debug`コマンドを利用する
            - Podに追加の一時的なコンテナ（Ephemeral Container）を起動
            - そのコンテナを使ってデバッグやトラブルシューティングを行う
            - `kubectl debug sample-pod --image=amsy810/tools:v2.0 -it -- bash`
                - sample-podに任意のコマンドでデバッグ用コンテナを起動して接続
    - ローカルマシンからPodへのポートフォワーディング（port-forward）
        - `kubectl port-forward sample-pod 8888:80`
            - localhost:8888 宛の通信が指定したPodの80/TCPポートに転送されるようになる
        - pod名ではなく、DeploymentリソースやServiceリソースに紐づくPodに対してポートフォワーディングすることもできる
            - `kubectl port-forward deploment/sample-deployment 8888:80`
            - `kubectl port-forward service/sample-service 8888:80`
        - `kubectl port-forward` 実行中に通信できるPodは常に同一の1つのPodのみ
    - コンテナのログ確認（logs）
        - `kubectl logs`
            - 標準出力と標準エラー出力を確認できる
            - ログは標準出力と標準エラー出力に出力するのがベストプラクティス
    - Sternによるログ確認
        - Pod名を部分一致で検索し、一致したPodがログ表示の対象となる
        - `kubectl logs`でできることはほぼ全てできる。他の機能は以下
            - 各コンテナのログを時系列で表示
            - タイムスタンプの表示 `--timestamps`
            - 特定のラベルが付与されたPodのログのみ表示 `--selector`
            - 除外するログを正規表現で指定可能 `--exclude`
    - コンテナとローカルマシン間でのファイルのコピー：cp
        - `kubectl cp sample-pod:etc/hostname ./hostname`
            sample-pod内の/etc/hostnameファイルをローカルにコピー
        - `kubectl cp hostname sample-pod:/tmp/newfile`
            ローカルファイルをコンテナにコピー
    - kubectl のサブコマンド拡張プラグイン
        - krewというプラグインマネージャによる管理が推奨されている
            - krew サブコマンドのインストールが必要            
        - krew に対応したプラグインはkubernetes-sigs/krew-indexリポジトリにリスト化されている
    - kubectlにおけるデバッグ
        - kubectlはKubernetes Master常のAPIと通信を行うことでクラスタ管理を実施している
        - `-v`オプションでログレベルを指定し、処理内容を詳細にみることができる
        - HTTP Request/Response の概要を表示する場合は、 `v=6`以降のログレベルを出力すること
            - `kubectl -v=6 get pod`
        - Response Body / Response Body まで確認する場合は `v=8`以降のログレベル
            - `kubectl -v=8 apply -f sample-pod.yaml`
        - リソースの情報を書き換える際はHTTP PATCHリクエストを使用し、Kubernetes独自のStrategic Merge Patchの処理が行われている
    - その他のTIPS
        - alias
            - kubectl-aliases
                - https://github.com/ahmetb/kubectl-aliases
        - kube-psl
            現在操作しているKubernetesクラスタ、Namespaceを表示する
        - Podが起動しない場合のデバッグ
            `kubectl logs`を使用し、コンテナが出力しているログを確認する方法
            `kubectl describe`を使用し、Eventsの項目を確認する方法
            `kubectl run`コマンドを使用し、コンテナ上のシェルで各院する方法

# 第５章 workloads APIs カテゴリ
## 5.1 Workload APIs カテゴリの概要
クラスタ上にコンテナを起動させるのに利用するリソース
- Pod (Tier1)
- ReplicationController (Tier2)
- ReplicaSet (Tier2)
- Deployment (Tier1)
- DaemonSet (Tier2)
- StatefulSet (Tier2)
- Job (Tier2)
- CronJob (Tier1)
## 5.2 Pod
- Workloadsリソースの最小単位。Kubernetesの最小単位でもある
- 1つ以上のコンテナから構成される
- 同じPodに含まれるコンテナ同士はネットワーク的に隔離されておらず、IPアドレスを共有している
    - 2つのコンテナが入ったPodを作成した場合、この2つのコンテナは同一IPアドレスをもつ
    - そのためPod内のコンテナはお互いにlocalhost宛で通信可能
    - Pod単位でIPAddressが割り当てられる（1Podに1NIC）
- 多くの場合、１つのPodに1つのコンテナを内包する
- メインコンテナを補助するサブコンテナを含めるように1つのPodに複数のコンテナをない方することもある
    - サブコンテナの例：
        - プロキシの役割をするコンテナ
        - 設定値の動的な書き換えを行うコンテナ
        - ローカルキャッシュ用のコンテナ
        - SSL終端用のコンテナ
    - Podのデザインパターンがある
    - nginxコンテナとredisコンテナのようなメインのコンテナを一つのPodに同居させるような構成は個々のPodの移動容易性が失われ複雑化するため推奨されない
- Podのデザインパターン
    ３つある
    - サイドカーパターン
        - メインのコンテナに加えて補助的な機能を追加するコンテナをない方したパターン
            - 特定の変更を検知した際に動的に設定を変更するコンテナ
            - Gitリポジトリとローカルストレージを同期するコンテナ
            - アプリケーションのログファイルをオブジェクトストレージに転送するコンテナ
    - アンバサダーパターン：外部システムとのやり取りの代理を行う
        - メインのコンテナが外部のシステムと接続する際に代理で中継を行うコンテナ（アンバサダーコンテナ）を内包したパターン
            - シャーディングされたデータベースに接続する場合
                - メインから分割されたデータベースの一つを選択して接続すると、DBとの結合度が高くなる
                - アンバサダーコンテナを使用してメインのコンテナからは常にlocalhostを指定してアンバサダーコンテナへ接続し、アンバサダーコンテナが複数の接続先に中継して接続するように構成しておく
                    - アプリ側からみると一つの接続先に接続しているようになる
    - アダプタパターン：外部からのアクセスのインターフェースとなる
        - 外部からのリクエストに対して差分を吸収するコンテナ（アダプタ）をない方したパターン
            - MySQLのメトリクスの出力フォーマットをPrometheusの形式に成形する
- Podの作成
    - Podはネットワークの名前空間を共有している（同一IPアドレス）ため、通常のVM常で80/TCPポートをbindするサービスを2つ起動できない。
    - コンテナへのログイン
        - kubectl exec -it sample-pod -- /bin/bash
            - 擬似端末を生成し（-t）、標準入力をパススルー（-i）しながら /bin/bashを起動する
    - Pod名に_は使えない
    - Podに割り当てられるIPアドレス
        - Kubernetes Node のホストのIPアドレスとはレンジが異なる
        - 外部からは疎通性がないIPアドレスが払い出される
        - ホストのネットワークを利用する設定（spec.hostNetwork）を有効化することで、ホスト常でただプロセスを起動するのと同じネットワーク構成でPodを起動させることができる
            - hostNetworkを利用したPodはポート番号を衝突させられないため基本的には利用しない
            - NodePort Serviceなどで解決できないか検討すること

            

