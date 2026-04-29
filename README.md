# ansible-iac-project

Ansible을 활용한 AWS 인프라 IaC(Infrastructure as Code) 구성 프로젝트입니다.  
→ 프로젝트 전체 개요는 [infra-portfolio](https://github.com/Hyungkwon-lee/infra-portfloio.git) 참고

---

## 인프라 구성 흐름

```
network → security → iam → app_origin → jenkins → loadbalancer → asg
```

```bash
# 전체 실행
source scripts/token.sh {MFA_CODE}
ansible-playbook site.yml -e "action=main"
```

---

## 디렉토리 구조 및 문서

| 경로 | 설명 | 문서 |
|------|------|------|
| `site.yml` | 전체 실행 흐름, MFA 인증, 변수 구성 | [site.md](./site.md) |
| `roles/iam` | IAM Role, Instance Profile | [iam.md](./roles/iam/iam.md) |
| `roles/network` | VPC, Subnet, IGW, NAT, Route Table | [network.md](./roles/network/network.md) |
| `roles/security` | Security Group | [security.md](./roles/security/security.md) |
| `roles/app_origin` | WAS EC2, AMI 생성 | [app_origin.md](./roles/app_origin/app_origin.md) |
| `roles/jenkins` | Jenkins EC2 | [jenkins.md](./roles/jenkins/jenkins.md) |
| `roles/loadbalancer` | ALB, Target Group | [loadbalancer.md](./roles/loadbalancer/loadbalancer.md) |
| `roles/asg` | Launch Template, Auto Scaling Group | [asg.md](./roles/asg/asg.md) |

---

## 담당 작업

- AWS 인프라 전체 설계 및 Ansible Playbook 작성
- MFA 기반 STS 임시 Credential 인증 구성
- Role 단위 모듈화 및 멱등성 처리
- 구축/삭제 양방향 Playbook 구성

---

## 기술 스택

`Ansible` `AWS` `VPC` `EC2` `ALB` `ASG` `IAM` `CodeDeploy` `S3`  
`amazon.aws` `community.aws`
