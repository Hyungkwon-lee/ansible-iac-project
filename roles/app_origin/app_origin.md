# roles/app_origin

WAS EC2 원본 인스턴스를 생성하고, Docker와 CodeDeploy Agent를 설치한 뒤
AMI를 생성합니다. 생성된 AMI는 ASG의 Launch Template 기반 이미지로 사용됩니다.

---

## 생성 리소스

- App Origin EC2 (`user03-app-origin`)
- AMI (`user03-app-image-{타임스탬프}`)

---

## 실행

```bash
ansible-playbook site.yml -e "action=deploy" --tags origin
ansible-playbook playbooks/pb-app-origin.yml -e "action=deploy"
```

---

## 구축 흐름 (main.yml)

Public Subnet / SG 조회
→ App Origin EC2 생성 (user_data 포함)
→ EC2 running 상태 대기
→ user_data 완료 대기 (180초)
→ AMI 생성
→ AMI ID 출력

---

## 주요 설계 포인트

### user_data — EC2 초기 설정 자동화

EC2 생성 시 user_data로 아래 작업을 자동 수행합니다.

- Docker 설치 및 서비스 활성화
- CodeDeploy Agent 설치 및 서비스 활성화
- 설치 로그를 `/var/log/user-data.log`에 기록

```bash
# CodeDeploy Agent 설치 (리전별 S3 버킷에서 다운로드)
wget https://aws-codedeploy-{{ region }}.s3.{{ region }}.amazonaws.com/latest/install
chmod +x ./install && ./install auto
```

### 180초 추가 대기

```yaml
- name: Wait extra time for user_data completion
  ansible.builtin.pause:
    seconds: 180
```

EC2가 `running` 상태가 되어도 user_data 스크립트는 백그라운드에서 계속 실행됩니다.
AMI 생성 전에 CodeDeploy Agent 설치가 완료되어야 하므로 추가 대기를 넣었습니다.

> 개선 가능: 고정 대기 대신 CodeDeploy Agent 프로세스 확인 루프로 대체하면 더 안정적입니다.

### AMI 타임스탬프 태그

```yaml
name: "{{ prefix }}-app-image-{{ lookup('pipe', 'date +%Y%m%d%H%M%S') }}"
tags:
  Name: "{{ prefix }}-app-image"
```

이름에 타임스탬프를 붙여 여러 번 실행해도 AMI가 구분됩니다.
`Name` 태그는 고정값으로 유지해 terminate 시 태그 기준으로 조회합니다.

### App Origin EC2의 역할

이 EC2는 AMI 생성을 위한 원본 인스턴스입니다.
AMI 생성 후에는 ASG가 이 이미지를 기반으로 인스턴스를 자동 생성합니다.
원본 EC2는 이후 별도로 종료하거나 유지할 수 있습니다.

---

## 삭제 흐름 (terminate.yml)  

AMI 조회 → AMI 등록 해제 → 연결된 스냅샷 삭제 → Origin EC2 종료

AMI를 삭제해도 스냅샷은 자동으로 삭제되지 않습니다.
`subelements`로 AMI에 연결된 스냅샷을 순회하며 함께 정리합니다.

---

## 변수 (group_vars/main.yml)

| 변수 | 값 | 설명 |
|------|----|------|
| `ami_id` | `ami-084a56dceed3eb9bb` | Ubuntu 24.04 LTS (서울 리전) |
| `instance_type` | `t3.medium` | 원본 EC2 사양 |
| `key_name` | `user03-key` | SSH 키페어 |
