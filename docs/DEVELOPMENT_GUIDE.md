# 🛠️ 개발 가이드 — Production IoT Backend

[← README](../README.md)

**Version**: 4.4.0 | **Last Updated**: 2026-03-19

---

## 📋 목차

1. [개발 환경 설정](#1-개발-환경-설정)
2. [MSA 서비스 로컬 실행](#2-msa-서비스-로컬-실행)
3. [프로젝트 구조](#3-프로젝트-구조)
4. [코딩 규칙](#4-코딩-규칙)
5. [DI 컨테이너 등록 방법](#5-di-컨테이너-등록-방법)
6. [WebSocket 메시지 개발 패턴](#6-websocket-메시지-개발-패턴)
7. [Kafka 메시지 개발 패턴](#7-kafka-메시지-개발-패턴)
8. [테스트 및 디버깅](#8-테스트-및-디버깅)
9. [에러 코드 작성 가이드](#9-에러-코드-작성-가이드)
10. [빌드 및 배포](#10-빌드-및-배포)

---

## 1. 개발 환경 설정

### 필수 도구

```bash
# Bun.js (Main API)
curl -fsSL https://bun.sh/install | bash
bun --version  # 1.0.0+

# Go (Image Service)
# https://go.dev/dl/ — 1.26+
go version

# .NET 10 SDK (PLC Service)
# https://dotnet.microsoft.com/download/dotnet/10.0
dotnet --version  # 10.x
```

### Main API 초기 설정

```bash
# 클론
git clone <main-backend-repo>
cd coolingroad-backend

# 의존성 설치
bun install

# 환경 변수
cp _env .env
# .env 편집 — JWT_SECRET, DB 접속 정보 등 필수값 입력

# 인프라 (Docker)
docker compose up -d mysql mongodb redis kafka minio timescaledb

# DB 마이그레이션
bunx drizzle-kit push:mysql --config [개발 설정 파일]
bunx drizzle-kit push:pg --config [TimescaleDB 개발 설정 파일]

# 서버 실행
bun run dev
# → http://localhost:8100
```

---

## 2. MSA 서비스 로컬 실행

### coolroad-image-service (Go)

```bash
# 리포지토리 클론
git clone <image-service-repo> ../coolroad-image-service
cd ../coolroad-image-service

# 환경 변수 — Main API .env 공유
cp ../<main-backend>/.env .env
# WORKER_CONCURRENCY=2, QUEUE_SIZE=30, FFMPEG_TIMEOUT=10 추가

# 사전 요구사항
# - ffmpeg 설치: sudo apt install ffmpeg (Ubuntu)
# - librdkafka: Docker 이미지에 포함됨 (로컬 빌드 시 apt install librdkafka-dev)

# Docker로 실행 (권장 — 의존성 자동 포함)
docker compose -f docker-compose.dev.yml up --build

# 또는 로컬 빌드 (CGO 필요)
CGO_ENABLED=1 go build -tags musl -o image-service .
./image-service
```

**동작 확인**

```bash
# Kafka takepicture 토픽에 테스트 메시지 발행
kafka-console-producer \
  --bootstrap-server localhost:9094 \
  --topic takepicture_cctvcam_dev

# 메시지 입력 (JSON)
{"meta":{"messageId":"01ARZ3NDEKTSV4RRFFQ69G5FAV","messageType":"cctv.takepicture","timestamp":"2026-03-19T10:00:00Z","version":"1.0.0","source":"test"},"payload":{"datarootid":"123","siteId":1,"sprayhistoryid":"456","rtspaddr":"rtsp://...","timestamp":"2026-03-19T10:00:00Z","snowflakeId":"789","interval":0}}

# Image Service 로그 확인
docker logs coolroad-image-service -f
```

---

### coolroad-plc-service (C# .NET 10)

```bash
# 리포지토리 클론
git clone <plc-service-repo> ../coolroad-plc-service
cd ../coolroad-plc-service

# FAKE PLC로 로컬 개발 (실제 장비 불필요)
docker compose up  # 기본값: PlcService__PlcType=FAKE

# 또는 dotnet CLI로 직접 실행
cd src/[PLC 서비스 프로젝트]
dotnet run
```

**appsettings.Development.json 설정**

```json
{
  "PLC 서비스": {
    "Environment": "development",
    "PlcType": "FAKE",
    "KafkaBrokers": "localhost:9094,localhost:9095,localhost:9096",
    "MySqlConnectionString": "Server=localhost;Port=3306;Database=smartroad_dev;...",
    "TimescaleDbConnectionString": "Host=localhost;Port=5433;Database=ts_db;...",
    "DefaultSprayDuration": 3,
    "MaxPlcQueueSize": 50
  }
}
```

**동작 확인 — Kafka 메시지 발행으로 분사 테스트**

```bash
# sendmsg_plcapi_dev 토픽에 START_SPRAY 발행
kafka-console-producer \
  --bootstrap-server localhost:9094 \
  --topic sendmsg_plcapi_dev

# 메시지 (key: start_spray)
{"meta":{"messageId":"01ARZ3NDEKTSV4RRFFQ69G5FAV","messageType":"plc.control.instruct","timestamp":"2026-03-19T10:00:00Z","version":"1.0.0","source":"test"},"payload":{"siteId":1,"instruct":"START_SPRAY","decision":"MANUAL","duration":3}}

# PLC Service 로그 확인
docker logs coolroad-plc-service -f
```

---

### 전체 스택 로컬 실행 (Main API + MSA 서비스)

```bash
# 개발 전체 스택 한 번에 실행
docker compose \
  -f docker-compose.yml \
  -f docker-compose.dev.yml \
  -f ../coolroad-image-service/docker-compose.dev.yml \
  up

# PLC Service 포함
docker compose \
  -f docker-compose.yml \
  -f docker-compose.dev.yml \
  -f ../coolroad-image-service/docker-compose.dev.yml \
  -f ../coolroad-plc-service/docker-compose.dev.yml \
  up
```

---

## 3. 프로젝트 구조

> v4.4.0 기준. 레이어별 역할 분리가 적용되어 있으며, PLC 관련 모듈은 C# MSA 서비스로 이관됐다.

```
src/
├── configs/          # 환경 설정, DB 스키마, 유효성 검증 타입
├── coremodules/
│   ├── database/     # MySQL · TimescaleDB · MongoDB · Redis 연결 및 베이스 레포지토리
│   └── messagequeue/ # Kafka 프로듀서(벌크+DLQ) · 컨슈머
├── modules/
│   ├── auth/         # 인증·이메일·MFA 서비스
│   ├── coolingroad/  # 분사 제어·이력 서비스
│   ├── admin/        # 관리자 서비스
│   ├── maintenance/  # 유지보수 서비스
│   ├── notice/       # 공지사항 서비스
│   ├── schedule/     # 스케줄러·기상청 연동 서비스
│   ├── ai/           # AI·STT Kafka 컨슈머
│   ├── cctv/         # CCTV 스트리밍 서비스
│   ├── websocket/    # WebSocket 실시간 서비스
│   └── usecases/     # 공용 유스케이스 (사이트 소유권 조회)
├── dto/              # 응답 코드 상수
├── [DI 설정]          # DI 컨테이너 전체 등록
└── [서버 진입점]      # 서버 엔트리 포인트
```


## 4. 코딩 규칙

### Controller 구조 패턴

```typescript
@singleton()
export class MyController extends 베이스 컨트롤러 {
    constructor(
        @inject(MySQL 연결 모듈) private mysql: MySQL 연결 모듈,
        @inject(MongoDB 로거) private logger: MongoDB 로거,
        @inject("snowflake_my") private snowflake: Snowflake ID 생성기,
        @inject(인증 가드) private guard: 인증 가드,
        @inject(MyService) private service: MyService,  // Service DI 주입
    ) { super() }

    async onbind(app: Elysia): Promise<void> {
        const authRoute = new Elysia()
            .state({ log: {} as RequestLog, authority: {} as AuthGuardResponse })
            .use(this.guard.authguard({ roles: ["USER"] }))
            .onRequest(this.interceptRequest(app))          // v4.2.3 팩토리
            .onAfterResponse(this.logResponse(this.logger)) // v4.2.3 팩토리
            .group('/api/myroute', (app) => app
                .get('/', async ({ store }) => {
                    if (store.authority.authcode !== [에러 코드])
                        return { code: store.authority.authcode };
                    return toJsonSafe(await this.service.getData());
                })
            );
        app.use(authRoute);
    }
}
```

### 응답 코드 사용

```typescript
// ✅ ResponseCodes 사용
return { code: [에러 코드], data: toJsonSafe(result.data) };
return { code: [에러 코드] 오류 };
return { code: [에러 코드] 조회 결과 };

// ❌ 하드코딩 금지
return { code: [하드코딩 금지] }; // ResponseCodes 사용
return { code: [에러 코드] 오류 };
```

### BigInt 직렬화

```typescript
// ✅ toJsonSafe 래핑 필수 (BigInt → string 자동 변환)
return toJsonSafe({ code: [성공 코드], id: bigintValue });

// ❌ BigInt 직접 반환 → JSON.stringify 에러
return { id: bigintValue };
```

### 로깅

```typescript
// MongoDB 로깅 (베이스 컨트롤러 — HTTP 요청/응답)
.onRequest(this.interceptRequest(app))
.onAfterResponse(this.logResponse(this.logger))

// 콘솔 로깅 (개발·프로덕션 모두 출력)
consoleLog("메시지");
consoleLogError("에러", err);
```

---

## 5. DI 컨테이너 등록 방법

신규 모듈 추가 시 DI 컨테이너에 등록한다.

```typescript
// src/DI 컨테이너

import { MyService } from "./modules/mymodule/my.service";
import { MyController } from "./modules/mymodule/my.controller";

// Snowflake ID 생성기 ID (각 컨트롤러별 고유 worker-id 사용)
container.registerInstance("snowflake_my", new Snowflake ID 생성기(9));

// Service → Controller 순서로 등록
container.registerSingleton(MyService);
container.registerSingleton(MyController);
```

```typescript
// src/서버 엔트리 포인트
await container.resolve(MyController).onbind(app);
```

**등록 현황 (v4.4.0)**

| 범주 | 수 | 예시 |
|------|-----|------|
| Controllers | 9 | 인증 컨트롤러, CoolingRoadController, ... |
| Services | 9 | 인증 서비스, 분사 제어 서비스, ... |
| Repositories (MySQL) | 13 | 사용자 레포지토리, 사이트 레포지토리, ... |
| Repositories (TimescaleDB) | 2 | 기상 데이터 레포지토리, 센서 데이터 레포지토리 |
| Infrastructure | 7 | MySQL 연결 모듈, TimescaleDB 연결 모듈, MongoDBConnect, ... |
| Snowflake ID 생성기 ID | 4 | snowflake_auth, snowflake_coolingroad, ... |

---

## 6. WebSocket 메시지 개발 패턴

### 메시지 타입 정의

```typescript
// WebSocket 메시지 타입 모듈
export type TWebSocketMsg_YourMessage = {
    siteId: number;
    data: YourDataType;
    timestamp: Date;
}
```

### 토픽 및 키 등록

```typescript
// WebSocket 메시지 타입 모듈
export const WEBSOCKET_MESSAGE_KEYS = {
    YOUR_MESSAGE: "your_message_key",
} as const;

// WebSocket 컨트롤러 — MESSAGE_KEY_TO_TOPIC_MAP에 추가
const MESSAGE_KEY_TO_TOPIC_MAP = {
    [WEBSOCKET_MESSAGE_KEYS.YOUR_MESSAGE]: "ws.your.topic",
};
```

### 백엔드에서 전송

```typescript
// Kafka 프로듀서 경유 (DI 주입된 인스턴스 사용)
await this.producehelper.enqueue(
    getTopicName(KAFKA_TOPICS.SENDMSG_WEBSOCKETAPI),
    { key: WEBSOCKET_MESSAGE_KEYS.YOUR_MESSAGE, value: JSON.stringify(payload) }
);
```

### 프론트엔드 수신

```javascript
wsManager.on('ws.your.topic', (payload) => {
    console.log('Received:', payload);
});
```

---

## 7. Kafka 메시지 개발 패턴

### 메시지 발행 (Kafka 프로듀서)

```typescript
// ✅ enqueue() 사용 — 0.3초 버퍼 배치 전송 + DLQ 보장
await this.producehelper.enqueue(
    getTopicName(KAFKA_TOPICS.SENDMSG_PLCAPI),
    { key: "start_spray", value: JSON.stringify(message) }
);

// ❌ producer.send() 직접 호출 금지 — DLQ 처리 불가
```

### 새 Kafka 토픽 추가

```typescript
// Kafka 메시지 큐 모듈
export const KAFKA_TOPICS = {
    YOUR_NEW_TOPIC: "your_new_topic",
} as const;

// 환경별 토픽 이름 자동 분기
export function getTopicName(topic: string): string {
    return IS_PRODUCTION ? topic : `${topic}_dev`;
}
```

### MSA 서비스 간 계약

Kafka 메시지 구조 변경 시 반드시 [KAFKA_MESSAGE_SPEC.md](./api/KAFKA_MESSAGE_SPEC.md)를 먼저 업데이트한다. MSA 서비스(C#, Go)도 이 명세를 기준으로 타입을 정의한다.

---

## 8. 테스트 및 디버깅

### API 테스트

```bash
# Health Check (개발)
curl http://localhost:8100/health

# 로그인
curl -X POST http://localhost:8100/api/auth/signin/user \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"Test1234!@","role":"USER"}'

# 인증 API 호출
curl http://localhost:8100/api/coolingroad \
  -H "Authorization: Bearer <token>"
```

### OpenAPI 문서

```
http://localhost:8100/openapi
```

### HTML 테스트 페이지

```
http://localhost:8100/testpages/plc-monitor-v3.html    # PLC 모니터링
http://localhost:8100/testpages/cctv-streaming-test.html  # CCTV 스트리밍
```

### VS Code 디버그 설정

```json
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "bun",
            "request": "launch",
            "name": "Debug Bun",
            "program": "${workspaceFolder}/src/서버 엔트리 포인트",
            "watchMode": true
        }
    ]
}
```

### 로그 확인

```bash
# MongoDB 요청 로그
mongosh smartroad_dev
db.request_logs.find().limit(10).sort({timestamp: -1})
db.exception_log.find().limit(10).sort({timestamp: -1})

# Docker 로그
docker compose logs -f backend

# Redis 캐시 상태
redis-cli -p 6380
keys *
ttl developer:user:*
```

---

## 9. 에러 코드 작성 가이드

### DB 결과 처리 패턴

```typescript
// ✅ DB 오류와 빈 배열을 반드시 구분 (v3.8.0+)
const result = await repo.readByCondition(...);

if (result.code !== [DB 결과 코드]) {
    return { code: [에러 코드] 오류 };
}
if (emptyArray(result.data)) {
    return { code: [에러 코드] 조회 결과 };
}
return { code: [에러 코드], data: toJsonSafe(result.data) };
```

### 에러 코드 선택 기준

| 상황 | 코드 |
|------|------|
| DB 드라이버/ORM 오류 | 데이터베이스 예외 코드 |
| 조회 결과 없음 | 빈 결과 코드 |
| 관리 사용자 없음 | 관리 대상 없음 코드 |
| 관리 사이트 없음 | 관리 사이트 없음 코드 |
| 요청 데이터 오류 | 요청 데이터 에러 코드 |
| JWT 만료/무효 | 인증 만료 코드 |
| 이메일 미인증 | 이메일 인증 필요 코드 |

---

## 10. 빌드 및 배포

```bash
# Docker 이미지 빌드
docker build -t coolingroad-backend .

# 프로덕션 배포
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 전체 스택 (MSA 포함) 배포
docker compose \
  -f docker-compose.yml -f docker-compose.prod.yml \
  -f ../coolroad-image-service/docker-compose.prod.yml \
  -f ../coolroad-plc-service/docker-compose.prod.yml \
  up -d
```

상세 인프라 배포 → [DEPLOYMENT.md](./DEPLOYMENT.md)

---

**Last Updated**: 2026-03-19 | **Version**: 4.4.0
