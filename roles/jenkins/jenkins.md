# roles/jenkins

Jenkins EC2를 Private Subnet에 생성하고 Docker Compose로 Jenkins를 실행합니다.

---

## 생성 리소스

- Jenkins EC2 (`user03-jenkins`)
  - Private Subnet (ap-northeast-2a)
  - 고정 IP: `172.43.64.100`
  - 볼륨: 25GB gp3

---

## 실행

```bash
ansible-playbook site.yml -e "action=main" --tags jenkins
ansible-playbook playbooks/pb-jenkins.yml -e "action=main"
```

---

## 주요 설계 포인트

### Private Subnet 배치

```yaml
vpc_subnet_id: "{{ jenkins_subnet.subnets[0].id }}"  # pri-a
```

Jenkins는 외부에서 직접 접근할 필요가 없고,
ALB를 통해서만 접근하는 구조입니다.
보안상 Private Subnet에 배치했습니다.

### 고정 IP 할당

```yaml
private_ip_address: "{{ jenkins_fixed_ip }}"  # 172.43.64.100
```

ALB Target Group에 Jenkins를 등록할 때 IP 기반으로 등록하므로
재생성 시에도 IP가 변경되지 않도록 고정했습니다.

### 중복 생성 방지

```yaml
exact_count: 1
filters:
  "tag:Name": "{{ prefix }}-jenkins"
  instance-state-name:
    - pending
    - running
    - stopping
    - stopped
```

이미 존재하는 Jenkins 인스턴스가 있으면 새로 생성하지 않습니다.

### user_data — Jenkins 자동 설치

```bash
git clone {{ jenkins_git_repo }}
cd jenkins
./docker-install.sh
docker compose up -d
```

EC2 생성 시 GitHub에서 Jenkins 설정을 클론하고
Docker Compose로 Jenkins 컨테이너를 자동 실행합니다.

> 본 프로젝트에서는 강사님이 제공한 Git 리포지토리 기반으로 Jenkins를 구성했습니다.
> (`group_vars/main.yml`의 `jenkins_git_repo` 참고)

### app_origin과의 차이

| 항목 | app_origin | jenkins |
|------|-----------|---------|
| 서브넷 | Public | Private |
| 퍼블릭 IP | 할당 | 미할당 |
| 고정 IP | 없음 | 172.43.64.100 |
| 이후 용도 | AMI 생성 후 종료 가능 | 상시 운영 |
| Jenkins 설치 | 없음 | Docker Compose |

---

## 삭제 흐름 (terminate.yml)

태그 기준으로 Jenkins EC2 조회 → 종료 (wait: yes)

---

## 변수 (group_vars/main.yml)

| 변수 | 값 | 설명 |
|------|----|------|
| `jenkins_fixed_ip` | `172.43.64.100` | Jenkins 고정 IP |
| `jenkins_volume_size` | `25` | 볼륨 크기 (GB) |
| `jenkins_git_repo` | GitHub URL | Jenkins 설정 리포 |