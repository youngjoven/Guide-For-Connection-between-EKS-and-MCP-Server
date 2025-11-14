# MCP 서버를 EKS Pod에 배포하기

> 로컬 MCP에서 클러스터 내부 MCP로 - Pod 기반 MCP 서버 구축 가이드

**작성일**: 2025-11-12
**실습 환경**: AWS EKS 1.32, Python 3.11+, Podman 5.0+

**📌 Podman 사용**: 이 가이드는 Docker 대신 Podman을 사용합니다. Podman은 데몬리스 아키텍처로 더 안전하고 rootless 실행이 가능하며, Docker CLI와 거의 호환됩니다.

---

## 📚 목차

1. [소개](#소개)
2. [로컬 MCP vs Pod MCP 비교](#로컬-mcp-vs-pod-mcp-비교)
3. [사전 준비](#사전-준비)
4. [STEP 1: MCP 서버 애플리케이션 개발](#step-1-mcp-서버-애플리케이션-개발)
5. [STEP 2: Podman 이미지 빌드](#step-2-podman-이미지-빌드)
6. [STEP 3: ECR에 푸시](#step-3-ecr에-푸시)
7. [STEP 4: EKS에 배포](#step-4-eks에-배포)
8. [STEP 5: 테스트 및 검증](#step-5-테스트-및-검증)
9. [고급 시나리오](#고급-시나리오)
10. [트러블슈팅](#트러블슈팅)

---

## 소개

### 이 가이드의 목표

기존에는 **로컬 Mac에서 MCP 서버**를 실행했습니다. 이제 **EKS Pod 안에 MCP 서버**를 배포하여:
- 클러스터 내부 리소스에 직접 접근
- 서비스별 독립적인 MCP 서버 운영
- 확장 가능하고 안정적인 MCP 인프라 구축

### 사용 사례

**로컬 MCP (기존)**:
- EKS 클러스터 관리
- AWS 리소스 조회
- kubectl 명령 실행

**Pod MCP (신규)**:
- 클러스터 내부 데이터베이스 백업
- 캐시 관리 (Redis)
- 내부 API 호출
- 로그 수집/분석

---

## 로컬 MCP vs Pod MCP 비교

### 로컬 MCP (현재 사용 중)

```
┌──────────────────────────────┐
│  내 Mac (로컬)                │
│                              │
│  Cursor IDE (MCP Host)       │
│    ↓                         │
│  eks-mcp-server              │
│  (내 Mac에서 실행)            │
└──────────┬───────────────────┘
           │
           │ kubectl/AWS API
           ↓
      ┌─────────────┐
      │  AWS EKS    │
      └─────────────┘
```

**특징**:
- ✅ 설정 간단
- ✅ 디버깅 쉬움
- ❌ 네트워크 지연
- ❌ 클러스터 내부 리소스 접근 제한

### Pod MCP (이번에 구현)

```
┌──────────────────────────────┐
│  내 Mac (로컬)                │
│                              │
│  Cursor IDE (MCP Host)       │
│    │                         │
│    │ HTTP/WebSocket          │
└────┼──────────────────────────┘
     │
     │ Port-Forward / Ingress
     ↓
┌─────────────────────────────┐
│  AWS EKS (클라우드)          │
│                             │
│  ┌─────────────────────┐   │
│  │  Pod                │   │
│  │  ┌───────────────┐  │   │
│  │  │ MCP Server    │  │   │
│  │  │ (FastAPI)     │  │   │
│  │  └───────┬───────┘  │   │
│  │          ↓           │   │
│  │  내부 리소스 접근    │   │
│  │  - Redis            │   │
│  │  - PostgreSQL       │   │
│  │  - 내부 API         │   │
│  └─────────────────────┘   │
└─────────────────────────────┘
```

**특징**:
- ✅ 클러스터 내부 리소스 직접 접근
- ✅ 빠른 네트워크 속도
- ✅ 확장 가능 (Deployment)
- ❌ 설정 복잡
- ❌ 디버깅 어려움

### 비교표

| 구분 | 로컬 MCP | Pod MCP |
|------|----------|---------|
| **서버 위치** | 내 Mac | EKS Pod |
| **접근 방식** | kubectl/AWS API | 클러스터 내부 API |
| **네트워크** | 외부 → 클러스터 | 클러스터 내부 |
| **지연 시간** | 높음 | 낮음 |
| **내부 리소스 접근** | 제한적 | 완전한 접근 |
| **설정 난이도** | 쉬움 ⭐ | 어려움 ⭐⭐⭐ |
| **용도** | 클러스터 관리 | 애플리케이션 작업 |

---

## 사전 준비

### 1. 기존 EKS 클러스터

```bash
# 클러스터 확인
kubectl get nodes

# 출력:
# NAME                                               STATUS   ROLES    AGE
# ip-172-31-39-85.ap-northeast-2.compute.internal    Ready    <none>   3h
# ip-172-31-58-246.ap-northeast-2.compute.internal   Ready    <none>   3h
```

### 2. 필수 도구 설치

#### Podman vs Docker

| 특징 | Docker | Podman |
|------|--------|--------|
| **아키텍처** | 데몬 기반 | 데몬리스 |
| **루트 권한** | 필요 (기본) | rootless 가능 |
| **보안** | 중앙 데몬 공격 위험 | 프로세스 격리 |
| **명령어** | `docker` | `podman` (거의 동일) |

```bash
# Podman 설치 확인
podman --version

# macOS Podman 설치 (필요시)
brew install podman

# Podman 머신 초기화 (macOS)
podman machine init
podman machine start

# Podman 머신 상태 확인
podman machine list

# AWS CLI 설치 확인
aws --version

# kubectl 설치 확인
kubectl version --client
```

**💡 Tip**: Podman은 Docker CLI와 거의 호환되므로 `alias docker=podman`으로 설정하여 기존 Docker 명령어를 그대로 사용할 수 있습니다.

### 3. 작업 디렉토리 생성

```bash
mkdir -p ~/eks-mcp-pod
cd ~/eks-mcp-pod
```

---

## STEP 1: MCP 서버 애플리케이션 개발

### 1-1. 간단한 MCP 서버 예제

이 예제는 클러스터 내부 Redis를 관리하는 MCP 서버입니다.

#### requirements.txt 생성

```bash
cat > requirements.txt << 'EOF'
fastapi==0.104.1
uvicorn==0.24.0
redis==5.0.1
pydantic==2.5.0
httpx==0.25.2
EOF
```

#### app.py 생성

```python
cat > app.py << 'EOF'
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import redis
import os
import json

app = FastAPI()

# Redis 연결 설정 (클러스터 내부)
REDIS_HOST = os.getenv("REDIS_HOST", "redis-service")
REDIS_PORT = int(os.getenv("REDIS_PORT", 6379))

redis_client = redis.Redis(
    host=REDIS_HOST,
    port=REDIS_PORT,
    decode_responses=True
)

class MCPRequest(BaseModel):
    tool: str
    params: dict = {}

class MCPResponse(BaseModel):
    success: bool
    data: dict = {}
    error: str = None

@app.get("/")
async def root():
    return {"service": "MCP Server in Pod", "status": "running"}

@app.get("/health")
async def health():
    try:
        redis_client.ping()
        return {"status": "healthy", "redis": "connected"}
    except Exception as e:
        raise HTTPException(status_code=503, detail=f"Redis unavailable: {str(e)}")

@app.post("/tools/list")
async def list_tools():
    """MCP 도구 목록 반환"""
    return {
        "tools": [
            {
                "name": "redis_get",
                "description": "Get value from Redis",
                "params": ["key"]
            },
            {
                "name": "redis_set",
                "description": "Set value in Redis",
                "params": ["key", "value"]
            },
            {
                "name": "redis_keys",
                "description": "List all keys in Redis",
                "params": []
            },
            {
                "name": "redis_del",
                "description": "Delete key from Redis",
                "params": ["key"]
            }
        ]
    }

@app.post("/tools/execute")
async def execute_tool(request: MCPRequest):
    """MCP 도구 실행"""
    try:
        tool = request.tool
        params = request.params

        if tool == "redis_get":
            key = params.get("key")
            value = redis_client.get(key)
            return MCPResponse(
                success=True,
                data={"key": key, "value": value}
            )

        elif tool == "redis_set":
            key = params.get("key")
            value = params.get("value")
            redis_client.set(key, value)
            return MCPResponse(
                success=True,
                data={"key": key, "value": value, "action": "set"}
            )

        elif tool == "redis_keys":
            keys = redis_client.keys("*")
            return MCPResponse(
                success=True,
                data={"keys": keys, "count": len(keys)}
            )

        elif tool == "redis_del":
            key = params.get("key")
            result = redis_client.delete(key)
            return MCPResponse(
                success=True,
                data={"key": key, "deleted": bool(result)}
            )

        else:
            raise HTTPException(status_code=400, detail=f"Unknown tool: {tool}")

    except Exception as e:
        return MCPResponse(
            success=False,
            error=str(e)
        )

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8080)
EOF
```

### 1-2. 로컬 테스트 (선택 사항)

```bash
# 가상환경 생성
python3 -m venv venv
source venv/bin/activate

# 의존성 설치
pip install -r requirements.txt

# Redis 로컬 실행 (Podman)
podman run -d --name redis -p 6379:6379 redis:7-alpine

# MCP 서버 실행
python app.py

# 테스트 (다른 터미널)
curl http://localhost:8080/
curl http://localhost:8080/health
curl -X POST http://localhost:8080/tools/list

# 종료
deactivate
podman stop redis && podman rm redis
```

---

## STEP 2: Podman 이미지 빌드

### 2-1. Dockerfile 생성

```dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

# 의존성 복사 및 설치
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 애플리케이션 복사
COPY app.py .

# 포트 노출
EXPOSE 8080

# 헬스체크
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import httpx; httpx.get('http://localhost:8080/health')"

# 실행
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8080"]
EOF
```

### 2-2. .dockerignore 생성

**📌 Note**: Podman은 Dockerfile과 .dockerignore 파일을 그대로 지원합니다.

```bash
cat > .dockerignore << 'EOF'
venv/
__pycache__/
*.pyc
*.pyo
*.pyd
.Python
.DS_Store
*.md
EOF
```

### 2-3. Podman 이미지 빌드

```bash
# 이미지 빌드
podman build -t mcp-server:latest .

# 빌드 확인
podman images | grep mcp-server

# 이미지 상세 정보 확인
podman inspect mcp-server:latest

# 로컬 테스트 (선택 사항)
podman run -d --name mcp-test -p 8080:8080 \
  -e REDIS_HOST=host.containers.internal \
  mcp-server:latest

# 컨테이너 상태 확인
podman ps

# 로그 확인
podman logs mcp-test

# 테스트
curl http://localhost:8080/health

# 종료
podman stop mcp-test && podman rm mcp-test
```

**💡 Podman 빌드 옵션**:
```bash
# 멀티 아키텍처 빌드 (ARM64 + AMD64)
podman build --platform linux/amd64,linux/arm64 -t mcp-server:latest .

# 빌드 캐시 무시
podman build --no-cache -t mcp-server:latest .

# 빌드 과정 상세 출력
podman build --log-level=debug -t mcp-server:latest .
```

---

## STEP 3: ECR에 푸시

### 3-1. ECR 리포지토리 생성

```bash
# 리포지토리 생성
aws ecr create-repository \
  --repository-name mcp-server \
  --region ap-northeast-2

# 출력:
# {
#     "repository": {
#         "repositoryArn": "arn:aws:ecr:ap-northeast-2:123456789012:repository/mcp-server",
#         "registryId": "123456789012",
#         "repositoryName": "mcp-server",
#         "repositoryUri": "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/mcp-server"
#     }
# }
```

### 3-2. ECR 로그인

```bash
# 계정 ID 확인
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo $AWS_ACCOUNT_ID

# ECR 로그인 (Podman)
aws ecr get-login-password --region ap-northeast-2 | \
  podman login --username AWS --password-stdin \
  ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com
```

### 3-3. 이미지 태그 및 푸시

```bash
# 이미지 태그
podman tag mcp-server:latest \
  ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com/mcp-server:latest

# ECR에 푸시
podman push ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com/mcp-server:latest

# 푸시 확인
aws ecr describe-images \
  --repository-name mcp-server \
  --region ap-northeast-2
```

---

## STEP 4: EKS에 배포

### 4-1. Redis 배포 (테스트용)

MCP 서버가 사용할 Redis를 먼저 배포합니다.

```bash
cat > redis-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  selector:
    app: redis
  ports:
  - protocol: TCP
    port: 6379
    targetPort: 6379
  type: ClusterIP
EOF

# 배포
kubectl apply -f redis-deployment.yaml

# 확인
kubectl get pods -l app=redis
kubectl get service redis-service
```

### 4-2. MCP 서버 배포

```bash
# 환경 변수 설정
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Deployment 매니페스트 생성
cat > mcp-server-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-server
  labels:
    app: mcp-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mcp-server
  template:
    metadata:
      labels:
        app: mcp-server
    spec:
      containers:
      - name: mcp-server
        image: ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com/mcp-server:latest
        ports:
        - containerPort: 8080
        env:
        - name: REDIS_HOST
          value: "redis-service"
        - name: REDIS_PORT
          value: "6379"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: mcp-server-service
spec:
  selector:
    app: mcp-server
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer
EOF

# 배포
kubectl apply -f mcp-server-deployment.yaml
```

### 4-3. 배포 확인

```bash
# Pod 확인
kubectl get pods -l app=mcp-server

# 출력:
# NAME                          READY   STATUS    RESTARTS   AGE
# mcp-server-5d59d67564-abc12   1/1     Running   0          1m
# mcp-server-5d59d67564-def34   1/1     Running   0          1m

# Service 확인
kubectl get service mcp-server-service

# LoadBalancer URL 확인 (2-3분 대기)
export MCP_URL=$(kubectl get service mcp-server-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "MCP Server URL: http://$MCP_URL"

# Pod 로그 확인
kubectl logs -l app=mcp-server --tail=50
```

---

## STEP 5: 테스트 및 검증

### 5-1. 기본 테스트

```bash
# Health Check
curl http://$MCP_URL/health

# 출력:
# {"status":"healthy","redis":"connected"}

# 도구 목록
curl -X POST http://$MCP_URL/tools/list

# 출력:
# {
#   "tools": [
#     {"name": "redis_set", ...},
#     {"name": "redis_get", ...},
#     ...
#   ]
# }
```

### 5-2. Redis 작업 테스트

```bash
# Redis에 값 저장
curl -X POST http://$MCP_URL/tools/execute \
  -H "Content-Type: application/json" \
  -d '{
    "tool": "redis_set",
    "params": {
      "key": "test-key",
      "value": "Hello from Pod MCP!"
    }
  }'

# 출력:
# {
#   "success": true,
#   "data": {
#     "key": "test-key",
#     "value": "Hello from Pod MCP!",
#     "action": "set"
#   }
# }

# Redis에서 값 조회
curl -X POST http://$MCP_URL/tools/execute \
  -H "Content-Type: application/json" \
  -d '{
    "tool": "redis_get",
    "params": {
      "key": "test-key"
    }
  }'

# 출력:
# {
#   "success": true,
#   "data": {
#     "key": "test-key",
#     "value": "Hello from Pod MCP!"
#   }
# }

# 모든 키 조회
curl -X POST http://$MCP_URL/tools/execute \
  -H "Content-Type: application/json" \
  -d '{
    "tool": "redis_keys",
    "params": {}
  }'

# 키 삭제
curl -X POST http://$MCP_URL/tools/execute \
  -H "Content-Type: application/json" \
  -d '{
    "tool": "redis_del",
    "params": {
      "key": "test-key"
    }
  }'
```

### 5-3. Port-Forward로 직접 접근

LoadBalancer 대신 Port-Forward로 테스트할 수도 있습니다.

```bash
# Port-Forward 시작
kubectl port-forward service/mcp-server-service 8080:80

# 다른 터미널에서 테스트
curl http://localhost:8080/health
curl -X POST http://localhost:8080/tools/list
```

### 5-4. Cursor에서 사용 (고급)

MCP 서버를 HTTP로 노출했으므로, Cursor의 `mcp.json`에서 HTTP 클라이언트로 연결할 수 있습니다.

**참고**: 현재 MCP 프로토콜은 stdio 기반이므로, HTTP 어댑터를 별도로 구현해야 합니다. 이는 고급 주제이므로 별도 섹션에서 다룹니다.

---

## 고급 시나리오

### 시나리오 1: PostgreSQL 백업 MCP 서버

```python
# app.py에 추가
import psycopg2
import subprocess
from datetime import datetime

@app.post("/tools/execute")
async def execute_tool(request: MCPRequest):
    # ... 기존 코드 ...

    elif tool == "postgres_backup":
        # PostgreSQL 백업
        db_host = os.getenv("POSTGRES_HOST", "postgres-service")
        db_name = params.get("database")
        backup_file = f"/backups/{db_name}_{datetime.now():%Y%m%d_%H%M%S}.sql"

        cmd = f"pg_dump -h {db_host} -U postgres {db_name} > {backup_file}"
        subprocess.run(cmd, shell=True, check=True)

        return MCPResponse(
            success=True,
            data={"backup_file": backup_file, "size": os.path.getsize(backup_file)}
        )
```

### 시나리오 2: 내부 API 프록시 MCP 서버

```python
import httpx

@app.post("/tools/execute")
async def execute_tool(request: MCPRequest):
    # ... 기존 코드 ...

    elif tool == "internal_api_call":
        # 클러스터 내부 API 호출
        service_name = params.get("service")
        endpoint = params.get("endpoint")
        method = params.get("method", "GET")

        url = f"http://{service_name}.default.svc.cluster.local{endpoint}"

        async with httpx.AsyncClient() as client:
            if method == "GET":
                response = await client.get(url)
            elif method == "POST":
                response = await client.post(url, json=params.get("data", {}))

            return MCPResponse(
                success=True,
                data={
                    "status_code": response.status_code,
                    "body": response.json()
                }
            )
```

### 시나리오 3: 로그 수집 및 분석

```python
@app.post("/tools/execute")
async def execute_tool(request: MCPRequest):
    # ... 기존 코드 ...

    elif tool == "analyze_logs":
        # Pod 로그 분석
        namespace = params.get("namespace", "default")
        label_selector = params.get("label")

        cmd = f"kubectl logs -n {namespace} -l {label_selector} --tail=1000"
        result = subprocess.run(cmd, shell=True, capture_output=True, text=True)

        logs = result.stdout
        error_count = logs.count("ERROR")
        warning_count = logs.count("WARNING")

        return MCPResponse(
            success=True,
            data={
                "total_lines": len(logs.splitlines()),
                "errors": error_count,
                "warnings": warning_count,
                "sample": logs.splitlines()[-10:]  # 마지막 10줄
            }
        )
```

---

## 트러블슈팅

### 문제 1: Pod가 ImagePullBackOff

**증상:**
```bash
kubectl get pods
# mcp-server-xxx   0/1   ImagePullBackOff   0   2m
```

**해결:**
```bash
# ECR 인증 확인
aws ecr describe-repositories --region ap-northeast-2

# 이미지 존재 확인
aws ecr describe-images --repository-name mcp-server --region ap-northeast-2

# Pod 상세 정보
kubectl describe pod -l app=mcp-server

# 일반적인 원인:
# 1. 이미지 이름 오타
# 2. ECR 인증 문제
# 3. IAM 권한 부족

# 해결: IAM 노드 역할에 ECR 권한 추가
# AmazonEC2ContainerRegistryReadOnly 정책 확인
```

### 문제 2: Redis 연결 실패

**증상:**
```bash
curl http://$MCP_URL/health
# {"detail":"Redis unavailable: ..."}
```

**해결:**
```bash
# Redis Pod 확인
kubectl get pods -l app=redis

# Redis Service 확인
kubectl get service redis-service

# Redis 연결 테스트
kubectl run redis-test --rm -it --image=redis:7-alpine -- redis-cli -h redis-service ping

# 출력: PONG

# MCP 서버 로그 확인
kubectl logs -l app=mcp-server

# 환경 변수 확인
kubectl exec -it $(kubectl get pod -l app=mcp-server -o jsonpath='{.items[0].metadata.name}') -- env | grep REDIS
```

### 문제 3: LoadBalancer IP가 할당 안 됨

**증상:**
```bash
kubectl get service mcp-server-service
# EXTERNAL-IP   <pending>
```

**해결:**
```bash
# 1. 2-3분 대기

# 2. 서브넷 태그 확인 (VPC 콘솔)
# 퍼블릭 서브넷: kubernetes.io/role/elb = 1

# 3. 대안: NodePort 사용
kubectl patch service mcp-server-service -p '{"spec":{"type":"NodePort"}}'

# 4. 대안: Port-Forward 사용
kubectl port-forward service/mcp-server-service 8080:80
```

### 문제 4: 메모리/CPU 부족

**증상:**
```bash
kubectl get pods
# mcp-server-xxx   0/1   Pending   0   5m

kubectl describe pod mcp-server-xxx
# Warning: FailedScheduling ... Insufficient cpu/memory
```

**해결:**
```bash
# 리소스 요청량 줄이기
kubectl edit deployment mcp-server

# resources 섹션 수정:
# requests:
#   memory: "64Mi"
#   cpu: "50m"

# 또는 노드 추가
# EKS Console → 노드 그룹 → 원하는 크기 증가
```

### 문제 5: Health Check 실패

**증상:**
```bash
kubectl get pods
# mcp-server-xxx   0/1   CrashLoopBackOff   3   2m
```

**해결:**
```bash
# Pod 로그 확인
kubectl logs -l app=mcp-server --tail=100

# 이벤트 확인
kubectl get events --sort-by='.lastTimestamp' | grep mcp-server

# Health Check 엔드포인트 직접 테스트
kubectl exec -it $(kubectl get pod -l app=mcp-server -o jsonpath='{.items[0].metadata.name}') -- curl localhost:8080/health

# Health Check 설정 완화 (임시)
kubectl edit deployment mcp-server
# initialDelaySeconds: 30 증가
# periodSeconds: 60 증가
```

### 문제 6: Podman 머신 오류 (macOS)

**증상:**
```bash
podman build -t mcp-server:latest .
# Error: cannot connect to Podman socket
```

**해결:**
```bash
# Podman 머신 상태 확인
podman machine list

# 머신이 stopped 상태라면 시작
podman machine start

# 머신이 없다면 초기화
podman machine init
podman machine start

# 재시작이 필요한 경우
podman machine stop
podman machine start

# 완전히 재설정 (마지막 수단)
podman machine rm
podman machine init
podman machine start
```

### 문제 7: ECR 푸시 시 인증 오류

**증상:**
```bash
podman push ...
# Error: authentication required
```

**해결:**
```bash
# ECR 로그인 재시도
aws ecr get-login-password --region ap-northeast-2 | \
  podman login --username AWS --password-stdin \
  ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com

# AWS 자격 증명 확인
aws sts get-caller-identity

# Podman 로그인 상태 확인
cat ~/.docker/config.json

# 또는 (Podman 전용 위치)
cat $XDG_RUNTIME_DIR/containers/auth.json
```

---

## 리소스 정리

### 단계별 삭제

```bash
# 1. MCP 서버 삭제
kubectl delete -f mcp-server-deployment.yaml

# 2. Redis 삭제
kubectl delete -f redis-deployment.yaml

# 3. ECR 이미지 삭제
aws ecr batch-delete-image \
  --repository-name mcp-server \
  --image-ids imageTag=latest \
  --region ap-northeast-2

# 4. ECR 리포지토리 삭제
aws ecr delete-repository \
  --repository-name mcp-server \
  --region ap-northeast-2 \
  --force

# 5. 로컬 Podman 이미지 삭제
podman rmi mcp-server:latest
podman rmi ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com/mcp-server:latest
```

---

## 다음 단계

### 1. 보안 강화

- **RBAC 설정**: ServiceAccount 생성 및 권한 제한
- **Secret 관리**: Redis 비밀번호를 Kubernetes Secret으로 관리
- **Network Policy**: Pod 간 통신 제한

### 2. 모니터링

- **로그 수집**: Fluentd/Fluent Bit로 CloudWatch Logs 전송
- **메트릭**: Prometheus로 MCP 서버 메트릭 수집
- **알림**: 에러 발생 시 Slack/Email 알림

### 3. 고가용성

- **Horizontal Pod Autoscaler**: CPU/메모리 기반 자동 스케일링
- **PodDisruptionBudget**: 최소 가용 Pod 수 보장
- **Multi-AZ 배포**: 노드를 여러 AZ에 분산

### 4. CI/CD 파이프라인

- **GitHub Actions**: 코드 푸시 시 자동 빌드/배포
- **ArgoCD**: GitOps 기반 배포 자동화

---

## 참고 자료

### 공식 문서

- [Kubernetes Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [AWS ECR User Guide](https://docs.aws.amazon.com/ecr/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)

### 관련 가이드

- `EKS_MCP_완벽가이드.md` - 로컬 MCP 서버 구축 가이드

---

## 마치며

### 🎉 완료!

이제 다음을 배웠습니다:

- ✅ MCP 서버를 Python FastAPI로 개발
- ✅ Docker 이미지 빌드 및 ECR 푸시
- ✅ EKS Pod에 MCP 서버 배포
- ✅ 클러스터 내부 Redis와 통신
- ✅ HTTP API로 MCP 도구 실행

### 주요 차이점

| 항목 | 로컬 MCP | Pod MCP |
|------|----------|---------|
| 실행 위치 | 내 Mac | EKS Pod |
| 통신 방식 | stdio | HTTP API |
| 내부 리소스 | 제한적 | 완전한 접근 |
| 확장성 | 불가 | Deployment 복제 |

### 실전 활용

Pod MCP는 다음 상황에 유용합니다:

1. **데이터베이스 작업**: 백업, 복원, 마이그레이션
2. **캐시 관리**: Redis 키 관리, 만료 설정
3. **내부 API 호출**: 마이크로서비스 간 통신
4. **로그 분석**: 실시간 로그 수집 및 분석
5. **배치 작업**: 정기적인 유지보수 작업

---

**작성일**: 2025-11-12
**버전**: 1.0
**테스트 환경**: AWS EKS 1.32, Python 3.11, FastAPI 0.104, Podman 5.0+

**즐거운 클라우드 여정 되세요!** 🚀☁️
