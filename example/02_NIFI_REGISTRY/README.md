# NiFi Registry 학습 예제

## 📌 개요

Apache NiFi Registry를 Docker Compose로 구성하여 NiFi 플로우의 버전 관리를 학습하는 예제입니다.

## 🏗️ 아키텍처

```
┌─────────────┐         ┌──────────────────┐
│    NiFi     │ ◄────►  │  NiFi Registry   │
│   (8443)    │         │    (18443)       │
└─────────────┘         └──────────────────┘
       │                         │
       └───────────┬─────────────┘
                   │
             [nifi-network]
```

## 🚀 시작하기

### 1. 환경 시작

```bash
# 프로젝트 디렉토리로 이동
cd /path/to/nifi/example/02_NIFI_REGISTRY

# Registry 데이터 디렉토리 생성 (권한 문제 방지)
mkdir -p registry-data/database registry-data/flow_storage

# Docker Compose로 NiFi와 Registry 시작
docker-compose up -d

# 컨테이너 상태 확인 (최대 60초 대기)
docker-compose ps

# Registry API 동작 확인
curl -f http://localhost:18080/nifi-registry-api/buckets

# 로그 확인 (필요시)
docker-compose logs -f nifi-registry
```

### 2. 접속 정보

- **NiFi UI**: https://localhost:8443/nifi
- **Registry UI**: http://localhost:18080/nifi-registry
- **NiFi 계정 정보**:
  - Username: `admin`
  - Password: `password1234`
- **Registry**: 별도 인증 없음 (HTTP 모드)

## 📚 버전 관리 개념

### Process Group 단위 버전 관리

NiFi Registry는 **Process Group 단위로 버전을 관리**합니다. 개별 Processor나 Connection이 아닌 Process Group을 하나의 버전 관리 단위로 사용합니다.

#### 버전 관리 가능/불가능 항목

✅ **버전 관리 가능**
- Process Group (프로세스 그룹)
- 중첩된 Process Group (각각 독립적으로 관리)

❌ **버전 관리 불가능**
- 개별 Processor
- 개별 Connection
- Root Canvas (최상위 캔버스)

#### 버전 관리 구조

```
NiFi Registry
└── Bucket (예: dev-flows)
    ├── Flow 1 (Process Group: ETL Pipeline)
    │   ├── Version 1 - 초기 버전
    │   ├── Version 2 - PutFile 추가
    │   └── Version 3 - 오류 처리 추가
    └── Flow 2 (Process Group: Data Validation)
        ├── Version 1 - 초기 검증 로직
        └── Version 2 - 추가 검증 규칙
```

#### Process Group 버전에 포함되는 내용

- ✅ 모든 Processor와 설정값
- ✅ Connection (프로세서 간 연결)
- ✅ Controller Service 설정
- ✅ Variables (프로세스 그룹 변수)
- ✅ 중첩된 Process Group (참조 형태)
- ❌ 실제 데이터
- ❌ 민감한 정보 (패스워드 등)

#### 독립적 버전 관리 예시

```
Main Process Group (v3)
├── Sub Process Group A (v5)  # 독립적 버전 관리
├── Sub Process Group B (v2)  # 독립적 버전 관리
└── Sub Process Group C       # 버전 관리 안 함
```

#### 왜 Process Group 단위인가?

1. **모듈화**: 관련 프로세서들을 논리적 단위로 그룹화
2. **재사용성**: 다른 NiFi 인스턴스에서 Import 가능
3. **독립성**: 각 팀이 자신의 Process Group만 관리
4. **유연성**: 전체 플로우 중 일부만 선택적 버전 관리

#### 실무 구조 예시

```
Root Canvas (버전 관리 X)
├── [PG] 데이터 수집 (v10)        # 팀 A가 관리
│   ├── GetFile
│   ├── GetHTTP
│   └── ConsumeKafka
├── [PG] 데이터 변환 (v7)         # 팀 B가 관리
│   ├── ConvertRecord
│   ├── JoltTransformJSON
│   └── ReplaceText
├── [PG] 데이터 저장 (v5)         # 팀 C가 관리
│   ├── PutDatabaseRecord
│   └── PutHDFS
└── 임시 테스트 Processor          # 버전 관리 안 함
```

## 📝 실습 과정

### Step 1: Registry Client 설정

