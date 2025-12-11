# Ansible ロール: Mellanox OFED ドライバー

このロールは、Red Hat 系、Debian/Ubuntu 系、および Proxmox VE ベースのシステムに
Mellanox（NVIDIA）の Open‑Fabrics Enterprise Distribution (OFED) ドライバーを
インストールし設定します。`ktooi/ansible‑role‑crio` ロールと同じディレクトリ
レイアウトとモジュール構造に従っており、タスクや変数、ハンドラーが分離され
拡張しやすくなっています。また、カーネル更新、SR‑IOV、vDPA 設定に関する
追加機能を備えています。対象は現在の長期サポート (LTS) リリースである
Ubuntu 20.04/22.04/24.04、Debian 11/12、Proxmox 7/8、Red Hat 7/8/9
（CentOS、Rocky、Alma などのクローンを含む）です。

## 前提条件

* Python 3.6 以上を備えた Ansible コントローラー。
* 対象ホストは SSH で到達可能で、特権昇格 (sudo) が有効になっている必要があります。
* Red Hat 系ディストリビューションでは DNF を利用するため `python3‑dnf` パッケージが必要です。

## ロール変数

ほとんどの変数には適切なデフォルト値が設定されているため、特別な設定をしなくてもロールを実行できます。必要に応じてプレイブックやインベントリで変数を上書きしてください。

| 変数 | デフォルト | 説明 |
|------|----------|------|
| `ofed_version` | `"latest-24.10"` | 使用する Mellanox OFED リポジトリのリリース。デフォルトでは 24.10 の長期サポート（LTS）ブランチの最新サブリリース（例: 24.10‑3.2.5.0）をターゲットにします。NVIDIA によれば、24.10 は最後のスタンドアロン MLNX_OFED LTS リリースであり、3 年間の更新が提供されます。 |
| `ofed_repository_url` | `"https://linux.mellanox.com/public/repo/mlnx_ofed"` | Mellanox 公開リポジトリのベース URL。ベンダーのドキュメントでは、このリポジトリから `.repo`/`.list` ファイルをダウンロードする方法を推奨しています。 |
| `ofed_target_release` | *空* | リポジトリパス内のディストリビューション名を明示的に指定する場合に使用します（例: `rhel7.9`、`ubuntu22.04`）。未設定の場合は `ansible_distribution` と `ansible_distribution_version` から自動的に算出します。 |
| `ofed_update_kernel` | `true` | Mellanox ドライバーに対応したカーネルをインストールして起動させるかどうか。Red Hat 系ではデフォルトで有効となっており、GRUB を更新します。 |
| `ofed_nobest` | `false` | Red Hat 系でカーネルをインストールする際、`dnf` に `--nobest` オプションを渡すかどうか。 |
| `ofed_grub_backup` | `true` | GRUB 設定を変更する前に既存設定のバックアップを作成するかどうか（Red Hat 系のみ）。 |
| `ofed_manage_openib_conf` | `false` | `/etc/infiniband/openib.conf` を管理するかどうか。`true` の場合、テンプレートを展開し `openibd` サービスを再起動します。 |
| `ofed_force_mode` | `false` | `openib.conf` 内の `FORCE_MODE` フラグを設定し、他のドライバーが存在してもドライバースタックをロードさせます。 |
| `ofed_rdma_ucm_load` | `false` | `openib.conf` 内の `RDMA_UCM_LOAD` フラグを設定します。 |
| `ofed_remove_distro_packages` | `true` | ディストリビューションが提供する競合する RDMA/InfiniBand パッケージを削除します。Debian/Ubuntu/Proxmox システムでインストール前にこれらのパッケージを purge することが NVIDIA によって推奨されています。 |
| `ofed_enable_sriov` | `false` | Mellanox デバイスで SR‑IOV を有効にします。`true` にすると IOMMU カーネルオプションを追加し、`mlxconfig` で NIC ファームウェアを設定し、指定されたインターフェースに仮想関数 (VF) を作成します。 |
| `ofed_sriov_interfaces` | `[]` | SR‑IOV を有効にする物理インターフェース名または PCI アドレスのリスト。各インターフェースには `ofed_sriov_num_vfs` で指定された数の VF を作成しますが、`ofed_sriov_num_vfs_map` で個別に上書きできます。 |
| `ofed_sriov_num_vfs` | `8` | SR‑IOV が有効な場合に、各インターフェースに作成する仮想関数のデフォルト数。 |
| `ofed_sriov_num_vfs_map` | `{}` | インターフェース名や PCI アドレスをキーに VF 数を個別指定するマッピング。 |
| `ofed_sriov_iommu_options` | `"intel_iommu=on iommu=pt pci=realloc"` | SR‑IOV 有効時に GRUB に追加するカーネルパラメータ。AMD システムでは適宜変更してください。 |
| `ofed_enable_vdpa` | `false` | vDPA (virtio Data Path Acceleration) を有効にします。`true` の場合、`vhost_vdpa` モジュールをロードし、eswitch モードを設定し、vDPA デバイスを作成します。vDPA は SR‑IOV に依存するため、`ofed_enable_sriov` が `false` でも vDPA を有効にすると SR‑IOV 設定が実行されます。 |
| `ofed_vdpa_eswitch_mode` | `"switchdev"` | vDPA 有効時に物理 NIC に設定する eswitch モード（ハードウェアオフロードには通常 `switchdev` を使用します）。 |
| `ofed_vdpa_pf_devices` | `[]` | vDPA を有効にする物理関数 (PF) の PCI アドレスのリスト。空の場合は `ofed_sriov_interfaces` から派生した PF を使用します。 |
| `ofed_vdpa_devices` | `[]` | 作成する vDPA デバイスのリスト。各エントリでは `name`、管理対象 VF の PCI アドレスである `mgmtdev`、任意の `mac` を指定します。 |

