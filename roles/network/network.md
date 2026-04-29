# roles/network

AWS 네트워크 기반 인프라를 구성합니다.
모든 리소스의 기반이 되므로 가장 먼저 실행되어야 합니다.

---

## 생성 리소스

- VPC (`172.43.0.0/16`)
- Public Subnet × 2 (ap-northeast-2a, 2c)
- Private Subnet × 2 (ap-northeast-2a, 2c)
- Internet Gateway
- NAT Gateway + EIP
- Public Route Table (IGW 연결)
- Private Route Table (NAT GW 연결)

---

## 실행

```bash
# 전체 실행 시
ansible-playbook site.yml -e "action=main" --tags network

# 개별 실행 시
ansible-playbook playbooks/pb-network.yml -e "action=main"
```

---

## 구축 흐름 (main.yml)

VPC 생성
→ Public/Private Subnet × 4 생성 (loop)
→ IGW 생성
→ EIP 조회/생성 (중복 방지)
→ NAT Gateway 조회/생성 (중복 방지)
→ Public RT 생성 → IGW 기본 경로 추가 → Public Subnet 연결
→ Private RT 생성 → NAT GW 기본 경로 추가 → Private Subnet 연결

---

## 주요 설계 포인트

### 멱등성 처리 — EIP / NAT Gateway 중복 생성 방지

```yaml
# 기존 EIP 조회 후 없을 때만 생성
- name: Allocate EIP for NAT Gateway
  when: nat_eip_info.addresses | length == 0

# 기존 NAT GW 조회 후 없을 때만 생성
- name: Create NAT Gateway
  when: existing_nat_gw_id == ''
```

Playbook을 여러 번 실행해도 리소스가 중복 생성되지 않도록 처리했습니다.
NAT Gateway는 생성 비용이 발생하므로 특히 중요합니다.

### purge_routes: false

```yaml
- name: Associate Public Subnets with Public Route Table
  purge_routes: false
```

서브넷 연결 시 기존 라우트가 삭제되지 않도록 설정했습니다.
기본값(`true`)으로 두면 서브넷 연결 작업 시 이미 추가한 IGW/NAT 경로가 삭제됩니다.

### 서브넷 리스트 loop 처리

```yaml
loop: "{{ pub_subnets + pri_subnets }}"
```

`group_vars/main.yml`의 리스트를 그대로 loop에 활용해
서브넷 추가/변경 시 코드 수정 없이 변수만 수정하면 됩니다.

---

## 삭제 흐름 (terminate.yml)

의존관계 역순으로 삭제합니다.

Route Table 삭제
→ NAT Gateway 삭제 (wait: yes)
→ EIP 릴리스
→ Subnet 삭제
→ IGW 삭제
→ VPC 삭제

> NAT Gateway 삭제는 시간이 걸리므로 `wait: yes`로 완전히 삭제될 때까지 대기합니다.
> 삭제 실패 시 다음 단계로 넘어가도록 `ignore_errors: yes`를 적용했습니다.

---

## 변수 (group_vars/main.yml)

| 변수 | 값 | 설명 |
|------|----|------|
| `vpc_cidr` | `172.43.0.0/16` | VPC CIDR |
| `pub_subnets` | 2개 리스트 | Public 서브넷 (2a, 2c) |
| `pri_subnets` | 2개 리스트 | Private 서브넷 (2a, 2c) |
| `prefix` | `user03` | 모든 리소스 이름 prefix |
| `region` | `ap-northeast-2` | AWS 서울 리전 |