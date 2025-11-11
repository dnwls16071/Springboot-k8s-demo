# Jenkins Kubernetes Spring Boot Pipeline 설정 가이드

이 문서는 EC2에서 Jenkins + Kubernetes + Spring Boot CI/CD 파이프라인을 설정하는 전체 과정을 설명합니다.

## 목차
1. [사전 요구사항](#사전-요구사항)
2. [EC2 환경 설정](#ec2-환경-설정)
3. [Docker Hub 설정](#docker-hub-설정)
4. [Jenkins 설정](#jenkins-설정)
5. [Kubernetes 설정](#kubernetes-설정)
6. [Git Repository 설정](#git-repository-설정)
7. [Pipeline 실행](#pipeline-실행)
8. [트러블슈팅](#트러블슈팅)

---

## 사전 요구사항

### 필요한 것들
- EC2 인스턴스 (Ubuntu 또는 Amazon Linux)
- Docker Hub 계정
- Kubernetes 클러스터 (Minikube, EKS, 또는 기타)
- Git Repository (GitHub, GitLab 등)
- Jenkins 8080 포트 (Spring Boot는 8081 사용)

---

## EC2 환경 설정

### 1. Docker 설치 및 설정

```bash
# Amazon Linux 2
sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker

# Ubuntu
sudo apt-get update
sudo apt-get install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker

# Jenkins 사용자에게 Docker 권한 부여
sudo usermod -aG docker jenkins
sudo usermod -aG docker ec2-user  # 또는 ubuntu

# 재시작 (권한 적용)
sudo systemctl restart docker
```

### 2. kubectl 설치

```bash
# kubectl 다운로드
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# 설치
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# 확인
kubectl version --client
```

### 3. envsubst 설치 (환경변수 치환용)

```bash
# Amazon Linux
sudo yum install gettext -y

# Ubuntu
sudo apt-get install gettext-base -y

# 확인
envsubst --version
```

### 4. Jenkins 설치 (이미 설치되어 있으면 생략)

```bash
# Amazon Linux 2
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum install jenkins java-17-amazon-corretto -y
sudo systemctl start jenkins
sudo systemctl enable jenkins

# 초기 비밀번호 확인
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

---

## Docker Hub 설정

### 1. Docker Hub Repository 생성

1. https://hub.docker.com 접속 및 로그인
2. **Repositories** → **Create Repository** 클릭
3. Repository 이름: `springboot-demo`
4. Visibility: `Public` 선택 (또는 Private)
5. **Create** 클릭

### 2. Access Token 생성 (보안 권장)

1. Docker Hub → **Account Settings** → **Security**
2. **New Access Token** 클릭
3. Description: `Jenkins CI/CD`
4. Access permissions: `Read, Write, Delete`
5. **Generate** 클릭
6. 생성된 토큰 복사 (다시 볼 수 없으니 안전한 곳에 저장!)

---

## Jenkins 설정

### 1. 필수 플러그인 설치

Jenkins 관리 → 플러그인 관리 → Available plugins 탭:

- [x] Pipeline
- [x] Git
- [x] Docker Pipeline
- [x] Kubernetes CLI
- [x] Credentials Binding Plugin

설치 후 Jenkins 재시작:
```bash
sudo systemctl restart jenkins
```

### 2. Credentials 등록

Jenkins 관리 → Credentials → System → Global credentials (unrestricted):

#### a) Docker Hub Credentials
- **Add Credentials** 클릭
- Kind: `Username with password`
- Username: Docker Hub 사용자명 (예: `dnwls16071`)
- Password: Docker Hub Access Token (위에서 생성한 토큰)
- ID: `docker-hub-credentials` (Jenkinsfile에서 사용)
- Description: `Docker Hub Access Token`
- **Create**

#### b) Kubeconfig Credentials
- **Add Credentials** 클릭
- Kind: `Secret file`
- File: EC2의 `~/.kube/config` 파일 업로드
- ID: `kubeconfig` (Jenkinsfile에서 사용)
- Description: `Kubernetes Config`
- **Create**

**kubeconfig 파일 가져오기:**
```bash
# EC2에서
cat ~/.kube/config

# 로컬에서 파일로 저장 후 Jenkins에 업로드
```

### 3. Jenkins Pipeline Job 생성

1. Jenkins 대시보드 → **New Item**
2. Item name: `springboot-k8s-pipeline`
3. Type: **Pipeline** 선택
4. **OK**

**Pipeline 설정:**
- Definition: `Pipeline script from SCM`
- SCM: `Git`
- Repository URL: Git Repository URL 입력
  ```
  예: https://github.com/your-username/springboot-k8s-demo.git
  ```
- Branch: `*/main` (또는 사용하는 브랜치)
- Script Path: `Jenkinsfile`
- **Save**

---

## Kubernetes 설정

### 1. kubeconfig 설정 확인

```bash
# kubectl이 클러스터에 연결되는지 확인
kubectl cluster-info
kubectl get nodes

# 안 되면 kubeconfig 설정 필요
export KUBECONFIG=/path/to/kubeconfig
```

### 2. Metrics Server 설치 (HPA 사용시 필수)

```bash
# Metrics Server 설치
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Minikube 사용시
minikube addons enable metrics-server

# 확인
kubectl top nodes
```

### 3. Namespace 생성 (선택사항)

```bash
# Jenkins Pipeline이 자동으로 생성하지만, 수동으로도 가능
kubectl apply -f deploy/namespace.yml
```

---

## Git Repository 설정

### 1. Git Repository 초기화 (아직 안했다면)

```bash
cd /Users/jwj/Desktop/test

# Git 초기화
git init

# 원격 저장소 추가
git remote add origin https://github.com/your-username/springboot-k8s-demo.git

# .gitignore 확인 (이미 있음)
cat .gitignore
```

### 2. 첫 커밋 및 푸시

```bash
# 모든 파일 추가
git add .

# 커밋
git commit -m "Initial commit: Jenkins K8s Spring Boot Pipeline"

# 푸시
git push -u origin main
```

### 3. Webhook 설정 (선택사항 - 자동 빌드)

**GitHub Webhook:**
1. GitHub Repository → Settings → Webhooks → Add webhook
2. Payload URL: `http://your-jenkins-ec2-ip:8080/github-webhook/`
3. Content type: `application/json`
4. Events: `Just the push event`
5. **Add webhook**

**Jenkins에서:**
- Pipeline Job → Configure → Build Triggers
- [x] `GitHub hook trigger for GITScm polling` 체크
- **Save**

---

## Pipeline 실행

### 1. 수동 빌드

1. Jenkins → `springboot-k8s-pipeline` Job 클릭
2. **Build Now** 클릭
3. Build History에서 빌드 번호 클릭
4. **Console Output** 확인

### 2. Pipeline 단계

```
1. Checkout      - Git에서 소스 코드 가져오기
2. Build         - Gradle로 빌드
3. Test          - 단위 테스트 실행
4. Docker Build  - Docker 이미지 빌드
5. Docker Push   - Docker Hub에 푸시
6. Deploy to K8s - Kubernetes에 배포
7. Verify        - 배포 검증
```

### 3. 배포 확인

```bash
# Namespace의 리소스 확인
kubectl get all -n springboot-demo

# Pod 로그 확인
kubectl logs -f deployment/springboot-app -n springboot-demo

# Service 확인
kubectl get svc -n springboot-demo

# 애플리케이션 접속 (NodePort 사용)
curl http://your-node-ip:30081
curl http://your-node-ip:30081/health
```

### 4. 접속 테스트

```bash
# 로컬에서 포트포워딩
kubectl port-forward -n springboot-demo svc/springboot-service 8081:8081

# 브라우저에서
http://localhost:8081
http://localhost:8081/health
http://localhost:8081/actuator/health
```

---

## 트러블슈팅

### 문제 1: Docker 권한 오류
```
ERROR: Got permission denied while trying to connect to the Docker daemon socket
```

**해결:**
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### 문제 2: kubectl이 클러스터를 찾을 수 없음
```
ERROR: The connection to the server localhost:8080 was refused
```

**해결:**
```bash
# kubeconfig 확인
echo $KUBECONFIG
cat ~/.kube/config

# Jenkins에 올바른 kubeconfig credentials 등록 확인
```

### 문제 3: Docker 이미지 Pull 실패
```
Failed to pull image "dnwls16071/springboot-demo:1"
```

**해결:**
```bash
# Docker Hub에 이미지가 푸시되었는지 확인
docker images | grep springboot-demo

# Jenkins에서 docker-hub-credentials가 올바른지 확인
```

### 문제 4: Pod가 CrashLoopBackOff 상태
```bash
kubectl get pods -n springboot-demo
# NAME                              READY   STATUS             RESTARTS
# springboot-app-xxx-yyy            0/1     CrashLoopBackOff   5
```

**해결:**
```bash
# Pod 로그 확인
kubectl logs springboot-app-xxx-yyy -n springboot-demo

# 일반적인 원인:
# 1. 애플리케이션 포트 불일치 (8081 확인)
# 2. Health check 경로 오류
# 3. 메모리 부족
```

### 문제 5: HPA가 작동하지 않음
```bash
kubectl get hpa -n springboot-demo
# TARGETS: <unknown>/60%
```

**해결:**
```bash
# Metrics Server 설치 확인
kubectl get deployment metrics-server -n kube-system

# 없으면 설치
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### 문제 6: envsubst: command not found
```
/bin/sh: envsubst: command not found
```

**해결:**
```bash
# Amazon Linux
sudo yum install gettext -y

# Ubuntu
sudo apt-get install gettext-base -y
```

### 문제 7: Gradle 빌드 실패
```
ERROR: Could not find or load main class org.gradle.wrapper.GradleWrapperMain
```

**해결:**
```bash
# gradlew 실행 권한 확인
chmod +x gradlew

# Gradle wrapper 재생성
gradle wrapper
```

---

## 주요 파일 설명

### 프로젝트 구조
```
.
├── Jenkinsfile              # Jenkins Pipeline 정의
├── Dockerfile               # Multi-stage Docker 이미지 빌드
├── build.gradle             # Gradle 빌드 설정
├── deploy/
│   ├── namespace.yml        # Kubernetes Namespace
│   ├── configMap.yml        # 환경 설정
│   ├── deployment.yml       # Pod 배포 설정
│   ├── service.yml          # Service (NodePort)
│   ├── hpa.yml              # HorizontalPodAutoscaler
│   ├── pv.yml               # PersistentVolume (옵션)
│   └── pvc.yml              # PersistentVolumeClaim (옵션)
└── src/                     # Spring Boot 소스 코드
```

### 환경변수 (Jenkinsfile에서 설정)
- `DOCKER_IMAGE_NAME`: `springboot-demo`
- `DOCKER_REGISTRY`: `dnwls16071/` (본인의 Docker Hub username)
- `IMAGE_TAG`: Jenkins 빌드 번호
- `NAMESPACE`: `springboot-demo`

---

## 다음 단계

1. **모니터링 추가**
   - Prometheus + Grafana 설치
   - Spring Boot Actuator metrics 연동

2. **로깅 설정**
   - ELK Stack (Elasticsearch, Logstash, Kibana)
   - 또는 CloudWatch Logs

3. **보안 강화**
   - Network Policies
   - Pod Security Standards
   - Secret 관리 (Vault, AWS Secrets Manager)

4. **Blue-Green 배포**
   - 무중단 배포 전략

5. **테스트 자동화**
   - Integration Tests
   - E2E Tests

---

## 참고 링크

- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Docker Hub](https://hub.docker.com/)
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)

---

## 문의

문제가 발생하면 다음을 확인하세요:
1. Jenkins Console Output
2. Kubernetes Pod Logs: `kubectl logs -f <pod-name> -n springboot-demo`
3. Kubernetes Events: `kubectl get events -n springboot-demo --sort-by='.lastTimestamp'`