# Cluster Bootstrap Runbook

kubeadm 기반 3노드(master 1 + worker 2) 클러스터를 처음부터 구축하는 절차.
버전 고정: **Kubernetes v1.36.4** / containerd 1.7.29

> **표기**: 실제 입력한 명령이 diary/INC에 남아 있는 것은 그대로 적었고, 명령 원문이 남지 않아 표준 절차로 채운 것은 `[재구성]`으로 표시했다. `[재구성]` 항목은 다음 재구축 때 실제로 실행해 보고 확정할 것.

---

## 0. 사전 준비 (전 노드 공통)

### 0-1. VM 클론 후 정리 `[재구성]`

Full Clone 직후 hostname, machine-id, SSH host key가 템플릿과 같다 (Day 2). 노드마다 다시 만든다.

```bash
sudo hostnamectl set-hostname k8s-master        # worker는 k8s-worker1, k8s-worker2
sudo rm -f /etc/machine-id && sudo systemd-machine-id-setup
sudo rm -f /etc/ssh/ssh_host_* && sudo ssh-keygen -A && sudo systemctl restart sshd
```

### 0-2. 네트워크 `[재구성]`

어댑터 매핑(MAC 주소 대조로 확인): `enp0s3`=NAT, `enp0s8`=Host-only, `enp0s9`=Internal.

| 노드 | Host-only (enp0s8) | Internal (enp0s9) |
|---|---|---|
| master | 172.25.250.10/24 | 10.10.0.10/24 |
| worker1 | 172.25.250.11/24 | 10.10.0.11/24 |
| worker2 | 172.25.250.12/24 | 10.10.0.12/24 |

```bash
# master 예시 (worker는 IP 끝자리만 .11 / .12)
sudo nmcli con mod enp0s8 ipv4.method manual ipv4.addresses 172.25.250.10/24 ipv4.never-default yes
sudo nmcli con mod enp0s9 ipv4.method manual ipv4.addresses 10.10.0.10/24 ipv4.never-default yes

# NAT는 DHCP. 클론에 남은 고정 IP(172.17.0.200)를 반드시 제거 (Day 4 장애)
sudo nmcli con mod enp0s3 ipv4.method auto ipv4.addresses ""
sudo nmcli connection reload
sudo nmcli con up enp0s3; sudo nmcli con up enp0s8; sudo nmcli con up enp0s9
```

**검증:**
```bash
ip -4 a show enp0s8 ; ip -4 a show enp0s9
ip route                                  # default 경로가 enp0s3 하나뿐인지
cat /etc/NetworkManager/system-connections/enp0s3.nmconnection   # address1= 줄이 없는지 (nmcli clear가 파일에 반영 안 되는 경우가 있었음)
ping -c 3 8.8.8.8
```

### 0-3. swap off, 커널 모듈, sysctl `[재구성]`

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

**검증 (재부팅 후에도 동일해야 함):**
```bash
free -h ; swapon --show                    # Swap 0B, 출력 없음
lsmod | grep -e overlay -e br_netfilter
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward   # 전부 1
```

### 0-4. containerd 1.7.29 (Day 4)