1. NiFi UI 접속 (https://localhost:8443/nifi)
2. 우측 상단 메뉴 → Controller Settings 클릭
3. Registry Clients 탭 선택
4. ➕ 버튼으로 새 Registry Client 추가
   ```
   Name: Local Registry
   URL: http://nifi-registry:18080
   Description: 로컬 개발 환경 Registry
   ```

### Step 2: Registry에 Bucket 생성

1. Registry UI 접속 (http://localhost:18080/nifi-registry)
2. Settings → New Bucket
3. Bucket 정보 입력
   ```
   Bucket Name: dev-flows
   Description: 개발 환경 플로우 저장소
   ```

### Step 3: Process Group 버전 관리

1. NiFi UI에서 새 Process Group 생성

   - Canvas 빈 공간 드래그 → Process Group 추가
   - Name: `Sample ETL Pipeline`

2. Process Group에 간단한 플로우 구성

   - GetFile 프로세서 추가

3. 버전 관리 시작
   - Process Group 우클릭 → Version → Start version control
   - 설정:
     ```
     Registry: Local Registry
     Bucket: dev-flows
     Flow Name: Sample ETL Pipeline
     Flow Description: 샘플 ETL 파이프라인
     Version Comments: 초기 버전 생성
     ```

### Step 4: 버전 관리 실습

#### 변경사항 만들기

1. Process Group 더블클릭하여 진입
2. 새 프로세서 추가 (예: PutFile)
3. 기존 연결 수정

#### 변경사항 확인

- Process Group 우클릭 → Version → Show local changes
- 추가/수정/삭제된 컴포넌트 확인

#### 새 버전 커밋

- Process Group 우클릭 → Version → Commit local changes
- 커밋 메시지 입력: `PutFile 프로세서 추가`

#### 버전 히스토리 확인

1. Registry UI에서 dev-flows 버킷 클릭
2. Sample ETL Pipeline 플로우 선택
3. 버전 목록 확인

### Step 5: 버전 롤백 실습

1. NiFi UI에서 Process Group 우클릭
2. Version → Change version
3. 이전 버전 선택하여 롤백

### Step 6: 다른 NiFi 인스턴스에서 플로우 가져오기

1. Canvas 빈 공간 드래그
2. Add Process Group 선택
3. Import from Registry 옵션 선택
4. Registry, Bucket, Flow, Version 선택

## 🔍 주요 기능 탐색

### 버전 비교

- 두 버전 간 차이점 시각적 확인
- 변경 이력 추적

### 플로우 내보내기/가져오기

```bash
# Registry API로 플로우 정보 조회
curl -f http://localhost:18080/nifi-registry-api/buckets

# 특정 플로우 버전 다운로드
curl -f "http://localhost:18080/nifi-registry-api/buckets/{bucketId}/flows/{flowId}/versions/{version}/export" \
  -o flow-export.json
```

## 📊 모니터링

### 컨테이너 리소스 사용량 확인

```bash
docker stats nifi nifi-registry
```

### 볼륨 확인

```bash
# 생성된 볼륨 목록
docker volume ls | grep 02_nifi

# 볼륨 상세 정보
docker volume inspect 02_nifi_registry_nifi-conf
```

## 🛠️ 문제 해결

### 1. Registry 연결 실패

```bash
# 네트워크 연결 확인
docker exec nifi ping -c 3 nifi-registry

# Registry API 상태 확인
curl -f http://localhost:18080/nifi-registry-api/buckets

# 권한 문제시 데이터 디렉토리 재생성
rm -rf registry-data
mkdir -p registry-data/database registry-data/flow_storage
docker-compose restart nifi-registry
```

### 2. 메모리 부족

```yaml
# docker-compose.yml에 리소스 제한 추가
services:
  nifi:
    mem_limit: 2g
    environment:
      - NIFI_JVM_HEAP_INIT=512m
      - NIFI_JVM_HEAP_MAX=2g
```

### 3. 로그 확인

```bash
# NiFi 로그
docker exec nifi tail -f /opt/nifi/nifi-current/logs/nifi-app.log

# Registry 로그
docker exec nifi-registry tail -f /opt/nifi-registry/nifi-registry-current/logs/nifi-registry-app.log
```

## 🗂️ Git으로 버전 관리

### Registry 데이터 Git 관리

NiFi Registry의 플로우 버전 이력을 Git으로 관리하여 코드와 함께 버전 관리할 수 있습니다.

#### 1. Git 관리 대상 디렉토리

```
registry-data/
├── database/         # Registry 메타데이터 (버킷, 플로우 정보, 커밋 메시지)
└── flow_storage/     # 실제 플로우 파일들 (각 버전의 JSON 파일)
```

#### 2. Git 초기 설정

```bash
# .gitignore 파일 생성 (필요한 파일만 관리)
cat > .gitignore << EOF
# Docker volumes (동적 데이터)
nifi-logs/
nifi-database/
nifi-flowfile/
nifi-content/
nifi-provenance/
nifi-state/

# Temporary files
*.tmp
*.temp
*.log

# OS files
.DS_Store
Thumbs.db
EOF

# Registry 데이터를 Git에 추가
git add registry-data/
git add docker-compose.yml
git add .gitignore
git commit -m "feat: NiFi Registry 플로우 버전 관리 추가"
```

#### 3. 플로우 변경 후 Git 커밋

```bash
# NiFi에서 플로우 변경 후 Registry에 커밋
# 그 다음 Git에도 커밋
git add registry-data/
git commit -m "update: ETL 파이프라인 PutFile 프로세서 추가"
```

#### 4. 다른 환경에서 복원

```bash
# Git에서 코드와 Registry 데이터 클론
git clone <repository-url>
cd <project-directory>/example/02_NIFI_REGISTRY

# Docker 환경 시작 (registry-data가 이미 있으므로 Registry 자동 복원)
docker-compose up -d

# Registry UI에서 저장된 플로우들 확인
# http://localhost:18080/nifi-registry
# 모든 버전 이력이 그대로 복원됨
```

##### NiFi에 Process Group 복원하기

Registry는 복원되지만 **NiFi Canvas는 비어있습니다**. Process Group을 복원하려면:

1. **NiFi UI 접속** (https://localhost:8443/nifi)
2. **Registry Client 설정** (Step 1 참조)
3. **Process Group Import**:
   - Canvas 빈 공간 드래그 → Add Process Group
   - "Import from Registry" 선택
   - Registry: Local Registry
   - Bucket: 복원된 버킷 선택
   - Flow: 원하는 플로우 선택
   - Version: 최신 버전 또는 특정 버전 선택
4. **필요한 모든 Process Group 반복 Import**

💡 **참고**: Registry는 플로우의 "저장소" 역할만 합니다. 실제 실행 중인 Process Group은 NiFi에서 수동으로 Import해야 합니다.

#### 5. 추가 백업 옵션 (선택사항)

Git 외에 추가로 백업이 필요한 경우:

```bash
# 특정 시점 백업 (압축 파일로 저장)
tar -czf registry-backup-$(date +%Y%m%d).tar.gz registry-data/

# 압축 파일에서 복원
tar -xzf registry-backup-20240102.tar.gz
docker-compose restart nifi-registry
```

**💡 Git으로 관리 중이라면**: `git clone` → `docker-compose up -d`만으로 완전한 복원이 가능합니다. 별도 백업/복원 작업이 필요 없습니다.

### ⚠️ 주의사항

- **registry-data 전체 관리**: flow_storage와 database 모두 Git으로 관리해야 완전한 복원 가능
- **민감 정보 제외**: 실제 데이터나 인증 정보는 Parameter Context로 분리
- **정기적 커밋**: NiFi Registry에 커밋할 때마다 Git에도 커밋 권장

## 🧹 정리

```bash
# 컨테이너 중지 및 삭제
docker-compose down

# 볼륨도 함께 삭제 (데이터 완전 삭제)
docker-compose down -v
```

## 📚 추가 학습 자료

- [Apache NiFi Registry 공식 문서](https://nifi.apache.org/registry.html)
- [NiFi Registry REST API](https://nifi.apache.org/registry-api/)
- [버전 관리 베스트 프랙티스](https://nifi.apache.org/docs/nifi-registry-docs/)

## 💡 팁

1. **커밋 메시지**: Git처럼 명확하고 의미 있는 커밋 메시지 작성
2. **브랜치 전략**: Bucket을 환경별(dev, test, prod)로 분리
3. **백업**: Registry 데이터는 주기적으로 백업
4. **자동화**: CI/CD 파이프라인과 통합하여 배포 자동화

## ⚠️ 주의사항

- 민감한 정보(패스워드, API 키)는 버전 관리되지 않음
- Parameter Context를 활용하여 환경별 설정 분리
- Controller Service는 별도 구성 필요
