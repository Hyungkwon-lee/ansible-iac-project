# roles/loadbalancer

Application Load Balancer와 Target Group을 구성합니다.
ASG 생성 전에 실행되어야 합니다.

---

## 생성 리소스

- ALB (`user03-alb`) — Public Subnet 2개에 배치
- App Target Group (`user03-app-tg`) — instance 타입, ASG 연동용
- Jenkins Target Group (`user03-jenkins-tg`) — IP 타입, 고정 IP 직접 등록

---

## 실행

```bash
ansible-playbook site.yml -e "action=main" --tags traffic
ansible-playbook playbooks/pb-loadbalancer.yml -e "action=main"
```

---

## 구축 흐름 (main.yml)

Public Subnet / Web SG 조회
→ App TG 생성 (instance 타입)
→ Jenkins TG 생성 (ip 타입, 고정 IP 등록)
→ ALB 생성 (기본 404 응답 Listener만)
→ 20초 안정화 대기
→ ALB Listener에 Host 기반 Rule 추가

---

## 주요 설계 포인트

### ALB 2단계 생성

```yaml
# 1단계: ALB 먼저 생성 (기본 Listener만)
DefaultActions:
  - Type: fixed-response
    StatusCode: "404"

# 2단계: 안정화 후 Rule 추가
Rules:
  - Priority: 10
    Conditions:
      - Field: host-header
        Values: ["{{ jenkins_domain }}"]
    Actions:
      - Type: forward → jenkins-tg

  - Priority: 20
    Conditions:
      - Field: host-header
        Values: ["{{ app_domain }}"]
    Actions:
      - Type: forward → app-tg
```

ALB가 완전히 생성되기 전에 Rule을 추가하면 실패하는 경우가 있어
ALB 생성 → 안정화 대기 → Rule 추가 순서로 2단계로 나눴습니다.

### App TG vs Jenkins TG 타입 차이

| 항목 | App TG | Jenkins TG |
|------|--------|-----------|
| target_type | instance | ip |
| 등록 방식 | ASG가 자동 등록 | 고정 IP 직접 등록 |
| health_check_path | `/` | `/login` |

App TG는 ASG가 인스턴스를 자동으로 등록/해제합니다.
Jenkins TG는 고정 IP(`172.43.64.100`)를 직접 등록하는 방식입니다.

Jenkins의 health check path를 `/login`으로 설정한 이유는
Jenkins 루트(`/`)가 리다이렉트 응답을 반환해 health check가 실패할 수 있기 때문입니다.

### 기본 응답 404

매핑된 도메인 외의 요청은 404로 응답합니다.
의도하지 않은 접근을 차단하는 효과가 있습니다.

---

## 삭제 흐름 (terminate.yml)

ALB 삭제 (wait: yes) → App TG 삭제 → Jenkins TG 삭제

ALB가 삭제되기 전에 TG를 삭제하면 실패할 수 있으므로
ALB 먼저 삭제 후 TG를 삭제합니다.

---

## 변수 (group_vars/main.yml)

| 변수 | 값 | 설명 |
|------|----|------|
| `jenkins_domain` | `user03-jenkins.busanit.com` | Jenkins ALB 라우팅 도메인 |
| `app_domain` | `user03-app.busanit.com` | App ALB 라우팅 도메인 |
| `jenkins_fixed_ip` | `172.43.64.100` | Jenkins TG 등록 IP |