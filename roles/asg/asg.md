# roles/asg

app_origin에서 생성한 AMI를 기반으로 Launch Template과 Auto Scaling Group을 구성합니다.
모든 리소스가 완성된 후 마지막에 실행됩니다.

---

## 생성 리소스

- Launch Template (`user03-app-lt`)
- Auto Scaling Group (`user03-app-asg`)
  - Private Subnet 2개 (2a, 2c)
  - min: 1 / desired: 2 / max: 3

---

## 실행

```bash
ansible-playbook site.yml -e "action=main" --tags traffic
ansible-playbook playbooks/pb-asg.yml -e "action=main"
```

---

## 구축 흐름 (main.yml)

Private Subnet / SG 조회
→ app_origin에서 생성한 최신 AMI 조회
→ App TG ARN 조회
→ Launch Template 생성
→ ASG 생성 (App TG 연동)

---

## 주요 설계 포인트

### 최신 AMI 자동 선택

```yaml
target_ami_id: "{{ (found_ami.images | sort(attribute='creation_date') | last).image_id }}"
```

app_origin role에서 타임스탬프 기반으로 여러 AMI가 생성될 수 있습니다.
`creation_date` 기준 정렬 후 마지막 항목을 선택해 항상 최신 AMI를 사용합니다.

### health_check_type: EC2

```yaml
health_check_type: EC2
health_check_period: 300
```

**ELB Health Check 사용 시 문제**
초기 구축 시 `health_check_type: ELB`로 설정했을 때,
App EC2에 웹 서비스가 아직 배포되지 않은 상태에서
ALB Health Check가 실패 → ASG가 EC2를 비정상으로 판단 → 종료 후 재생성을 반복했습니다.

**EC2 Health Check로 변경**
EC2 상태(running)만 확인하므로 웹 서비스가 없어도 정상으로 판단합니다.
CodeDeploy로 웹 서비스가 배포된 이후 ELB Health Check로 전환하는 방식이 안정적입니다.

**개선 가능한 방향**
App Origin EC2에 nginx-proxy를 베이스로 설치해두면
초기 상태에서도 Health Check에 응답할 수 있어 ELB Health Check 유지가 가능합니다.

### ASG 인스턴스 태그 전파

```yaml
tags:
  - key: Name
    value: "{{ prefix }}-asg-instance"
      propagate_at_launch: true
```

ASG가 생성하는 모든 인스턴스에 Name 태그가 자동으로 붙습니다.
태그가 없으면 콘솔에서 인스턴스 식별이 어렵습니다.

---

## 삭제 흐름 (terminate.yml)

ASG 삭제 (wait_timeout: 600) → Launch Template 삭제

ASG 삭제 시 관리 중인 EC2 인스턴스도 함께 종료됩니다.
종료 완료까지 시간이 걸리므로 `wait_for_instances: yes`로 대기합니다.

---

## 변수 (group_vars/main.yml)

| 변수 | 값 | 설명 |
|------|----|------|
| `instance_type` | `t3.medium` | ASG 인스턴스 사양 |
| `key_name` | `user03-key` | SSH 키페어 |