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