OS ごとの追加変数は `vars/` ディレクトリに定義されており、インストールするパッケージや削除する競合パッケージなどを指定します。詳細は各ファイルを参照してください。

## 依存関係

外部ロールへの依存はありませんが、対象プラットフォームに応じて `apt`、`yum`、`dnf` などのモジュールを利用します。

## プレイブック例

```yaml
---
- hosts: all
  become: true
  roles:
    - role: ansible-role-ofed
      vars:
        ofed_version: "latest-24.10"
        ofed_manage_openib_conf: true
        ofed_force_mode: true
        ofed_rdma_ucm_load: false
```

この例では、Mellanox OFED ドライバーを 24.10 LTS ブランチの最新リリースからインストールし、対応するカーネルをインストールし、`/etc/infiniband/openib.conf` を管理します。変数は必要に応じて上書きできます。詳細は上記の表を参照してください。

## メモ

* **リポジトリファイル**: ロールは Mellanox 公開ミラーから適切なリポジトリファイル (`.repo` は Red Hat 系、`.list` は Debian/Ubuntu 系) をダウンロードします。これはベンダーが推奨する方法です。
* **競合パッケージの削除**: Debian/Ubuntu/Proxmox システムでは、インストール前に `libipathverbs1`、`librdmacm1`、`libibverbs1`、`libmthca1`、`openmpi-bin`、`ibverbs-utils`、`infiniband-diags`、`ibutils`、`perftest` などのパッケージを purge します。引用している行はこれらのパッケージを削除する `apt-get` コマンドを示しており、バージョン指定ではありません。これを無効にしたい場合は `ofed_remove_distro_packages` を false に設定してください。
* **カーネル更新**: Red Hat 系ディストリビューションでは、Mellanox の `kmod-mlnx-ofa_kernel` パッケージに一致する `kernel` パッケージを検出し、インストールします。`ofed_update_kernel` が有効な場合、該当するカーネルをインストールし、GRUB のデフォルトエントリを更新します。
* **openib.conf の管理**: `ofed_manage_openib_conf` が true の場合、Jinja2 テンプレートを `/etc/infiniband/openib.conf` に展開し、`FORCE_MODE` や `RDMA_UCM_LOAD` を変数に従って設定します。ファイルが変更された場合は `openibd` サービスを再起動します。
* **Proxmox VE サポート**: Proxmox ホストは Debian 派生として扱われます。`ansible_distribution` が `Proxmox VE` に一致する場合、自動的に Debian 用のパッケージマネージャとリポジトリ設定が使用されます。
* **SR‑IOV サポート**: `ofed_enable_sriov` を true にすると SR‑IOV を有効にします。`mlxconfig` でファームウェアフラグを設定し、GRUB に IOMMU オプションを追加し、各指定インターフェースに VF を作成します。全インターフェースの VF 数は `ofed_sriov_num_vfs` で指定できますが、個別に `ofed_sriov_num_vfs_map` で上書きできます。必要なツール (`iproute2` など) をインストールし、変更が必要な場合は再起動が必要です。
* **vDPA サポート**: `ofed_enable_vdpa` を true にすると、ハードウェア vDPA 加速を設定します。`vhost_vdpa` モジュールをロードし、`devlink` で Mellanox NIC の eswitch モードを `switchdev` に設定し、必要に応じて `ofed_vdpa_devices` で指定された vDPA デバイスを作成します。各デバイスには名前、管理対象 VF の PCI アドレス、および任意の MAC アドレスを指定できます。vDPA は SR‑IOV に依存するため、`ofed_enable_vdpa` が true の場合は `ofed_sriov_interfaces`（または `ofed_vdpa_pf_devices`）を必ず指定し、ロールが必要な仮想関数を作成できるようにしてください。これにより virtio‑vDPA 加速 VM のデプロイ基盤を提供します。

## ライセンス

Apache‑2.0

## 作者情報

* **Kodai Tooi** [GitHub](https://github.com/ktooi), [Qiita](https://qiita.com/ktooi)