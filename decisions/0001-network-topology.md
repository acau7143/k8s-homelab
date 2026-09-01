# 0001: Host-only + Internal Network 이중 구성

## Context
- kubeadm으로 멀티노드 K8s 클러스터를 구축하려면 마스터/워커 노드 간 통신망과 관리자가 각 노드에 SSH로 접속할 관리망이 필요함
- VirtualBox는 NAT, Bridged, Host-only, Internal Network 등 여러 어댑터 타입을 지원하며, 노드 생성 전에 조합을 결정해야 함
- 실무 AWS VPC에서는 public subnet(관리/외부 접근)과 private subnet(내부 통신 전용)을 분리하는 게 표준 설계이며, 이 개념을 로컬 환경에서 재현하고자 함
- **참고**: 패키지 설치(containerd, kubeadm 등)를 위한 인터넷 접속은 별도 NAT 어댑터로 항상 유지하며, 이 문서는 관리망/클러스터망 분리에 대한 결정만 다룸 (NAT는 설계 논의 대상이 아닌 필수 인프라)

## Decision
각 노드(마스터 1, 워커 2)에 네트워크 어댑터를 아래와 같이 구성한다.
- **Host-only Adapter**: 호스트 PC → 각 노드 SSH 접속용 관리망
- **Internal Network**: 노드 간 kubeadm/kubelet/CNI 통신 전용, 호스트 OS와 격리

## Why
- Host-only는 호스트-게스트 간에만 연결되고 외부와 격리되어 "관리 목적 전용 통로" 역할 → VPC public subnet의 배스천 호스트 SSH 접근과 유사
- Internal Network는 VirtualBox 내부에서만 존재하는 완전 격리망 → VPC private subnet(내부 통신만 허용) 개념과 유사
- 관리 트래픽과 클러스터 트래픽을 어댑터 단위로 분리해두면, 나중에 kube-proxy/CNI가 실제로 어느 인터페이스를 쓰는지 구분해서 관찰하기 쉬움

## 대안과의 비교
| 대안 | 장점 | 단점 | 채택 여부 |
|---|---|---|---|
| NAT 단일 네트워크 | 설정 간단 | 관리망/클러스터망 구분 안 됨, subnet 분리 개념 실습 불가 | 미채택 |
| Bridged 단일 네트워크 | 물리망과 동일 대역, 설정 쉬움 | 집 공유기 대역 그대로 사용 → 다른 기기와 충돌 위험 | 미채택 |
| Host-only + Internal 이중 구성 | 관리망/클러스터망 분리, VPC 개념 재현 | 어댑터 2개 관리 필요 | **채택** |

## Result
(Day 2 VM 생성 및 네트워크 설정 후 실제 검증 결과로 채움 — 예: `ip a`로 노드별 인터페이스 확인, Host-only 대역 SSH 성공 여부, Internal 대역에서만 노드 간 ping 성공 여부)