> Docker 저장소 추가는 **3대 모두에** 해야 한다. 한 대에만 하고 넘어가 설치가 실패한 적이 있다.

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y containerd.io-1.7.29                                  # [재구성] 버전 지정 형태
containerd config default | sudo tee /etc/containerd/config.toml          # [재구성]
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml   # [재구성]
sudo systemctl enable --now containerd
```

**검증:**
```bash
containerd --version                                  # v1.7.29
grep -B2 -A2 "SystemdCgroup" /etc/containerd/config.toml   # SystemdCgroup = true
systemctl status containerd --no-pager                # active (running)
```

---

## 1. kubeadm / kubelet / kubectl 설치 (전 노드 공통)

> 마스터, 워커1, 워커2 **전부 동일하게** 실행. 버전이 하나라도 다르면 이후 단계에서 스큐 문제 발생.

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

**검증:**
```bash
kubeadm version
kubelet --version
kubectl version --client
```
→ 3대 전부 같은 마이너/패치 버전인지 반드시 확인.

**알려진 정상 동작 (설치 직후):**
`systemctl status kubelet` 결과가 `activating (auto-restart)`로 반복 재시작하는 게 정상.
`journalctl -u kubelet`에 `config.yaml: no such file or directory` 에러가 찍힘 — `kubeadm init`/`join`을 실행해야 해소됨(§2에서 이어짐).

---

## 2. kubeadm init (마스터) (Day 6)

### 2-1. /etc/hosts (3대) `[재구성]`

일지에는 `10.10.0.10 k8s-master`만 적혀 있고 워커 줄은 같은 규칙으로 채움.

```bash
cat <<EOF | sudo tee -a /etc/hosts
10.10.0.10 k8s-master
10.10.0.11 k8s-worker1
10.10.0.12 k8s-worker2
EOF
```

### 2-2. firewalld: Internal 대역에서 오는 트래픽만 허용

`--add-port`로 전체 오픈하면 Host-only(관리망)에서도 API 서버에 닿아 망 분리가 무의미해지므로, 출발지를 `10.10.0.0/24`로 제한한 rich rule을 쓴다. 규칙 형태는 INC-002의 8472/udp 규칙에서 가져옴 `[재구성]`.

```bash
# master: 6443(API 서버), 2379-2380(etcd), 10250(kubelet), 10259(scheduler), 10257(controller-manager)
for port in 6443 2379-2380 10250 10259 10257; do
  sudo firewall-cmd --permanent --zone=public --add-rich-rule="rule family=\"ipv4\" source address=\"10.10.0.0/24\" port port=\"$port\" protocol=\"tcp\" accept"
done
sudo firewall-cmd --reload

# worker1, worker2: kubelet API 하나만
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="10.10.0.0/24" port port="10250" protocol="tcp" accept'
sudo firewall-cmd --reload
```

**검증:** `sudo firewall-cmd --list-rich-rules` (3대 모두)

> ⚠ 이 목록에는 **Flannel VXLAN(UDP 8472)이 빠져 있다.** 놓치면 INC-002가 재발한다 → §3-3에서 보강.

### 2-3. init

```bash
sudo kubeadm init --apiserver-advertise-address=10.10.0.10 --pod-network-cidr=10.244.0.0/16
```

- `--apiserver-advertise-address=10.10.0.10`: 컨트롤플레인 통신을 Internal망에 둔다 (decisions/0001).
- `--pod-network-cidr=10.244.0.0/16`: Flannel 기본값. Host-only, Internal 대역과 겹치지 않음을 확인.

### 2-4. kubectl 연결 `[재구성]`

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 2-5. baseline 확인 (CNI 설치 전)

```bash
kubectl get nodes                  # k8s-master NotReady  ← 정상 (CNI 없음)
kubectl describe node k8s-master   # Ready 조건에 NetworkPluginNotReady (cni plugin not initialized)
```

`container runtime is down` / `PLEG is not healthy` / `NetworkPluginNotReady`는 CNI 부재 하나의 원인에서 나온 연쇄 증상이다. `systemctl status containerd`로 런타임 자체는 정상임을 확인한다.

---

## 3. CNI 설치: Flannel (Day 7, 8, 10)

### 3-1. 설치 (마스터)

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml   # [재구성] URL
```

- Pod CIDR이 Flannel 기본값(`10.244.0.0/16`)과 같아서 매니페스트를 고칠 필요 없다.
- `latest` URL은 [재구성]이다. 당시 설치 버전과 URL이 일지에 남지 않았으므로, 다음에는 **버전을 고정해서 설치하고 버전을 기록**할 것.
- DaemonSet 이름은 `kube-flannel-ds`, 네임스페이스는 `kube-flannel`.

