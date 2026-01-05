## 清除集群

#### 清除节点

停止节点：`kubeadm reset -f`

移除kubelet服务

```shell
systemctl stop kubelet
systemctl disable kubelet
rm -rf  /etc/systemd/system/kubelet.service
rm -rf  /etc/systemd/system/kubelet.service.d
rm -rf /usr/local/bin/kube*
```



#### 停止&清除etcd

目前etcd部署在=10.160.144.93，10.160.144.96，10.160.144.111

```shell
systemctl stop etcd
systemctl disable etcd
rm -rf /var/lib/etcd
rm -f /usr/local/bin/etcd
rm -f /usr/local/bin/etcdctl
rm -f /etc/systemd/system/etcd.service
```



#### 删除镜像

```shell
docker rmi -f $(docker images -qa) 
systemctl stop docker
systemctl disable docker
rm -f  /etc/systemd/system/docker.service
rm -rf  /etc/systemd/system/docker.service.d
rm -f /usr/local/bin/docker
rm -f /usr/bin/docker
```

### 清除网卡

安装前，确保没有已经存在的虚拟网卡，例如Flannel的，cilium的额

使用ifconfig查看

```shell
ifconfig cilium_vxlan down
ifconfig cilium_net down
ifconfig cilium_host down
ifconfig docker0 down
ip link delete cilium_vxlan
ip link delete cilium_net
ip link delete cilium_host
ip link delete docker0
```



## 状态设置

#### 网络设置

##### 设置DNS：

```
cat << EOF > /etc/resolv.conf
nameserver 10.72.255.100
EOF
```

##### 设置yum

```shell
仓库设置：
cat << EOF > /etc/yum.repos.d/EulerOS.repo
[root@localhost yum.repos.d]# more EulerOS.repo
[base]
name=EulerOS-2.0SP10 base
baseurl=http://his-mirrors.huawei.com/install/euleros/2.10/os/x86_64/
enabled=1
gpgcheck=1
gpgkey=http://his-mirrors.huawei.com/install/euleros/2.10/os/RPM-GPG-KEY-EulerOS
EOF

设置yum代理：
cat << EOF >  /etc/yum.conf
[main]
gpgcheck=1
installonly_limit=3
clean_requirements_on_remove=True
best=True
skip_if_unavailable=False
proxy=http://proxy.huawei.com:8080/
proxy_username=username
proxy_password=password
EOF
```

##### 设置代理

```shell
export http_proxy=http://lwx639077:No6Before_@proxyhk.huawei.com:8080
export https_proxy=http://lwx639077:No6Before_@proxyhk.huawei.com:8080
export ftp_proxy=http://lwx639077:No6Before_@proxyhk.huawei.com:8080
export no_proxy=127.0.0.1,localhost,.huawei.com
###或者加入/etc/profile文件
```

设置服务器名和hosts

服务器列表如下：

