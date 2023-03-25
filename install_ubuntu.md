# kubeadmのインストール
## 始める前に
- 次のいずれかが動作しているマシンが必要です
  - Ubuntu 16.04+
  - Debian 9+
  - CentOS 7
  - Red Hat Enterprise Linux (RHEL) 7
  - Fedora 25+
  - HypriotOS v1.0.1+
  - Container Linux (tested with 1800.6.0)
  
```
cat /etc/os-release
```

- 1台あたり2GB以上のメモリ(2GBの場合、アプリ用のスペースはほとんどありません)
```
cat /proc/meminfo 
```
  
- 2コア以上のCPU
```
cat /proc/cpuinfo
```

- クラスター内のすべてのマシン間で通信可能なネットワーク(パブリックネットワークでもプライベートネットワークでも構いません)
```
ifconfig -a
```


- ユニークなhostname、MACアドレス、とproduct_uuidが各ノードに必要です。詳細はここを参照してください。
```
uname -n
sudo cat /sys/class/dmi/id/product_uuid
```

- マシン内の特定のポートが開いていること。詳細はここを参照してください。
```
nc 127.0.0.1 6443
```

- Swapがオフであること。kubeletが正常に動作するためにはswapは必ずオフでなければなりません。
```
swapon -s
```

# コンテナランタイムのインストール
## 前提
### IPv4フォワーディングを有効化し、iptablesからブリッジされたトラフィックを見えるようにする
```
# 事前確認
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward

# 設定
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# この構成に必要なカーネルパラメーター、再起動しても値は永続します
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 再起動せずにカーネルパラメーターを適用
sudo sysctl --system

# 事後確認
# br_netfilterとoverlayモジュールが読み込まれていることを確認する
lsmod | grep br_netfilter
lsmod | grep overlay

# カーネルパラメーターが1に設定されていることを確認する
# net.bridge.bridge-nf-call-iptables、
# net.bridge.bridge-nf-call-ip6tables、
# net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

# containerdインストール
## 準備
### aptリポジトリをセットアップする
#### パッケージ インデックスを更新しapt、パッケージをインストールして、aptHTTPS 経由でリポジトリを使用できるようにします。
```
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg
```

#### Docker の公式 GPG キーを追加します。
```
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

#### 次のコマンドを使用して、リポジトリをセットアップします。
```
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

#### パッケージ インデックスを更新しますapt。
```
sudo apt-get update
```

#### containerdをインストールします
```
sudo apt-get install containerd.io
```

#### containerdサービスの起動を確認する
```
systemctl status containerd
```

# cgroupドライバー
## 準備
```
cgroupドライバーはデフォルトでcgroupfsとなっている。
initシステムがsystemdの場合、systemd cgroupドライバーに変更する必要があるる。
また、コンテナランタイムがsystemd cgroupドライバーを使用する場合、
kubeletもsystemd cgroupドライバーを使用する必要がある。一致させなければならない。
```

#### initシステム確認
```
ps -p 1
```

#### systemd cgroupドライバーを構成する
```
# ファイル内を空にする
sudo cp -p  /etc/containerd/config.toml  /etc/containerd/config.toml.bak
sudo cp -p  /dev/null  /etc/containerd/config.toml
cat  /etc/containerd/config.toml
```

以下を追記する
```
cat <<EOF | sudo tee /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
EOF
```
確認する
```
cat  /etc/containerd/config.toml
```
containerdサービスを再起動する
```
systemctl status containerd
sudo systemctl restart containerd
systemctl status containerd
```

# kubeadm、kubelet、kubectlのインストール
aptのパッケージ一覧を更新し、Kubernetesのaptリポジトリを利用するのに必要なパッケージをインストールします:
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

Google Cloudの公開鍵をダウンロードします:
```
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

Kubernetesのaptリポジトリを追加します:
```
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

```

aptのパッケージ一覧を更新し、kubelet、kubeadm、kubectlをインストールします。そしてバージョンを固定します:
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

# コントロールプレーンノードの初期化(マスターノードのみ)
IPアドレスを確認する
```
ip addr show
```

コントロールプレーンノードを初期化する
コマンドの実行結果に作業継続に必要なコマンドが表示されるので、消さないように注意する
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16  --apiserver-advertise-address=<IPアドレス>
```

例：コマンドの実行結果
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.56.2:6443 --token pfa6r3.g6hi6n90gu61vl5p \
        --discovery-token-ca-cert-hash sha256:a48f08297c34026cfff713c200a32dcc959d9555fe9ed26f426a17e943e1ad4e
```

クラスターを開始する
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

kubectlコマンドが実行できることを確認する
```
kubectl get pod -A
```


# ネットワークアドオンのインストール(マスターノードのみ)
## Weave Netのインストール
```
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

weave netのpodが作成されていることを確認する
```
kubectl get pod -A
```

デーモンセット：Weave Netの設定を変更する
spec.template.spec.containers.env下に以下を入力する
```
      containers:
        - name: weave
          env:
            - name: IPALLOC_RANGE
              value: 10.244.0.0/16
```
```
kubectl get ds weave-net -n kube-system
kubectl edit ds weave-net -n kube-system
kubectl get ds weave-net -n kube-system
```

AGE欄を参照し、weave netポッドがデプロイされたことを確認する
```
kubectl get pod -A
```

マスターノードでクラスターの状態を確認する
```
kubectl get nodes
```

ワーカーノードをクラスターに追加する
```
sudo kubeadm join 192.168.56.2:6443 --token pfa6r3.g6hi6n90gu61vl5p \
        --discovery-token-ca-cert-hash sha256:a48f08297c34026cfff713c200a32dcc959d9555fe9ed26f426a17e943e1ad4e
```

マスターノードでクラスターの状態を確認する
```
kubectl get nodes
```

ポッドをデプロイできることを確認する
```
kubectl get pod
kubectl run nginx --image=nginx
kubectl get pod
kubectl delete pod nginx
kubectl get pod
```