### 3-2. 언더레이 인터페이스 고정 (Day 8, decisions/0003)

`--iface`를 안 주면 Flannel이 기본 경로를 가진 NAT(enp0s3)를 노드 주소로 잡는다 (Day 2의 `never-default` 때문).

```bash
kubectl patch daemonset kube-flannel-ds -n kube-flannel --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--iface=enp0s9"}]'
```

> `kubectl edit`은 YAML 들여쓰기 오류로 실패한 적이 있어 `patch`를 쓴다.

**검증:**
```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.flannel\.alpha\.coreos\.com/public-ip}{"\n"}{end}'
# k8s-master 10.10.0.10 / k8s-worker1 10.10.0.11 / k8s-worker2 10.10.0.12  (172.17.x면 잘못된 것)
```

### 3-3. 방화벽 보강 (Day 10, INC-002) — 3대 모두

```bash
# VXLAN 포트 허용 (양방향 통신이므로 모든 노드)
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="10.10.0.0/24" port port="8472" protocol="udp" accept'

# CNI 인터페이스를 trusted zone으로 (zone 미지정이면 public의 제한 정책이 적용되어 FORWARD가 막힘)
sudo firewall-cmd --zone=trusted --change-interface=cni0 --permanent
sudo firewall-cmd --zone=trusted --change-interface=flannel.1 --permanent
sudo firewall-cmd --reload
```

**검증:**
```bash
sudo firewall-cmd --get-active-zones          # trusted: cni0 flannel.1
sudo firewall-cmd --permanent --zone=trusted --list-interfaces   # runtime과 일치, 재부팅 후에도 유지
ping -c 3 <다른 노드의 Pod IP> ; curl -m 5 <다른 노드의 Pod IP>:80
```

> 다음 재구축 때는 3-2와 3-3을 **설치 전/직후에 한꺼번에 반영**하는 순서를 시험해 볼 것 (미검증).

---

## 4. kubeadm join (워커) (Day 7)

join 토큰은 기본 24시간 후 만료된다. 만료됐거나 임박했으면 재발급한다.

```bash
# 마스터에서
kubeadm token list                              # TTL 확인
kubeadm token create --print-join-command
# 출력: kubeadm join 10.10.0.10:6443 --token <REDACTED> --discovery-token-ca-cert-hash sha256:<REDACTED>

# worker1, worker2에서 위 출력을 그대로 sudo로 실행
sudo kubeadm join 10.10.0.10:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

> 실제 토큰과 해시는 공개 레포에 커밋하지 않는다. 필요할 때 위 명령으로 재발급한다.

**검증 (Phase 2 baseline):**
```bash
kubectl get nodes                      # 3대 모두 Ready (k8s-master, k8s-worker1, k8s-worker2)
kubectl get pods -n kube-flannel -o wide   # 3개 모두 1/1 Running, IP는 10.10.0.10 / .11 / .12
kubectl get pods -A -o wide
```

---

## 알려진 함정 (실제로 겪은 것)

| 함정 | 증상 | 막는 법 | 기록 |
|---|---|---|---|
| 클론에 NAT 고정 IP가 남음 | 3대가 같은 IP로 충돌, 워커 인터넷 불통 | §0-2에서 NAT를 DHCP로 | Day 4 |
| 저장소 추가를 한 대에만 함 | 나머지 노드 설치 실패 | §0-4를 3대 모두 | Day 4 |
| Flannel이 NAT망을 노드 주소로 잡음 | FDB dst가 172.17.x | §3-2 `--iface=enp0s9` | Day 8, decisions/0003 |
| VXLAN 포트와 CNI zone 누락 | 노드 간 Pod 통신 두절, NodePort 무응답 | §3-3 | Day 10, INC-002 |
| join 토큰 만료 | join 실패 | `token create --print-join-command` | Day 7 |