![image-20220910131615778](http://image.huawei.com/tiny-lts/v1/images/ce3e0b9b14094be38b3963f08cb96823_186x196.png@900-0-90-f.png)



以便可以通过服务器名可以访问集群内部所有网络

```shell
##在各个设备上执行
hostname #######
echo "#######" > /etc/hostname


cat << EOF > /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

#####Kubernetes node#####
# SH master node
10.160.144.93 shmaster1
10.160.144.96 shmaster2
10.160.144.111 shmaster3
# BJ work node
10.23.30.206 bjwork1
10.23.30.208 bjwork2
# CD work node
10.148.136.62 cdwork1
10.148.136.63 cdwork2
10.148.151.120 cdwork3

#####repository#####
10.160.58.161 wirepo.td-tech.com
10.3.253.54 his-mirrors.huawei.com
EOF
```



#### 关闭交换分区

执行：`/sbin/swapoff -a` 命令，确保`/sbin/swapon -s`， 如果没有显示结果

或者 执行 `sed -i 's/.*swap.*/#&/' /etc/fstab` 关闭交换区永久生效

#### 关闭防火墙

生产环境不执行此项，开发调试环境可以关闭

`systemctl stop firewalld.service` 

`systemctl disable firewalld.service`

#### 安装必要linux组件

确保 socat， conntrack-tools已经安装.

提前复制文件：libwrap0-7.6-1.433.x86_64.rpm，socat-2.0.0-0.b9.9.mga8.x86_64.rpm

```shell
rpm -ivh libwrap0-7.6-1.433.x86_64.rpm
rpm -ivh socat-2.0.0-0.b9.9.mga8.x86_64.rpm
yum -y install conntrack-tools
```

执行 `yum -y install socat conntrack-tools`，前提yum配置已经完成

检查安装的组件：

```
yum list installed|grep socat
yum list installed|grep conntrack
```



## 组件准备

kubernetes 1.25.0 ：https://storage.googleapis.com/kubernetes-release/release/v1.25.0/kubernetes-server-linux-amd64.tar.gz

etcd 3.5.4： https://github.com/etcd-io/etcd/releases/download/v3.5.4/etcd-v3.5.4-linux-amd64.tar.gz

containerd 1.6.8：https://github.com/containerd/containerd/releases/download/v1.6.8/containerd-1.6.8-linux-amd64.tar.gz

## 前置安装

### ETCD安装

service配置文件，使用已经有的证书

```shell
### /etc/systemd/system/etcd.service
[Unit]
Description=etcd
After=network.target

[Service]
Type=notify
User=root
EnvironmentFile=/etc/etcd.env
ExecStart=/usr/local/bin/etcd

NotifyAccess=all
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target

### /etc/etcd.env
# Environment file for etcd v3.5.4

# [member]
ETCD_NAME=master1
ETCD_DATA_DIR=/var/lib/etcd
ETCD_LISTEN_PEER_URLS=https://10.160.144.93:2380                            ### 其他节点配置对应IP
ETCD_LISTEN_CLIENT_URLS=https://10.160.144.93:2379,https://127.0.0.1:2379　　### 其他节点配置对应IP

# [cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://10.160.144.93:2380　　　　　　　　　　　### 其他节点配置对应IP
ETCD_INITIAL_CLUSTER=shmaster1=https://10.160.144.93:2380,shmaster2=https://10.160.144.96:2380,shmaster3=https://10.160.144.111:2380,
ETCD_INITIAL_CLUSTER_STATE=new                  ######第一个节点为new，剩余两个节点为existing
ETCD_INITIAL_CLUSTER_TOKEN=k8s_etcd
ETCD_ADVERTISE_CLIENT_URLS=https://10.160.144.93:2379　　　　　　　　　　　　　　　### 其他节点配置对应IP

# [security]
ETCD_TRUSTED_CA_FILE=/etc/ssl/etcd/ssl/ca.pem
ETCD_CERT_FILE=/etc/ssl/etcd/ssl/crt.pem
ETCD_KEY_FILE=/etc/ssl/etcd/ssl/key.pem
ETCD_CLIENT_CERT_AUTH=true

ETCD_PEER_TRUSTED_CA_FILE=/etc/ssl/etcd/ssl/ca.pem
ETCD_PEER_CERT_FILE=/etc/ssl/etcd/ssl/crt.pem
ETCD_PEER_KEY_FILE=/etc/ssl/etcd/ssl/key.pem
ETCD_PEER_CLIENT_CERT_AUTH=true

# CLI settings
ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
ETCDCTL_CACERT=/etc/ssl/etcd/ssl/ca.pem
ETCDCTL_CERT=/etc/ssl/etcd/ssl/crt.pem
ETCDCTL_KEY=/etc/ssl/etcd/ssl/key.pem


ETCD_ELECTION_TIMEOUT=5000
ETCD_HEARTBEAT_INTERVAL=250
ETCD_PROXY=off
ETCD_AUTO_COMPACTION_RETENTION=8

## 启动服务
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
```

校验etcd是否安装成功

```shell
[root@shmaster1 system]# etcdctl endpoint --cluster health
https://10.160.144.96:2379 is healthy: successfully committed proposal: took = 13.411197ms
https://10.160.144.93:2379 is healthy: successfully committed proposal: took = 13.07335ms
https://10.160.144.111:2379 is healthy: successfully committed proposal: took = 14.050796ms

[root@shmaster1 system]# etcdctl endpoint --cluster status -w table
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+
|          ENDPOINT           |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX |
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+
|  https://10.160.144.93:2379 | 5a5f8e1856269a0a |   3.5.4 |   20 kB |     false |      false |         2 |         13 |
| https://10.160.144.111:2379 | 68afbdf979002be6 |   3.5.4 |   20 kB |     false |      false |         2 |         13 |
|  https://10.160.144.96:2379 | 784e2cc135a3ccc0 |   3.5.4 |   20 kB |      true |      false |         2 |         13 |
+-----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+
```

### 安装containerd

复制可执行文件

containerd,containerd-shim,containerd-shim-runc-v1,containerd-shim-runc-v2,containerd-stress,crictl,critest,ctr

到/usr/local/bin, 并添加可执行权限

验证是否安装成功

```shell
crictl images
crictl pull wirepo.td-tech.com/cnp/busybox
```

准备containerd.service文件

```shell
cat /etc/systemd/system/containerd.service
# Copyright The containerd Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

Environment="HTTPS_PROXY=http://lwx639077:No6Before_@proxy.huawei.com:8080/"
Environment="HTTP_PROXY=http://lwx639077:No6Before_@proxy.huawei.com:8080/"
Environment="NO_PROXY=localhost,127.0.0.1,172.17.0.1/16,10.148.0.0/16,192.167.0.0/24,10.0.0.0/8,wirepo.td-tech.com/"
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```

设置containerd配置文件

```shell
mkdir /etc/containerd
containerd config default > /etc/containerd/config.toml
```

修改文件如下:

```
disabled_plugins = []
imports = []
oom_score = 0
plugin_dir = ""
required_plugins = []
root = "/var/lib/containerd"
state = "/run/containerd"
temp = ""
version = 2

[cgroup]
  path = ""

[debug]
  address = ""
  format = ""
  gid = 0
  level = ""
  uid = 0

[grpc]
  address = "/run/containerd/containerd.sock"
  gid = 0
  max_recv_message_size = 16777216
  max_send_message_size = 16777216
  tcp_address = ""
  tcp_tls_ca = ""
  tcp_tls_cert = ""
  tcp_tls_key = ""
  uid = 0

[metrics]
  address = ""
  grpc_histogram = false

[plugins]

  [plugins."io.containerd.gc.v1.scheduler"]
    deletion_threshold = 0
    mutation_threshold = 100
    pause_threshold = 0.02
    schedule_delay = "0s"
    startup_delay = "100ms"

  [plugins."io.containerd.grpc.v1.cri"]
    device_ownership_from_security_context = false
    disable_apparmor = false
    disable_cgroup = false
    disable_hugetlb_controller = true
    disable_proc_mount = false
    disable_tcp_service = true
    enable_selinux = false
    enable_tls_streaming = false
    enable_unprivileged_icmp = false
    enable_unprivileged_ports = false
    ignore_image_defined_volumes = false
    max_concurrent_downloads = 3
    max_container_log_line_size = 16384
    netns_mounts_under_state_dir = false
    restrict_oom_score_adj = false
    sandbox_image = "wirepo.td-tech.com/cnp/pause:3.8"
    selinux_category_range = 1024
    stats_collect_period = 10
    stream_idle_timeout = "4h0m0s"
    stream_server_address = "127.0.0.1"
    stream_server_port = "0"
    systemd_cgroup = false
    tolerate_missing_hugetlb_controller = true
    unset_seccomp_profile = ""

    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
      conf_template = ""
      ip_pref = ""
      max_conf_num = 1

    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "runc"
      disable_snapshot_annotations = true
      discard_unpacked_layers = false
      ignore_rdt_not_enabled_errors = false
      no_pivot = false
      snapshotter = "overlayfs"

      [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
        base_runtime_spec = ""
        cni_conf_dir = ""
        cni_max_conf_num = 0
        container_annotations = []
        pod_annotations = []
        privileged_without_host_devices = false
        runtime_engine = ""
        runtime_path = ""
        runtime_root = ""
        runtime_type = ""

        [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime.options]

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          base_runtime_spec = ""
          cni_conf_dir = ""
          cni_max_conf_num = 0
          container_annotations = []
          pod_annotations = []
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_path = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"

          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            BinaryName = ""
            CriuImagePath = ""
            CriuPath = ""
            CriuWorkPath = ""
            IoGid = 0
            IoUid = 0
            NoNewKeyring = false
            NoPivotRoot = false
            Root = ""
            ShimCgroup = ""
            SystemdCgroup = true

      [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime]
        base_runtime_spec = ""
        cni_conf_dir = ""
        cni_max_conf_num = 0
        container_annotations = []
        pod_annotations = []
        privileged_without_host_devices = false
        runtime_engine = ""
        runtime_path = ""
        runtime_root = ""
        runtime_type = ""

        [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime.options]

    [plugins."io.containerd.grpc.v1.cri".image_decryption]
      key_model = "node"

    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://registry-1.docker.io"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.aliyuncs.com"]
          endpoint = ["https://registry.aliyuncs.com"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."wirepo.td-tech.com"]
          endpoint = ["https://wirepo.td-tech.com"]
        [plugins."io.containerd.grpc.v1.cri".registry.configs]
          [plugins."io.containerd.grpc.v1.cri".registry.configs."wirepo.td-tech.com".tls]
            insecure_skip_verify = true

    [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""

  [plugins."io.containerd.internal.v1.opt"]
    path = "/opt/containerd"

  [plugins."io.containerd.internal.v1.restart"]
    interval = "10s"

  [plugins."io.containerd.internal.v1.tracing"]
    sampling_ratio = 1.0
    service_name = "containerd"

  [plugins."io.containerd.metadata.v1.bolt"]
    content_sharing_policy = "shared"

  [plugins."io.containerd.monitor.v1.cgroups"]
    no_prometheus = false

  [plugins."io.containerd.runtime.v1.linux"]
    no_shim = false
    runtime = "runc"
    runtime_root = ""
    shim = "containerd-shim"
    shim_debug = false

  [plugins."io.containerd.runtime.v2.task"]
    platforms = ["linux/amd64"]
    sched_core = false

  [plugins."io.containerd.service.v1.diff-service"]
    default = ["walking"]

  [plugins."io.containerd.service.v1.tasks-service"]
    rdt_config_file = ""

  [plugins."io.containerd.snapshotter.v1.aufs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.btrfs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.devmapper"]
    async_remove = false
    base_image_size = ""
    discard_blocks = false
    fs_options = ""
    fs_type = ""
    pool_name = ""
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.native"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.overlayfs"]
    root_path = ""
    upperdir_label = false

  [plugins."io.containerd.snapshotter.v1.zfs"]
    root_path = ""

  [plugins."io.containerd.tracing.processor.v1.otlp"]
    endpoint = ""
    insecure = false
    protocol = ""

[proxy_plugins]

[stream_processors]

  [stream_processors."io.containerd.ocicrypt.decoder.v1.tar"]
    accepts = ["application/vnd.oci.image.layer.v1.tar+encrypted"]
    args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
    env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
    path = "ctd-decoder"
    returns = "application/vnd.oci.image.layer.v1.tar"

  [stream_processors."io.containerd.ocicrypt.decoder.v1.tar.gzip"]
    accepts = ["application/vnd.oci.image.layer.v1.tar+gzip+encrypted"]
    args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
    env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
    path = "ctd-decoder"
    returns = "application/vnd.oci.image.layer.v1.tar+gzip"

[timeouts]
  "io.containerd.timeout.bolt.open" = "0s"
  "io.containerd.timeout.shim.cleanup" = "5s"
  "io.containerd.timeout.shim.load" = "5s"
  "io.containerd.timeout.shim.shutdown" = "3s"
  "io.containerd.timeout.task.state" = "2s"

[ttrpc]
  address = ""
  gid = 0
  uid = 0
```

设置crictl操作

```shell
cat <<EOF> /etc/crictl.yaml 
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```



## Master初始化

#### 复制K8S可执行文件

复制kubeadm, kubelet,kubectl到/usr/local/bin/ 并添加可执行权限

#### 准备所使用的镜像

查看所有需要使用的镜像:

```shell
[root@eLTE63a20729 ~]# kubeadm config images list
registry.k8s.io/kube-apiserver:v1.25.0
registry.k8s.io/kube-controller-manager:v1.25.0
registry.k8s.io/kube-scheduler:v1.25.0
registry.k8s.io/kube-proxy:v1.25.0
registry.k8s.io/pause:3.8
registry.k8s.io/etcd:3.5.4-0
registry.k8s.io/coredns/coredns:v1.9.3
```

将镜像在本地下载到本地,在上传到私库,以registry.k8s.io/pause:3.8 举例

```shell
docker pull registry.k8s.io/pause:3.8
docker tag registry.k8s.io/pause:3.8 wirepo.td-tech.com/cnp/pause:3.8
docker push wirepo.td-tech.com/cnp/pause:3.8
```

#### 安装kubelet

定义kublet服务文件

```shell
# 所有k8s节点配置kubelet service
cat > /etc/systemd/system/kubelet.service << EOF

[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
    --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig  \\
    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
    --config=/etc/kubernetes/kubelet-conf.yml \\
    --container-runtime-endpoint=unix:///run/containerd/containerd.sock  \\
    --node-labels=node.kubernetes.io/node=
    # --feature-gates=IPv6DualStack=true
    # --container-runtime=remote
    # --runtime-request-timeout=15m
    # --cgroup-driver=systemd

[Install]
WantedBy=multi-user.target
EOF
```

kublet配置文件

```sh
cat > /etc/kubernetes/kubelet-conf.yml <<EOF
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: systemd
cgroupsPerQOS: true
clusterDNS:
- 192.101.0.10
clusterDomain: cluster.local
containerLogMaxFiles: 5
containerLogMaxSize: 10Mi
contentType: application/vnd.kubernetes.protobuf
cpuCFSQuota: true
cpuManagerPolicy: none
cpuManagerReconcilePeriod: 10s
enableControllerAttachDetach: true
enableDebuggingHandlers: true
enforceNodeAllocatable:
- pods
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: true
fileCheckFrequency: 20s
hairpinMode: promiscuous-bridge
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 20s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
iptablesDropBit: 15
iptablesMasqueradeBit: 14
kubeAPIBurst: 10
kubeAPIQPS: 5
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
registryBurst: 10
registryPullQPS: 5
resolvConf: /etc/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
volumeStatsAggPeriod: 1m0s
EOF
```

初始化节点：

节点配置：

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.160.144.93
  bindPort: 6443

### abcdef.0123456789abcdef
bootstrapTokens:
  - token: "9a08jv.c0izixklcxtmnze7"
    usages:
      - authentication
      - signing
    groups:
      - system:bootstrappers:kubeadm:default-node-token
nodeRegistration:
  imagePullPolicy: IfNotPresent
  name: shmaster1
  taints: null
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
etcd:
  external:
    endpoints:
      - https://10.160.144.93:2379
      - https://10.160.144.96:2379
      - https://10.160.144.111:2379
    caFile: /etc/ssl/etcd/ssl/ca.pem
    certFile: /etc/ssl/etcd/ssl/crt.pem
    keyFile: /etc/ssl/etcd/ssl/key.pem
controllerManager: { }
imageRepository: wirepo.td-tech.com/cnp
kubernetesVersion: 1.25.0
controlPlaneEndpoint: 10.160.144.93
apiServer:
  certSANS:
    - shmaster1
    - shmaster2
    - shmaster3
    - localhost
    - 192.101.0.1
    - kubernetes
    - kubernetes.default
    - kubernetes.default.svc
networking:
  dnsDomain: cluster.local
  podSubnet: 192.168.0.0/16
  serviceSubnet: 192.101.0.0/16
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd

```



```shell
kubeadm init --config /root/installfiles/kubernetes/1.25.0/kubeadm-config93.yaml

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

装完以后会显示结果如下：

```shell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.h
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join 10.160.144.93:6443 --token 9a08jv.c0izixklcxtmnze7 \
        --discovery-token-ca-cert-hash sha256:e8861900c349b09677cdab4c0a89e2bfd632a500ee5c7834bc00c5f3a90cb1dc \
        --control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.160.144.93:6443 --token 9a08jv.c0izixklcxtmnze7 \
        --discovery-token-ca-cert-hash sha256:e8861900c349b09677cdab4c0a89e2bfd632a500ee5c7834bc00c5f3a90cb1dc

```



## 必备组件

## 加入集群

#### 其他节点准备

拷贝containerd文件，

复制 K8S 证书，切记，不要复制apiserver证书

```shell
scp /etc/kubernetes/pki/ca.* root@****:/etc/kubernetes/pki
scp /etc/kubernetes/pki/sa.* root@****:/etc/kubernetes/pki
scp /etc/kubernetes/pki/front-proxy-ca.* root@***:/etc/kubernetes/pki/
```



## 平台组件

## FAQ

#### containerd启动错误

验证containerd时,遇到如下错误:

```shell
WARN[0000] image connect using default endpoints: [unix:///var/run/dockershim.sock unix:///run/containerd/containerd.sock unix:///run/crio/crio.sock unix:///var/run/cri-dockerd.sock]. As the default settings are now deprecated, you should set the endpoint instead. 
ERRO[0002] connect endpoint 'unix:///var/run/dockershim.sock', make sure you are running as root and the endpoint has been started: context deadline exceeded 
```

解决办法：

```shell
cat <<EOF> /etc/crictl.yaml 
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

#### kubeadm初始化错误

```shell
 [WARNING Swap]: swap is enabled; production deployments should disable swap unless testing the NodeSwap feature gate of the kubelet
        [WARNING Service-Kubelet]: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
error execution phase preflight: [preflight] Some fatal errors occurred:
        [ERROR FileContent--proc-sys-net-bridge-bridge-nf-call-iptables]: /proc/sys/net/bridge/bridge-nf-call-iptables does not exist
        [ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1
        [ERROR FileExisting-conntrack]: conntrack not found in system path

```

```
modprobe br_netfilter
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 1 > /proc/sys/net/ipv4/ip_forward
```

#### 节点Not ready

cni plugin not initialized

```
container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized
```

安装cni插件

#### 无法访问服务

启动正常，但是无法通过端口访问NodePort，经查，并未在主机上开启监听端口·

#### no route to host

无法访问节点kubelet的状态，显示no route to host

![image-20220914150140877](http://image.huawei.com/tiny-lts/v1/images/2ca2f251daaf081a7bc09f479d461a68_754x242.png@900-0-90-f.png)

此类问题都应该是网络设置机器相关的。

请检查防火墙设置，iptables路由规则设置（Reject）
