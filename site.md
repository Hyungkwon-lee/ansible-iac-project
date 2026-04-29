# site.yml — 전체 실행 흐름

Ansible Playbook으로 AWS 인프라를 순차적으로 구축합니다.
모든 리소스는 `localhost`에서 AWS API를 호출하는 방식으로 생성됩니다.

---

## 실행 환경

| 항목 | 버전 |
|------|------|
| Ansible | 2.18.15 |
| Python | 3.11.14 |
| OS | Amazon Linux 2023 (EC2) |
| amazon.aws | 10.1.2 |
| community.aws | 10.1.0 |

컬렉션 설치:
```bash
ansible-galaxy collection install -r requirements.yml
```

---

## ansible.cfg 주요 설정

```ini
inventory = ./inventory.yml
private_key_file = ~/.ssh/user03-key.pem
remote_user = ec2-user
host_key_checking = False
interpreter_python = auto_silent
```

- `remote_user = ec2-user` — Ansible Control Node(Amazon Linux 2023) 기본 사용자
- `host_key_checking = False` — 새 EC2 접속 시 known_hosts 확인 생략
- `interpreter_python = auto_silent` — Python 경로 자동 감지, 경고 메시지 억제

---

## 실행 전 준비 — MFA 인증

AWS IAM User에 MFA가 적용되어 있어 일반 Access Key로는 API 호출이 불가합니다.
`token.sh`로 STS 임시 Credential을 발급받아 환경변수로 등록한 후 실행합니다.

```bash
source scripts/token.sh {MFA_CODE}
```

```bash
# token.sh 동작 방식
aws sts get-session-token \
  --serial-number arn:aws:iam::...:mfa/user03-MFA \
  --token-code {MFA_CODE} \
  --duration-seconds 43200   # 12시간 유효
```

발급된 `AccessKeyId`, `SecretAccessKey`, `SessionToken`을 환경변수로 export해
이후 Playbook 실행 시 자동으로 사용됩니다.

---

## 실행 명령어

```bash
# 전체 인프라 구축
ansible-playbook site.yml -e "action=deploy"

# 특정 role만 실행 (태그 사용)
ansible-playbook site.yml -e "action=deploy" --tags network
ansible-playbook site.yml -e "action=deploy" --tags sg
ansible-playbook site.yml -e "action=deploy" --tags iam
ansible-playbook site.yml -e "action=deploy" --tags origin
ansible-playbook site.yml -e "action=deploy" --tags jenkins
ansible-playbook site.yml -e "action=deploy" --tags loadbalancer
ansible-playbook site.yml -e "action=deploy" --tags asg
```

---

## 실행 순서

```
network → iam → security → app_origin → jenkins → loadbalancer → asg
```

의존관계에 따라 순서가 고정됩니다.

| 순서 | Role | 생성 리소스 | 이유 |
|------|------|------------|------|
| 1 | network | VPC, Subnet, IGW, NAT, Route Table | 모든 리소스의 기반 |
| 2 | iam | Security Group | EC2 생성 전 SG 필요 |
| 3 | security | IAM Role, Instance Profile | EC2 생성 전 Role 필요 |
| 4 | app_origin | WAS EC2, AMI | ASG의 Launch Template 기반 이미지 |
| 5 | jenkins | Jenkins EC2 | 독립적이나 네트워크/SG 필요 |
| 6 | loadbalancer | ALB, Target Group | ASG 연동 전 TG 필요 |
| 7 | asg | Launch Template, Auto Scaling Group | 모든 리소스 완성 후 |

---

## group_vars/main.yml — 중앙 변수 관리

모든 role에서 공통으로 사용하는 변수를 한 곳에서 관리합니다.

```yaml
region: "ap-northeast-2"
prefix: "user03"          # 모든 리소스 이름 prefix로 사용
vpc_cidr: "172.43.0.0/16"
ami_id: "ami-084a56dceed3eb9bb"   # Ubuntu 24.04 LTS (서울 리전)
instance_type: "t3.medium"
jenkins_fixed_ip: "172.43.64.100" # Jenkins EC2 고정 IP
```

**prefix 패턴**: 모든 리소스 이름을 `user03-{리소스명}` 형식으로 통일했습니다.
여러 사용자가 동일한 AWS 계정을 공유하는 학습 환경 특성상,
리소스 충돌 방지 및 소유자 식별을 위해 사용자명을 prefix로 적용했습니다.

**서브넷 리스트 구조**:
```yaml
pub_subnets:
  - { cidr: "172.43.0.0/24", az: "ap-northeast-2a", name: "pub-a" }
  - { cidr: "172.43.16.0/24", az: "ap-northeast-2c", name: "pub-c" }
```
리스트 형태로 관리해 role 내부에서 `loop`으로 반복 처리가 가능합니다.

---

## inventory.yml

```yaml
all:
  hosts:
    localhost:
      ansible_connection: local
```

AWS 리소스를 직접 SSH로 접속해 구성하는 방식이 아닌,
Ansible이 localhost에서 AWS API(boto3)를 호출하는 방식으로 동작합니다.
별도 타겟 서버가 필요 없어 inventory가 localhost만으로 구성됩니다.

## 수동 구성 항목

아래 항목은 Ansible 공식 모듈 미지원으로 인해 AWS 콘솔에서 직접 구성했습니다.

| 항목 | 이유 |
|------|------|
| S3 버킷 | 아티팩트 저장용, Ansible 모듈 제한 |
| CodeDeploy 애플리케이션 | 모듈 미지원 |
| CodeDeploy 배포 그룹 | 모듈 미지원 |

> CLI 스크립트(`aws deploy create-application` 등)로 자동화 가능하나,
> 프로젝트 범위상 콘솔 구성으로 진행했습니다.

---

## 개선 가능한 부분

- 현재 `site.yml`은 구축(`action=main`)만 지원
- 삭제는 각 role의 `terminate.yml`을 역순으로 실행해야 하며,
  별도 `teardown.yml` 작성으로 자동화 가능 (`action=terminate` 방식으로 통일)
