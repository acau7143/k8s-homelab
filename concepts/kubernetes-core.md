# Kubernetes 핵심 개념

> 날짜와 무관한 참고 문서. 배우면서 계속 보강한다.
>
> **표기**
> - 🔬 **프로젝트 확인**: 이 홈랩에서 직접 보거나 파일로 남아 있는 것
> - 📖 **일반 지식**: 문서/학습으로 아는 내용이고, 이 프로젝트에서 직접 확인하지는 않음
>
> 면접에서 "직접 해봤나요?"에 답할 때 이 경계를 기준으로 말한다.

---

## 큰 그림

비유: 식당 체인.

```
클러스터 = 식당 체인 전체
├─ Control Plane (마스터) = 본사 (주문 접수, 지시, 기록 보관)
└─ Node (워커)            = 매장 (실제로 음식을 만드는 곳)
     └─ Pod               = 주방 한 칸 (컨테이너가 들어 있는 실행 단위)
```

우리 클러스터: master 1대 + worker 2대 (`k8s-master`, `k8s-worker1`, `k8s-worker2`), v1.36.4.

---

## Node

컨테이너가 실제로 돌아가는 서버. VM이든 물리 서버든 클러스터에 join된 머신 하나가 Node.

| 노드 구성요소 | 역할 |
|---|---|
| kubelet | 노드마다 상주하는 관리인. API 서버에서 맡을 Pod를 받아 containerd로 띄우고 상태를 보고 |
| containerd | 컨테이너 실행기 |
| kube-proxy | Service 규칙(iptables)을 노드에 만들어 둠 |

- `kubectl get nodes`의 **Ready** = kubelet이 정상이고 CNI가 초기화됐다는 뜻. **노드 간 통신이 된다는 보증은 아님.**
- 🔬 Day 7: 3노드 모두 Ready. 그런데 UDP 8472가 막혀 있었음 (Day 10, INC-002). Ready와 통신 가능은 별개.

---

## Pod

쿠버네티스가 다루는 **가장 작은 배포 단위**. 컨테이너 1개 이상을 묶은 것.

- 쿠버네티스는 컨테이너가 아니라 Pod 단위로 배치한다.
- 한 Pod 안의 컨테이너는 IP 하나를 공유하고 localhost로 통신.
- **일회용**: 죽으면 같은 Pod가 부활하지 않고 **새 Pod가 새 IP로** 생김. (📖)
- 그래서 Pod IP에 직접 의존하지 않고 Service를 앞에 둔다.
- 🔬 Pod IP 대역: `10.244.0.0/16`을 노드마다 `/24`로 분할 (Day 8: `10.244.0.0/24`, `10.244.1.0/24` 확인).

### Static Pod

kubelet이 API 서버를 거치지 않고 노드의 매니페스트 파일에서 직접 띄우는 Pod.

- 🔬 마스터의 apiserver, etcd, scheduler, controller-manager가 이 방식 (Day 6 containerd 로그에서 static pod 컨테이너 기동 확인).
- 그래서 kubelet이 모든 노드의 핵심.

---

## Deployment

**"이 Pod를 N개 유지해라"** 라고 원하는 상태를 선언하는 오브젝트.

```
Deployment  "nginx 2개 유지"
   └─ ReplicaSet  (개수를 실제로 맞추는 중간 관리자)
        ├─ Pod 1
        └─ Pod 2
```

| 기능 | 설명 |
|---|---|
| 자가 치유 | Pod가 죽으면 ReplicaSet이 새로 띄움 |
| 스케일 | replicas 숫자만 바꾸면 늘리고 줄임 |
| 롤링 업데이트 | 이미지 버전을 바꾸면 Pod를 하나씩 교체 |
| 롤백 | 문제가 생기면 이전 버전으로 되돌림 |

핵심은 **선언형(declarative)**: "어떻게 하라"가 아니라 "이런 상태여야 한다"를 적으면 컨트롤러가 현재 상태를 거기에 맞춘다.

Pod를 직접 만들면 죽어도 아무도 다시 만들어 주지 않는다. 그래서 보통 Deployment로 만든다.

🔬 `phase3-network/nginx-clusterip.yaml`: `nginx-test` Deployment, replicas 2.

---

## DaemonSet

**"모든 노드에 Pod를 하나씩"**.

