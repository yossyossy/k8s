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

# ランタイムのインストール


