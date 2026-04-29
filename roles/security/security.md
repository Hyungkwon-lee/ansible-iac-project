# roles/security

AWS Security Group을 구성합니다.
EC2 인스턴스 생성 전에 실행되어야 합니다.

---

## 생성 리소스

| SG 이름 | 허용 포트 | 대상 | 용도 |
|---------|----------|------|------|
| `user03-ssh-sg` | 22 | 0.0.0.0/0 | SSH 접속 (개발 환경) |
| `user03-web-sg` | 80, 443 | 0.0.0.0/0 | 외부 사용자 HTTP/HTTPS 접근 |
| `user03-ssm-sg` | 443 | VPC CIDR 내부 | SSM Endpoint 통신용 |

---

## 실행

```bash
ansible-playbook site.yml -e "action=main" --tags sg
ansible-playbook playbooks/pb-security.yml -e "action=main"
```

---

## 주요 설계 포인트

### SSM SG — VPC 내부만 허용

```yaml
cidr_ip: "{{ vpc_cidr }}"  # 172.43.0.0/16
```

SSM Session Manager를 통해 EC2에 접속하는 구조에서
SSM Endpoint는 VPC 내부 트래픽만 허용하면 됩니다.
외부에서 직접 접근할 필요가 없어 VPC CIDR로 제한했습니다.

### SSH SG — 운영 환경 개선 포인트

```yaml
cidr_ip: 0.0.0.0/0  # 실제 운영 시 특정 IP로 제한 권장
```

현재는 학습 환경이라 전체 허용으로 설정했습니다.
실제 운영 환경에서는 관리자 IP로 제한하거나 SSH 자체를 제거하고
SSM Session Manager만 사용하는 방식이 보안상 권장됩니다.

---

## 삭제 흐름 (terminate.yml)

VPC ID 조회 → ssm-sg 삭제 → web-sg 삭제 → ssh-sg 삭제

SG 삭제 시 참조 관계가 있으면 삭제가 실패할 수 있습니다.
EC2 인스턴스가 먼저 삭제된 이후에 실행해야 합니다.

---

## 변수 (group_vars/main.yml)

| 변수 | 설명 |
|------|------|
| `prefix` | 리소스 이름 prefix (`user03`) |
| `vpc_cidr` | SSM SG 인바운드 허용 대역 (`172.43.0.0/16`) |