| | Deployment | DaemonSet |
|---|---|---|
| 개수 기준 | 내가 정한 N개 | 노드 수만큼 |
| 배치 | 스케줄러가 판단 (한 노드에 몰릴 수 있음) | 노드마다 정확히 1개 |
| 용도 | 일반 애플리케이션 | 노드마다 필요한 인프라 (네트워크, 로그 수집, 모니터링) |
| 이 프로젝트 | nginx-test | **Flannel** |

- Flannel을 DaemonSet으로 띄우는 이유: 노드마다 자기 Pod 대역(`/24`) 할당, `flannel.1` 인터페이스, 경로 설정이 따로 필요하기 때문. 한 노드에만 있으면 그 노드만 네트워크가 된다. 노드가 새로 join하면 Pod도 자동으로 하나 생긴다.
- 🔬 Day 7: `kube-flannel` Pod 3개 모두 `1/1 Running`, IP는 각 노드 Internal IP (`10.10.0.10/.11/.12`).

---

## Service

Pod가 죽고 새로 생길 때마다 IP가 바뀌므로, **바뀌지 않는 고정 주소**를 Pod 앞에 두는 오브젝트.

```
클라이언트 → Service (고정 가상 IP)
                ├─→ Pod 1
                └─→ Pod 2   (살아 있는 Pod들로 분산)
```

- Service는 **label selector**로 대상 Pod를 찾는다. Pod가 교체돼도 라벨이 같으면 자동으로 따라간다.
- 실제 분산은 각 노드의 **kube-proxy가 만든 iptables 규칙**이 한다. (🔬 Day 11~12: `KUBE-SERVICES → KUBE-SVC → KUBE-SEP` 체인, probability 기반 분산)
- kube-proxy는 **DNAT**(주소 변환)을 할 뿐 **DNS가 아니다**. 이름 → IP 변환은 CoreDNS의 일.

| 종류 | 접근 범위 |
|---|---|
| ClusterIP | 클러스터 안에서만 |
| NodePort | 모든 노드의 특정 포트(30000번대)로 외부 접근 |
| LoadBalancer | 클라우드 로드밸런서 연동. 홈랩에는 기본 제공 없음 |

🔬 `nginx-test-svc`: 파일명은 `nginx-clusterip.yaml`이지만 실제 타입은 **NodePort**. (파일명이 실제와 다르므로 헷갈리지 말 것)

---

## 기타 기본 개념

| 개념 | 설명 |
|---|---|
| Namespace | 클러스터 안의 논리적 구획 (`default`, `kube-system`, `kube-flannel` 등). 리눅스 namespace(커널 격리 기능)와는 별개 개념 → [docker-and-containers.md](docker-and-containers.md#namespace-두-가지-이름만-같은-별개-개념) |
| Label / Selector | 오브젝트에 붙이는 이름표와 그걸로 대상을 고르는 방법. Service가 Pod를 찾는 방식 |
| etcd | 클러스터의 모든 상태를 저장하는 DB |
| Scheduler | 새 Pod를 어느 Node에 놓을지 결정 |
| Controller Manager | "원하는 상태"와 "현재 상태"를 비교해 맞추는 컨트롤러 묶음 (ReplicaSet 컨트롤러 등) |
| CNI | Pod 네트워크를 구현하는 플러그인 규격. 이 프로젝트는 Flannel |

---

## `kubectl apply`부터 컨테이너 실행까지

```
1. kubectl → API 서버(6443)에 "nginx 2개 원함" 전달
2. API 서버가 etcd에 저장
3. Deployment 컨트롤러가 ReplicaSet 생성 → Pod 생성 요청
4. Scheduler가 각 Pod를 어느 Node에 놓을지 결정
5. 해당 Node의 kubelet이 API 서버에서 "내 Pod" 목록을 받아옴
6. kubelet → containerd에게 컨테이너 실행 지시
7. CNI(Flannel)가 Pod에 IP 부여
8. Service가 있으면 kube-proxy가 iptables 규칙 생성
```

- 5~6번: 🔬 Day 5에서 정리한 kubelet의 역할.
- 전체 흐름 자체는 📖 일반 지식. 이 프로젝트에서는 단계별로 하나씩 추적해 보지 않았다.

관련 문서: 컨테이너, 이미지, Docker와 Kubernetes의 관계는 [docker-and-containers.md](docker-and-containers.md) 참고.

---
