# Amazon EKS & MCP 완벽 실습 가이드

> AWS Console 수동 구성부터 Cursor의 Claude Code를 활용한 AI 자동화까지

**최종 업데이트**: 2025-11-12
**실습 환경**: macOS, AWS EKS 1.32, Cursor IDE

---

## 📚 목차

1. [소개](#소개)
2. [MCP란 무엇인가](#mcp란-무엇인가)
3. [사전 준비](#사전-준비)
4. [PART 1: AWS 환경 구성](#part-1-aws-환경-구성)
5. [PART 2: EKS 클러스터 수동 구축](#part-2-eks-클러스터-수동-구축)
6. [PART 3: Cursor MCP로 AI 자동화](#part-3-cursor-mcp로-ai-자동화)
7. [실습 예제](#실습-예제)
8. [리소스 정리](#리소스-정리)
9. [트러블슈팅](#트러블슈팅)
10. [참고 자료](#참고-자료)

---

## 소개

### 이 가이드의 목표

1. **AWS Console 수동 구성**: 클릭 하나하나의 의미를 이해하며 EKS 학습
2. **Cursor + MCP 활용**: AI와 대화하듯 쿠버네티스 관리
3. **실전 배포 경험**: Nginx 배포부터 트러블슈팅까지

### 학습 방법

```
Phase 1: 수동 구성 (2시간)
├─ AWS 환경 구성
├─ EKS 클러스터 생성
├─ 노드 그룹 추가
└─ 애플리케이션 배포

Phase 2: AI 자동화 (1시간)
├─ MCP 서버 설치
├─ Cursor 설정
└─ Claude Code로 관리
```

### 예상 비용

- **실습 3시간**: 약 $0.80 (1,000원)
- **1일 방치 시**: 약 $6.42 (8,000원)
- ⚠️ **실습 후 즉시 삭제 필수!**

---

## MCP란 무엇인가

### Model Context Protocol (MCP)

**MCP** = AI가 외부 도구를 사용할 수 있게 해주는 표준 프로토콜

### MCP 아키텍처

```
┌─────────────────────────────────────┐
│   Cursor IDE (MCP Host)             │  ← MCP 서버들을 실행/관리
│                                     │
│  ┌──────────────────────────────┐  │
│  │  Claude Code (MCP Client)    │  │  ← AI 어시스턴트
│  │  - 채팅창 (Cmd+L)             │  │
│  └──────────┬───────────────────┘  │
│             │ MCP Protocol          │
│             ↓                       │
│  ┌──────────────────────────────┐  │
│  │  eks-mcp-server              │  │  ← 실제 작업 수행
│  │  (내 Mac에서 실행)            │  │
│  └──────────┬───────────────────┘  │
└─────────────┼───────────────────────┘
              │ kubectl/AWS API
              ↓
         AWS EKS (클라우드)
```

### 구성 요소

| 구성 요소 | 역할 | 예시 |
|----------|------|------|
| **MCP Host** | MCP 서버 실행 환경 | Cursor IDE |
| **MCP Client** | AI 어시스턴트 | Claude Code |
| **MCP Server** | 실제 작업 수행 | eks-mcp-server |

### MCP가 해결하는 문제

**MCP 없이:**
```
사용자: "EKS 클러스터 상태 보여줘"
AI: "죄송합니다. 직접 조회할 수 없습니다."
```

**MCP 사용:**
```
사용자: "EKS 클러스터 상태 보여줘"
AI: → eks-mcp-server 호출 → AWS API 실행 → 결과 분석
AI: "my-first-eks-cluster가 ACTIVE 상태입니다. 노드 2개 Running..."
```

---

## 사전 준비

### 1. AWS 계정 생성

1. https://aws.amazon.com/ko/free/ 접속
2. 계정 생성 (신용카드 등록 필요)
3. IAM 사용자 생성 권장 (루트 계정 사용 지양)

### 2. AWS Access Key 생성

#### IAM 콘솔에서 생성

1. AWS Console → IAM → 사용자
2. 사용자 선택 (또는 신규 생성)
3. "보안 자격 증명" 탭 → "액세스 키 만들기"
4. 사용 사례: **"Command Line Interface (CLI)"** 선택
5. 키 정보 저장 (한 번만 표시!)

```
액세스 키 ID: AKIAIOSFODNN7EXAMPLE
보안 액세스 키: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

⚠️ **중요**: Secret Key는 다시 볼 수 없습니다!

### 3. 로컬 환경 설정

#### macOS

```bash
# AWS CLI 설치
brew install awscli

# kubectl 설치
brew install kubectl

# 버전 확인
aws --version
kubectl version --client

# AWS 자격 증명 설정
aws configure
```

#### 입력 정보

```bash
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: ap-northeast-2
Default output format [None]: json
```

#### 설정 확인

```bash
# 현재 계정 확인
aws sts get-caller-identity

# 출력:
# {
#     "UserId": "AIDAI...",
#     "Account": "123456789012",
#     "Arn": "arn:aws:iam::123456789012:user/my-user"
# }
```

---

## PART 1: AWS 환경 구성

### 1-1. IAM 역할 생성

#### EKS 클러스터 역할

**AWS Console → IAM → 역할**

1. "역할 만들기" 클릭
2. 신뢰할 수 있는 엔터티: **"AWS 서비스"**
3. 사용 사례: **"EKS - Cluster"** 선택
4. 권한: `AmazonEKSClusterPolicy` (자동 선택)
5. 역할 이름: `eksClusterRole`
6. "역할 만들기" 클릭

#### 노드 그룹 역할

1. "역할 만들기" 클릭
2. 신뢰할 수 있는 엔터티: **"AWS 서비스"** → **"EC2"**
3. 권한 정책 3개 선택:
   - `AmazonEKSWorkerNodePolicy`
   - `AmazonEC2ContainerRegistryReadOnly`
   - `AmazonEKS_CNI_Policy`
4. 역할 이름: `eksNodeGroupRole`
5. "역할 만들기" 클릭

### 1-2. VPC 생성 (CloudFormation)

**AWS Console → CloudFormation**

1. "스택 생성" → "새 리소스 사용(표준)"
2. 템플릿 소스: **"Amazon S3 URL"**
3. URL 입력:
```
https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml
```
4. 스택 이름: `eks-vpc-stack`
5. 파라미터: 기본값 사용
6. "전송" 클릭
7. ⏱️ **5분 대기** (상태: CREATE_COMPLETE)

#### VPC 정보 확인

CloudFormation → eks-vpc-stack → "출력" 탭

- `VpcId`: VPC ID 복사
- `SubnetIds`: 서브넷 4개 ID 복사
- `SecurityGroups`: 보안 그룹 ID 복사

---

## PART 2: EKS 클러스터 수동 구축

### 2-1. EKS 클러스터 생성

**AWS Console → EKS → 클러스터**

#### 클러스터 구성

1. "클러스터 추가" → "생성" 클릭

**화면 1: 이름 및 역할**
- 이름: `my-first-eks-cluster`
- Kubernetes 버전: `1.32`
- 클러스터 서비스 역할: `eksClusterRole`
- **Bootstrap cluster administrator access**: ✓ 체크 권장
- "다음" 클릭

**화면 2: 네트워킹**
- VPC: `eks-vpc-stack-VPC` 선택
- 서브넷: 4개 모두 체크
- 클러스터 엔드포인트 액세스: 퍼블릭 ✓, 프라이빗 ✓
- "다음" 클릭

**화면 3: 관찰 가능성**
- 제어 영역 로깅: 전체 활성화 (선택 사항)
- "다음" 클릭

**화면 4-5: 애드온**
- 기본 애드온 유지 (VPC CNI, kube-proxy, CoreDNS)
- "다음" 클릭

**화면 6: 검토 및 생성**
- "생성" 클릭
- ⏱️ **15-20분 대기** ☕

### 2-2. 노드 그룹 생성

**클러스터 상세 → 컴퓨팅 탭**

1. "노드 그룹 추가" 클릭

**화면 1: 이름 및 역할**
- 이름: `my-node-group`
- 노드 IAM 역할: `eksNodeGroupRole`
- "다음" 클릭

**화면 2: 컴퓨팅 구성**
- AMI: `Amazon Linux 2023 (AL2023_x86_64_STANDARD)`
- 용량 유형: `온디맨드`
- 인스턴스: `t3.medium`
- 디스크: `20 GiB`
- 조정 구성:
  - 최소: `2`
  - 최대: `2`
  - 원하는 크기: `2`
- "다음" 클릭

**화면 3: 네트워크**
- 서브넷: **프라이빗 서브넷 2개만 선택**
- SSH 액세스: 체크 안 함
- "다음" 클릭

**화면 4: 생성**
- "생성" 클릭
- ⏱️ **5-10분 대기**

### 2-3. kubectl 설정

#### kubeconfig 업데이트

```bash
# kubeconfig 설정
aws eks update-kubeconfig \
  --region ap-northeast-2 \
  --name my-first-eks-cluster

# 출력:
# Added new context arn:aws:eks:ap-northeast-2:123456789012:cluster/my-first-eks-cluster to ~/.kube/config
```

#### 연결 확인

```bash
# 클러스터 정보
kubectl cluster-info

# 노드 확인
kubectl get nodes

# 출력:
# NAME                                               STATUS   ROLES    AGE     VERSION
# ip-172-31-39-85.ap-northeast-2.compute.internal    Ready    <none>   3h15m   v1.32.9-eks-c39b1d0
# ip-172-31-58-246.ap-northeast-2.compute.internal   Ready    <none>   3h15m   v1.32.9-eks-c39b1d0
```

### 2-4. Nginx 배포

#### Deployment 생성

```bash
# 작업 디렉토리
mkdir -p ~/eks-demo
cd ~/eks-demo

# Deployment 매니페스트
cat > nginx-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF

# 배포
kubectl apply -f nginx-deployment.yaml
```

#### Service 생성

```bash
# Service 매니페스트
cat > nginx-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
EOF

# 배포
kubectl apply -f nginx-service.yaml
```

#### 배포 확인

```bash
# Deployment 확인
kubectl get deployment nginx-deployment

# Pod 확인
kubectl get pods -l app=nginx

# Service 확인 (EXTERNAL-IP 할당까지 2-3분 대기)
kubectl get service nginx-service

# 출력:
# NAME            TYPE           EXTERNAL-IP                                        PORT(S)
# nginx-service   LoadBalancer   af84e3655b778428399446429d592ba0-1779607503...   80:32762/TCP
```

#### 접속 테스트

```bash
# LoadBalancer URL 추출
export LB_URL=$(kubectl get service nginx-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "URL: http://$LB_URL"

# curl 테스트
curl http://$LB_URL
```

**브라우저에서 접속**: http://[LoadBalancer-URL]

Nginx 환영 페이지가 보이면 성공! 🎉

---

## PART 3: Cursor MCP로 AI 자동화

### 3-1. EKS MCP 서버 설치

```bash
# uvx 설치
pip3 install uvx

# 또는 pipx 사용
brew install pipx
pipx ensurepath

# 설치 확인
uvx awslabs.eks-mcp-server --help
```

### 3-2. Cursor 설치

1. https://cursor.sh/ 에서 다운로드
2. 설치 및 실행
3. Claude Code 기본 통합 확인

### 3-3. MCP 서버 구성

#### ~/.cursor/mcp.json 편집

```bash
# 파일 열기
open ~/.cursor/mcp.json

# 또는
code ~/.cursor/mcp.json
```

#### eks-mcp-server 추가

```json
{
  "mcpServers": {
    "eks-mcp-server": {
      "command": "uvx",
      "args": [
        "awslabs.eks-mcp-server",
        "--allow-write",
        "--allow-sensitive-data-access"
      ],
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR"
      }
    }
  }
}
```

**중요 설정값:**
- `--allow-write`: EKS 리소스 생성/수정 권한
- `--allow-sensitive-data-access`: 민감 데이터 접근 권한

#### Cursor 재시작

파일 저장 후 Cursor 완전 종료 → 재실행

### 3-4. MCP 연결 확인

1. **Cmd+L** (Mac) / **Ctrl+L** (Windows) - Claude Code 채팅창 열기
2. 하단/상단에서 MCP 서버 연결 상태 확인
3. 테스트:

```
List my EKS clusters
```

Claude가 클러스터 정보를 조회하면 성공! ✅

---

## 실습 예제

### 예제 1: 클러스터 정보 조회

**Cursor Claude Code에 입력:**

```
Show me detailed information about my EKS cluster "my-first-eks-cluster"
```

**Claude가 자동으로:**
- 클러스터 상태, 버전 조회
- 노드 정보 확인
- 네트워크 구성 표시
- 리소스 요약

### 예제 2: Deployment 상태 확인

```
Check the status of nginx-deployment and show me the pod details
```

**Claude가 자동으로:**
- Deployment 상태 조회 (3/3 ready)
- Pod 목록 및 위치 확인
- 리소스 사용량 표시
- 이벤트 확인

**예상 결과:**
```
nginx-deployment: 3/3 replicas ready

Pods:
- nginx-deployment-6fd79d5db-kxvwb: Running on ip-172-31-58-246
- nginx-deployment-6fd79d5db-lt522: Running on ip-172-31-39-85
- nginx-deployment-6fd79d5db-t2tq6: Running on ip-172-31-58-246

Resources: CPU 1-2m, Memory 3Mi per pod
```

### 예제 3: 새 애플리케이션 배포

```
Create and deploy a simple "Hello EKS" application:
- Use nginx image
- Create a deployment named "hello-eks" with 2 replicas
- Expose it via LoadBalancer service on port 80
```

**Claude가 자동으로:**
1. Kubernetes 매니페스트 생성
2. Deployment 생성
3. Service (LoadBalancer) 생성
4. 배포 상태 확인
5. LoadBalancer URL 제공

### 예제 4: 트러블슈팅

#### 문제 상황 만들기

```bash
# 터미널에서 잘못된 이미지로 배포
kubectl create deployment test-fail --image=nginx:wrong-version
```

#### Claude에게 해결 요청

```
The "test-fail" deployment pods are not starting.
Diagnose the issue and fix it.
```

**Claude가 자동으로:**
1. Pod 상태 확인 → ImagePullBackOff 발견
2. Pod 이벤트 조회
3. 로그 확인
4. 문제 원인 설명: "이미지 태그 'wrong-version'이 존재하지 않음"
5. 해결 방법 제시: `kubectl set image deployment/test-fail nginx=nginx:latest`
6. 수정 후 재확인

### 예제 5: 스케일링

```
Scale the nginx-deployment to 5 replicas and monitor the rollout
```

**Claude가 자동으로:**
- Deployment 스케일 조정
- Rollout 상태 모니터링
- 새 Pod 배치 확인
- 최종 상태 보고

---

## 리소스 정리

⚠️ **중요: 실습 종료 후 반드시 모든 리소스 삭제!**

### 삭제 순서

#### 1. Kubernetes 리소스 삭제

```bash
# Service 삭제 (LoadBalancer 먼저!)
kubectl delete service nginx-service

# Deployment 삭제
kubectl delete deployment nginx-deployment

# 전체 확인
kubectl get all
```

#### 2. 노드 그룹 삭제

**AWS Console:**
1. EKS → 클러스터 → my-first-eks-cluster
2. 컴퓨팅 탭 → my-node-group 선택
3. "삭제" 버튼
4. 노드 그룹 이름 입력: `my-node-group`
5. "삭제" 클릭
6. ⏱️ 5-10분 대기

#### 3. EKS 클러스터 삭제

**AWS Console:**
1. EKS → 클러스터 목록
2. my-first-eks-cluster 선택
3. "삭제" 버튼
4. 클러스터 이름 입력: `my-first-eks-cluster`
5. "삭제" 클릭
6. ⏱️ 10-15분 대기

#### 4. CloudFormation 스택 삭제

**AWS Console:**
1. CloudFormation 콘솔
2. eks-vpc-stack 선택
3. "삭제" 버튼
4. "삭제" 확인
5. ⏱️ 5분 대기

#### 5. IAM 역할 삭제

**AWS Console:**
1. IAM → 역할
2. eksClusterRole 선택 → 삭제
3. eksNodeGroupRole 선택 → 삭제

#### 6. 추가 리소스 확인

- **EC2 LoadBalancer**: EC2 콘솔에서 수동 삭제 (남아있을 경우)
- **CloudWatch 로그 그룹**: 선택 사항

### 삭제 완료 체크리스트

- [ ] Service (LoadBalancer) 삭제
- [ ] Deployment 삭제
- [ ] 노드 그룹 삭제
- [ ] EKS 클러스터 삭제
- [ ] CloudFormation 스택 삭제
- [ ] IAM 역할 2개 삭제
- [ ] LoadBalancer 삭제 확인

---

## 트러블슈팅

### 문제 1: kubectl 인증 실패

**증상:**
```bash
kubectl get nodes
# error: You must be logged in to the server (Unauthorized)
```

**원인:** Access Entry 미설정

**해결:**
```bash
# 1. 현재 IAM 사용자 확인
aws sts get-caller-identity

# 2. Access Entry 추가
aws eks create-access-entry \
  --cluster-name my-first-eks-cluster \
  --principal-arn arn:aws:iam::[계정ID]:user/[사용자명] \
  --type STANDARD \
  --region ap-northeast-2

# 3. 관리자 권한 부여
aws eks associate-access-policy \
  --cluster-name my-first-eks-cluster \
  --principal-arn arn:aws:iam::[계정ID]:user/[사용자명] \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster \
  --region ap-northeast-2

# 4. kubeconfig 재설정
aws eks update-kubeconfig \
  --region ap-northeast-2 \
  --name my-first-eks-cluster
```

### 문제 2: LoadBalancer IP가 <pending>

**증상:**
```bash
kubectl get svc
# nginx-service   LoadBalancer   <pending>   80:31234/TCP
```

**해결:**
1. 2-3분 대기 (정상)
2. 서브넷 태그 확인:
   - 퍼블릭: `kubernetes.io/role/elb: 1`
   - 프라이빗: `kubernetes.io/role/internal-elb: 1`
3. IAM 노드 역할 권한 확인

### 문제 3: Pod가 Pending

**증상:**
```bash
kubectl get pods
# nginx-xxx   0/1   Pending   0   5m
```

**해결:**
```bash
# Pod 상세 정보
kubectl describe pod [POD-NAME]

# 이벤트 확인
kubectl get events --sort-by='.lastTimestamp'

# 노드 리소스 확인
kubectl top nodes

# 원인:
# - 리소스 부족 → 노드 추가 또는 인스턴스 타입 확장
# - 스케줄링 제약 → nodeSelector, taint/toleration 확인
```

### 문제 4: MCP 연결 안 됨

**해결:**
1. Cursor 재시작
2. `~/.cursor/mcp.json` 파일 확인
3. uvx 설치 확인:
```bash
which uvx
uvx awslabs.eks-mcp-server --help
```
4. AWS 자격 증명 확인:
```bash
aws configure list
```

---

## 참고 자료

### 공식 문서

- [Amazon EKS 사용자 가이드](https://docs.aws.amazon.com/eks/)
- [Kubernetes 문서](https://kubernetes.io/docs/)
- [EKS MCP Server GitHub](https://github.com/awslabs/aws-mcp-servers)
- [MCP 공식 사이트](https://modelcontextprotocol.io/)
- [Cursor 공식 문서](https://docs.cursor.com/)

### AWS 블로그

- [EKS MCP 서버 발표 (영문)](https://aws.amazon.com/blogs/aws/accelerating-application-development-with-the-amazon-eks-model-context-protocol-server/)
- [EKS MCP 서버 발표 (한국어)](https://aws.amazon.com/ko/blogs/tech/accelerating-application-development-with-the-amazon-eks-model-context-protocol-server/)

### 커뮤니티

- [AWS 한국 사용자 모임 (AWSKRUG)](https://www.facebook.com/groups/awskrug/)
- [Kubernetes Slack](https://kubernetes.slack.com/)
- [AWS re:Post](https://repost.aws/)

### 학습 리소스

- [EKS Workshop](https://www.eksworkshop.com/)
- [AWS Skill Builder](https://explore.skillbuilder.aws/)
- [kubectl 치트 시트](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

## 용어 사전

| 용어 | 설명 |
|------|------|
| **EKS** | Amazon Elastic Kubernetes Service - AWS 관리형 Kubernetes |
| **MCP** | Model Context Protocol - AI와 외부 도구를 연결하는 프로토콜 |
| **MCP Host** | MCP 서버를 실행하는 환경 (Cursor IDE) |
| **MCP Client** | AI 어시스턴트 (Claude Code) |
| **MCP Server** | 실제 작업을 수행하는 도구 (eks-mcp-server) |
| **Pod** | Kubernetes의 최소 배포 단위 (1개 이상의 컨테이너) |
| **Deployment** | Pod 복제본을 관리하는 리소스 |
| **Service** | Pod에 네트워크 접근을 제공하는 리소스 |
| **LoadBalancer** | 외부 트래픽을 Pod로 분산하는 서비스 타입 |
| **kubectl** | Kubernetes 명령줄 도구 |
| **kubeconfig** | 클러스터 접속 정보를 담은 설정 파일 (~/.kube/config) |
| **IAM 역할** | AWS 리소스가 다른 서비스에 접근할 수 있는 권한 |
| **VPC** | Virtual Private Cloud - AWS 가상 네트워크 |
| **CloudFormation** | 인프라를 코드로 관리하는 AWS 서비스 |

---

## 마치며

### 🎉 축하합니다!

이 가이드를 통해 다음을 배웠습니다:

#### 핵심 개념

1. **MCP** = AI와 도구를 연결하는 프로토콜
2. **MCP Host (Cursor)** = 서버 실행 환경
3. **MCP Client (Claude Code)** = AI 어시스턴트
4. **MCP Server (eks-mcp-server)** = 실제 작업 수행

#### 실습 완료 내용

- ✅ AWS 환경 구성 (IAM, VPC)
- ✅ EKS 클러스터 수동 구축
- ✅ kubectl 설정 및 Nginx 배포
- ✅ Cursor MCP 설정
- ✅ Claude Code로 AI 자동화

#### 주요 이점

- AI와 대화하듯 Kubernetes 관리
- kubectl 명령어 암기 불필요
- 자동 트러블슈팅
- 빠른 프로토타이핑

### 다음 학습 단계

1. **보안 강화**
   - IRSA (IAM Roles for Service Accounts)
   - Network Policy
   - Secrets Manager 통합

2. **모니터링**
   - CloudWatch Container Insights
   - Prometheus + Grafana

3. **CI/CD**
   - GitHub Actions
   - ArgoCD (GitOps)

4. **고급 기능**
   - Horizontal Pod Autoscaler
   - Cluster Autoscaler
   - Fargate 프로파일

### 질문이나 문제?

- [AWS re:Post](https://repost.aws/)
- [AWSKRUG](https://www.facebook.com/groups/awskrug/)
- [EKS MCP GitHub Issues](https://github.com/awslabs/aws-mcp-servers/issues)

---

**최종 체크**: 실습 완료 후 모든 리소스 삭제했나요? ✓

**즐거운 클라우드 여정 되세요!** 🚀☁️

---

**작성일**: 2025-11-12
**버전**: 1.0
**환경**: macOS, AWS EKS 1.32, Cursor IDE, Claude Code
