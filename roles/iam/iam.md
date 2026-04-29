# roles/iam

EC2 인스턴스와 CodeDeploy 서비스에 필요한 IAM Role과 Instance Profile을 구성합니다.
EC2 생성 전에 실행되어야 합니다.

---

## 생성 리소스

| 리소스 | 이름 | 연결 정책 |
|--------|------|----------|
| IAM Role | `user03-app-role` | S3FullAccess, SSMManagedInstanceCore |
| IAM Role | `user03-jenkins-role` | S3FullAccess, SSMManagedInstanceCore, CodeDeployFullAccess |
| IAM Role | `user03-codedeploy-service-role` | AWSCodeDeployRole |
| Instance Profile | `user03-app-instance-profile` | app-role 연결 |
| Instance Profile | `user03-jenkins-instance-profile` | jenkins-role 연결 |

---

## 실행

```bash
ansible-playbook site.yml -e "action=deploy" --tags iam
ansible-playbook playbooks/pb-iam.yml -e "action=deploy"
```

---

## 주요 설계 포인트

### app-role vs jenkins-role 권한 차이

```yaml
# app-role
managed_policies:
  - AmazonS3FullAccess          # S3에서 아티팩트 다운로드
  - AmazonSSMManagedInstanceCore # SSM Session Manager 접속

# jenkins-role (app-role 권한 + CodeDeploy 추가)
managed_policies:
  - AmazonS3FullAccess
  - AmazonSSMManagedInstanceCore
  - AWSCodeDeployFullAccess      # CodeDeploy 배포 트리거
```

Jenkins EC2만 CodeDeploy 배포를 실행하므로 권한을 분리했습니다.
App EC2에 불필요한 권한을 부여하지 않는 최소 권한 원칙을 적용했습니다.

### Trust Policy 분리

```yaml
# EC2용 — EC2 서비스가 Role을 Assume
Principal:
  Service: ec2.amazonaws.com

# CodeDeploy용 — CodeDeploy 서비스가 Role을 Assume
Principal:
  Service: codedeploy.amazonaws.com
```

EC2 인스턴스용 Role과 CodeDeploy 서비스용 Role은
AssumeRole 주체가 다르므로 Trust Policy를 각각 정의했습니다.

### Instance Profile

EC2에 IAM Role을 직접 붙일 수 없습니다.
Instance Profile이 Role을 감싸는 컨테이너 역할을 하며,
EC2 생성 시 Instance Profile을 지정하는 방식으로 연결합니다.

---

## 삭제 흐름 (terminate.yml)

Role이 Instance Profile에 연결된 상태에서는 삭제가 불가합니다.
분리 → 삭제 순서로 진행합니다.

```
Instance Profile에서 Role 분리 (app, jenkins)
→ Instance Profile 삭제 (app, jenkins)
→ IAM Role 삭제 (app-role, jenkins-role, codedeploy-service-role)
```
