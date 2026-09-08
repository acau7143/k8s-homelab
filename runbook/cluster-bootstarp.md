# Cluster Bootstrap Runbook

kubeadm 기반 3노드(master 1 + worker 2) 클러스터를 처음부터 구축하는 절차.
버전 고정: **Kubernetes v1.36.4** / containerd 1.7.29

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

## 2. kubeadm init (마스터) — *다음 세션에서 작성 예정*

## 3. CNI 설치 — *Phase 3에서 이어서 작성*

## 4. kubeadm join (워커) — *다음 세션에서 작성 예정*