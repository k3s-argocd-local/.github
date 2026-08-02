# k3s + argocd GitOps 무중단 배포

k3s + ArgoCD GitOps 무중단 배포를 구현하고, API 유실 테스트를 통해 검증하는 프로젝트입니다.

## 🔗 관련 레포지토리
- [main-api-server](https://github.com/k3s-argocd-local/main-api-server): 백엔드 코드
- [k8s-manifests](https://github.com/k3s-argocd-local/k8s-manifests): 쿠버네티스 Manifest, K6, k3s + agrocd 세팅 코드

## 🛠️ 기술 스택 (Tech Stack)
### Application & CI/CD
- **Language & Framework:** Java 17 / Spring Boot 3.3.5
- **CI/CD 파이프라인:** GitHub Actions
- **Container Registry:** Docker Hub
- **GitOps Controller:** ArgoCD

### Infrastructure & Test
- **Container Orchestration:** k3s (Lightweight Kubernetes)
- **Cloud Infrastructure:** AWS EC2 (`t3a.medium` 싱글 마스터 노드 구조)
- **성능 및 유실 테스트:** K6

## 🔄 전체 CI/CD 파이프라인 아키텍처
1. **코드 변경 및 Push:** 개발자가 `main-api-server`에 코드를 푸시합니다.
2. **CI 빌드 및 이미지 업로드:** GitHub Actions가 동작하여 컨테이너 이미지를 빌드하고 Docker Hub에 Push합니다.
3. **매니페스트 업데이트:** CI 파이프라인이 `k8s-manifests` 저장소의 이미지 태그를 자동으로 업데이트합니다.
4. **Webhook 트리거:** GitHub의 Webhook이 ArgoCD에 변경 사항을 전달합니다.
5. **GitOps 동기화:** ArgoCD가 변경된 매니페스트를 k3s 클러스터에 동기화하고 새 이미지 배포를 트리거합니다.
6. **안전한 파드 교체:** 인프라 및 애플리케이션 레벨의 최적화 설정을 통해 유실률 0% 상태로 롤링 업데이트를 수행합니다.
<img width="959" height="415" alt="화면 캡처 2026-08-02 181754" src="https://github.com/user-attachments/assets/73ee6887-bcc8-4c3c-a4fc-b14d937a4c6e" />

## 🧪 API 유실 테스트 방법 (Test Methodology)
배포 전환기(Rollout)에서 발생하는 트래픽 누수를 인위적으로 유도하고 아키텍처 설정을 검증하기 위해 다음과 같이 테스트를 설계했습니다.

1. **지연 응답 API 환경 구축**
   - 백엔드 서버에 `Thread.sleep(5000)` 를 통해 인위적 응답 지연 API(`[GET /api/delay]`)를 구현합니다.
2. **4분 30초간 지속적인 요청**
   - 성능 테스트 도구인 **K6**를 활용하여 가상 유저(VUs) 10명이 총 4분 30초 동안 응답 받은 후 0.5초의 대기 시간을 두고 해당 API를 끊임없이 호출하도록 스크립트를 수행합니다.
3. **코드 변경 및 재배포 수행을 통한 전환기 유도**
   - K6 테스트 시작 30초 후 `main-api-server` 소스 코드를 수정 및 Push하여 ArgoCD 배포를 강제로 트리거합니다.
4. **유실률 검증**
   - 구버전 파드가 닫히고 신버전 파드로 교체되는 롤링 업데이트 과정에서 `Connection 끊김`이나 `HTTP 에러`가 발생하는지 실패율(http_req_failed)을 측정합니다.

## 📊 API 유실 테스트 결과 (K6)
* **검증 지표(http_req_failed):** 총 490건의 요청 중 **실패율 0.00% (유실 건수 0건)** 달성
<img width="727" height="570" alt="화면 캡처 2026-08-01 150948" src="https://github.com/user-attachments/assets/a6ad2f56-c5e4-455a-8e97-65a086295a69" />

## 🛠️ 트러블슈팅: 배포 프로세스 중 API 요청 유실 (Troubleshooting)

### 🚨 문제 상황 (Issue)
- 초기 아키텍처 상태에서 ArgoCD 배포(RollingUpdate) 진행 시, 구버전 파드가 닫히고 신버전 파드로 교체되는 전환기에 K6 테스트 상에서 HTTP 요청 실패(유실) 현상이 발생했습니다.

### 🔍 원인 분석
1. **Application 준비 시간 미확보:** 쿠버네티스 파드가 구동된 직후 스프링 부트 컨텍스트 초기화(DB 커넥션 풀 확보 등)가 완료되기 전에 트래픽이 유입되어 오류 응답 발생
2. **처리 중인 요청의 강제 종료:** 구버전 파드가 종료 신호(`SIGTERM`)를 받는 순간 톰캣 프로세스가 즉시 종료 과정을 밟으면서, 라우팅 격리 직전에 들어온 마지막 요청 및 기존 스레드에서 처리 중이던 HTTP 연결이 강제로 끊어짐

### 💡 문제 해결 과정 (Resolution)

#### [Application 레벨 처리]
- **Graceful Shutdown 활성화 (`server.shutdown=graceful`)**
  - `SIGTERM` 신호를 수신하더라도 진행 중인 톰캣 스레드의 HTTP 요청 처리를 강제 단절하지 않고, 안전하게 응답을 완료한 후 프로세스가 종료되도록 보장했습니다.
- **종료 유예 시간 최적화 (`spring.lifecycle.timeout-per-shutdown-phase=30s`)**
  - 인프라 격리 이후 남은 잔여 요청들을 처리할 수 있도록 최대 30초의 프로세스 종료 유예 타이밍을 설정했습니다.

#### [Infrastructure 레벨 처리]
- **Readiness Probe 적용 (`[GET /api/health]`)**
  - 스프링 부트 애플리케이션 초기화가 완전히 끝나고 실제 요청 처리가 가능한 상태(`health: 200`)를 검증한 후 라우팅(Endpoints)에 투입하도록 제어하여 배포 초기 유실을 차단했습니다.
- **PreStop Lifecycle Hook 도입 (`sleep 10`)**
  - 파드가 종료 절차에 들어가면 인프라 라우팅에서 먼저 격리될 수 있도록 10초간 배포 대기 시간을 부여하여, 전환기에 흘러 들어오는 잔여 요청의 유실을 방지했습니다.


## ⚠️ 아키텍처적 한계 및 대안 (Limitations & Alternatives)

### 1. 인프라 구조의 한계
- **싱글 마스터 노드 구성:** 비용 절감을 위해 다중 마스터가 아닌 `t3a.medium` 단일 인스턴스 구조를 채택했습니다. 
- **리스크:** 단일 마스터 노드가 다운될 경우 클러스터 전체의 제어 기능이 일시적으로 마비될 수 있는 구조적 한계가 존재합니다.

### 2. 복구 대안 및 해결 방안
- **선언적 매니페스트 기반 신속 복구:** 모든 인프라 상태가 [k8s-manifests](https://github.com/k3s-argocd-local/k8s-manifests) 저장소에 선언적으로 관리되고 있습니다.
- **장점:** 마스터 노드에 치명적인 장애가 발생하더라도, 새 EC2 인스턴스를 생성하고 해당 깃 레포지토리에 다시 연결하여 이전과 동일한 상태로 복구 할 수 있습니다.
