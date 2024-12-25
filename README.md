## AWS CICD Test

본 테스트는 **AWS EC2 인스턴스(Ubuntu 환경)** 와 리소스를 활용하여 GitHub Actions 를 통한 **자동화 배포**를 구축합니다.

<br />

> _Document._

- [GitHub Actions 와 AWS 로 CI/CD 구축하기](https://leeseunghee00.notion.site/GitHub-Actions-AWS-CI-CD-167889b5fe3d80f29e34fee5968ef04d?pvs=4)

<br />

> _Env._

- Java17
- IntelliJ
- AWS (EC2, S3, CodeDeploy)

<br />

> _Set._

1. 배포 스크립트: `scripts/deploy.sh`

```shell
#!/bin/bash

HOME=/home/ubuntu
REPOSITORY=your-repository  # 실행할 레포지토리 지정
...
```

<br />

2. 배포 설정 파일: `appspec.yml`

```yaml
version: 0.0    # 항상 0.0 으로 설정
os: your os     # 배포 운영체제 

# 배포할 파일 경로 지정
files:
  - source: /
    destination: /home/ubuntu/your-repository
    overwrite: yes

# 해당 파일의 소유자와 그룹 지정
permissions:
  - object: /home/ubuntu/your-repository/
    owner: ubuntu
    group: ubuntu

# 배포 특정 시점에 실행할 스크립트 정의
hooks:
  ApplicationStart:
    - location: scripts/deploy.sh
      timeout: 60
```

<br />

3. GitHub Actions 설정: `.github/workflows/aws-cicd-test.yml`
    - Actions 에서 **Java with Gradle** 로 기본 틀 생성 가능
    - AWS 변수 설정: `Settings > Actions secrets and variables` 에서 변수명과 값 설정

```yaml
name: Your CICD Title

# 실행 시점 설정
on:
  push:
    branches: [ "main" ]   # main 에 push 할 때
  pull_request:
    branches: [ "main" ]   # main 에 pr 할 때
    types: [ closed ]      # main 에 merge 할 때

# AWS 변수 설정
env:
  AWS_REGION: ap-northeast-2    # 서울 리전
  AWS_S3_BUCKET: your aws s3 bucket
  AWS_CODE_DEPLOY_APPLICATION: your aws codeDeploy
  AWS_CODE_DEPLOY_GROUP: your aws codeDeploy group

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      # Java17 설정
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      # Gradle 권한 부여 및 실행
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@af1da67850ed9a4cedd57bfd976089dd991e2582 # v4.0.0

      - name: Build with Gradle Wrapper
        run: ./gradlew build

      # 빌드 파일 압축
      - name: Make zip file
        run: zip -r ./$GITHUB_SHA.zip .
        shell: bash

      # AWS 권한 설정
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-region: ${{ env.AWS_REGION }}
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_KEY }}

      # S3 업로드
      - name: Upload to S3
        run: aws s3 cp --region $AWS_REGION ./$GITHUB_SHA.zip s3://$AWS_S3_BUCKET/$GITHUB_SHA.zip

      # SSH 등록
      - name: Add SSH Key
        run: |
          echo "${{ secrets.EC2_KEY_PAIR }}" > ~/your-key.pem
          chmod 600 ~/your-key.pem

      # SSH 디렉터리 생성
      - name: Create SSH Directory
        run: mkdir -p ~/.ssh

      # Host 키 등록
      - name: Add known host
        run: |
          ssh-keyscan -H ${{ secrets.EC2_PUBLIC_IP }} >> ~/.ssh/known_hosts

      # Deploy Script 실행
      - name: Run Deploy Script on EC2
        run: |
          ssh -T -i ~/your-key.pem ubuntu@${{ secrets.EC2_PUBLIC_IP }} << 'EOF'
          cd /home/ubuntu/your-repository
          chmod +x scripts/deploy.sh
          ./scripts/deploy.sh
          EOF
```
