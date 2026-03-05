# 아키텍처 — Production IoT Backend

[← README](../README.md)

**Version**: 3.8.0 | **Last Updated**: 2026-03-04

---

## 📋 목차

1. [전체 시스템 구조](#전체-시스템-구조)
2. [레이어 구조](#레이어-구조)
3. [Coremodules 구성](#coremodules-구성)
4. [핵심 데이터 흐름](#핵심-데이터-흐름)
5. [Kafka 벌크 전송 아키텍처](#kafka-벌크-전송-아키텍처)
6. [PLC 명령 처리 흐름](#plc-명령-처리-흐름)
7. [WebSocket 메시지 흐름](#websocket-메시지-흐름)
8. [보안 아키텍처](#보안-아키텍처)
9. [성능 최적화 전략](#성능-최적화-전략)

---

## 전체 시스템 구조

```mermaid
graph TB
    subgraph Client["🖥️ Client Layer"]
        WEB["Web / Mobile App"]
        ADMIN["Admin Dashboard"]
    end

    subgraph Gateway["🛡️ Nginx Gateway"]
        RL["Rate Limiting"]
        SSL["SSL/TLS 종료"]
        CCTV_P["CCTV 프록시"]
        MINIO_P["MinIO 캐시 프록시"]
        SPA["SPA 정적 서빙"]
    end

    subgraph App["⚡ Application (Bun.js + ElysiaJS)"]
        AUTH["Auth Controller"]
        COOLING["CoolingRoad Controller"]
        WS["WebSocket Controller"]
        PLC_C["PLC Controller"]
        SCHED["Scheduler Controller"]
        AI["AI Controller"]
        ADMIN_C["Admin Controller"]
        MAINT["Maintenance Controller"]
        NOTICE["Notice Controller"]
    end

    subgraph Core["🔧 Core Modules"]
        AUTHGUARD["authguard\nJWT+RBAC"]
        KAFKA_MOD["messagequeue\nKafka Helper+DLQ"]
        MYSQL_MOD["mysql\nBaseRepository"]
        MONGO_MOD["mongodb\nMongoLogger"]
        REDIS_MOD["redis\nCaching"]
        IMAGE["image\nFFmpeg WebP"]
        SEMAPHORE["semaphore\n동시성 제어"]
    end

    subgraph Store["💾 Storage"]
        MY[("MySQL\n15 tables")]
        MG[("MongoDB\n5 collections")]
        RD[("Redis")]
        MN[("MinIO")]
    end

    subgraph External["🌐 External"]
        PLC["PLC Devices\nModbus TCP"]
        KMA["기상청 API"]
        OLLAMA["Ollama LLM"]
        CAM["CCTV Cameras\nRTSP/ISAPI"]
    end

    WEB & ADMIN --> RL --> App
    App --> Core
    Core --> MY & MG & RD & MN
    PLC_C --> PLC
    SCHED --> KMA
    AI --> OLLAMA
    PLC_C --> CAM
    COOLING & SCHED --> KAFKA_MOD --> PLC_C
    WS --> KAFKA_MOD

    style KAFKA_MOD fill:#f39c12,color:#fff
    style AUTHGUARD fill:#e74c3c,color:#fff
```

---

## 레이어 구조

```mermaid
graph TB
    subgraph Presentation["Presentation Layer — 9 Controllers"]
        C1["Auth · 963 lines"]
        C2["CoolingRoad · 759 lines"]
        C3["WebSocket · 965 lines"]
        C4["PLC · 810 lines"]
        C5["Scheduler · 579 lines"]
        C6["AI · 453 lines"]
        C7["Admin · 157 lines"]
        C8["Maintenance · 246 lines"]
        C9["Notice · 209 lines"]
    end

    subgraph Business["Business Logic Layer"]
        S1["EmailService"]
        S2["MFAService"]
        S3["helper.coremodule\nevaluateActionConditions"]
    end

    subgraph DAL["Data Access Layer — Core Modules"]
        D1["mysql.coremodule\nBaseRepository<T>"]
        D2["mongodb.coremodule\nMongoLogger + TrashboxJWT"]
        D3["redis.coremodule\nCaching + Session"]
        D4["messagequeue.coremodule\nKafkaProducerHelper + KafkaConsumerHelper"]
        D5["authguard.coremodule\ncreateGuard() + RBAC"]
    end

    subgraph Infra["Infrastructure"]
        I1["di.ts — tsyringe DI Container"]
        I2["schema.ts — Drizzle ORM 15 Tables"]
        I3["configs.ts — ENV 중앙 관리"]
        I4["dto.responsecode.ts — 응답 코드"]
    end

    Presentation --> Business --> DAL --> Infra
```

### BaseRepository 패턴

모든 DB 접근은 제네릭 `BaseRepository<T>`를 통해 일관된 인터페이스로 처리:

```typescript
class BaseRepository<TTable extends MySqlTable> {
    async create(data: TTable['_']['inferInsert']): Promise<DBResult>
    async readById(id: bigint): Promise<DBResult<TTable['_']['inferSelect']>>
    async readByCondition<K>(
        where?: (t: TTable) => SQL | undefined,
        orderby?: { column: K; direction: 'asc' | 'desc' },
        limit?: number
    ): Promise<DBResult<Array<TTable['_']['inferSelect']>>>
    async updateById(id: bigint, data: Partial<...>): Promise<DBResult>
    async deleteById(id: bigint): Promise<DBResult>
}
```

---

## Coremodules 구성

```
src/coremodules/
├── authguard.coremodule.ts     JWT 인증 + RBAC (createGuard 통합)
├── avatar.coremodule.ts        아바타 리소스 관리
├── datetime.coremodule.ts      날짜/시간 헬퍼
├── idgenerator.coremodule.ts   Snowflake ID 생성 (7개 인스턴스, worker별 분리)
├── image.coremodule.ts         FFmpeg → WebP 직접 인코딩
├── helper.coremodule.ts        evaluateActionConditions / withTimeout / LogBuffer
├── semaphore.coremodule.ts     Semaphore 동시성 제어
├── stream.coremodule.ts        ByteStream 헬퍼
└── database/
    ├── mysql.coremodule.ts     MySQL + BaseRepository
    ├── mongodb.coremodule.ts   MongoDB + MongoLogger + TrashboxJWT
    └── redis.coremodule.ts     Redis + 캐싱/세션
messagequeue/
    ├── messagequeue.coremodule.ts  KafkaProducerHelper + KafkaConsumerHelper + DLQ
    └── kafka.messagetypes.ts       토픽 정의 + ULID + 타입가드
```

**DI 컨테이너 (tsyringe)**:
```typescript
// di.ts — 모든 의존성 싱글턴 등록
container.registerSingleton(MySqlConnect)
container.registerSingleton(MongoDBConnect)
container.registerSingleton(RedisConnect)
container.registerSingleton(MongoLogger)
container.registerSingleton(TrashboxJWT)
container.registerSingleton(AuthGuard)
container.registerSingleton(KafkaProducerHelper)

// Snowflake ID — 컨트롤러별 고유 worker-id
container.registerInstance("snowflake_auth",      new Snowflake(1))
container.registerInstance("snowflake_admin",     new Snowflake(2))
container.registerInstance("snowflake_plc",       new Snowflake(3))
container.registerInstance("snowflake_mainservice", new Snowflake(4))
// ...
```

---

## 핵심 데이터 흐름

### 살수 명령 전체 흐름

```mermaid
sequenceDiagram
    participant C as Client
    participant N as Nginx
    participant B as Backend
    participant K as Kafka
    participant P as PLC (Modbus TCP)
    participant WS as WebSocket
    participant DB as MySQL/Mongo/Redis

    C->>N: 분사 명령 요청 (CoolingRoad API)
    N->>N: Rate Limit 체크 (20r/m)
    N->>B: 통과
    B->>B: JWT + RBAC 검증
    B->>DB: 멱등성 체크 (Redis Set + MySQL plc_command)
    B->>K: START_SPRAY 이벤트 enqueue
    B->>C: 202 Accepted
    Note over K: 0.3초 후 벌크 전송
    K->>P: Modbus TCP 분사 명령
    P-->>K: 완료
    K->>WS: ws.spray.status 발행
    WS->>C: 실시간 알림
    B->>DB: MongoDB 분사 로그 기록
```

### 자동 분사 흐름

```mermaid
sequenceDiagram
    participant Cron
    participant Sched as Scheduler
    participant DB as MySQL
    participant Helper
    participant Kafka

    Cron->>Sched: 매시간 15분 (COOLINGROAD) / 20분 (CLEANROAD)
    Sched->>DB: useAuto=true 사이트 조회
    loop 각 사이트
        Sched->>DB: 최신 센서 + 기상 데이터
        Sched->>Helper: evaluateActionConditions()
        alt 조건 만족
            Helper-->>Sched: true
            Sched->>Kafka: START_SPRAY (decision: USER_SETTING)
        else
            Helper-->>Sched: false
        end
    end
```

---

## Kafka 벌크 전송 아키텍처

```mermaid
graph TB
    subgraph Producers["Producer Layer"]
        A1["WebSocket Controller"]
        A2["PLC Controller"]
        A3["AI Controller"]
        A4["Scheduler Controller"]
    end

    subgraph Helper["KafkaProducerHelper"]
        B1["enqueue()\n버퍼에 추가"]
        B2["flushBuffer()\n0.3초마다 실행\ntry-finally 보호"]
        B3["토픽별 그룹화\n최대 100개"]
    end

    subgraph Kafka["Kafka Cluster"]
        C1["sendmsg_plcapi"]
        C2["websocket_messages"]
        C3["stt-request"]
    end

    subgraph DLQ["실패 처리"]
        D1["kafka_dlq (MySQL)\nPENDING → RETRYING\n→ RESOLVED / FAILED"]
    end

    A1 & A2 & A3 & A4 -->|"enqueue()"| B1
    B1 -->|"Every 0.3s"| B2 --> B3
    B3 -->|"bulk send"| C1 & C2 & C3
    B3 -->|"전송 실패"| D1

    style B2 fill:#f39c12,color:#fff
    style D1 fill:#e74c3c,color:#fff
```

**Graceful Shutdown 보장**:
```typescript
async gracefullyShutdown() {
    while (this.buffer.length > 0) {
        await this.flushBuffer()  // 버퍼가 완전히 빌 때까지
    }
    clearInterval(this.flushtimer)  // 타이머 먼저 정지
    await this.producer.disconnect()
}
```

---

## PLC 명령 처리 흐름

```mermaid
graph TB
    subgraph Command["plc.command.ts — 명령 큐 관리"]
        STATE["PLCState\n- isLive: boolean\n- address: string\n- modbus: ModbusRTU\n- queue: PLCCommand[]\n- processing: boolean\n- canHandle: boolean"]
        ENQ["enqueueCommand()\n우선순위 지원 (addfirst)"]
        DEQ["dequeueCommand()\n순차 실행\ntry-catch 보호"]
        RESET["setPLCStateReset()\n장애 시 상태 초기화"]
    end

    subgraph Adapter["plc.modbusadapter.ts — 어댑터"]
        REAL["ModbusPLCAdapter\nModbus TCP 실제 통신"]
        FAKE["FakePLCAdapter\nRandom Data (개발용)"]
    end

    ENV["PLCTYPE=REAL/FAKE"] -.-> REAL
    ENV -.-> FAKE
    ENQ --> STATE --> DEQ --> REAL & FAKE
    DEQ -->|"실패"| RESET

    style REAL fill:#27ae60,color:#fff
    style FAKE fill:#f39c12,color:#fff
```

**PLC LOCAL 모드 차단** (v3.6.8):
```typescript
// canHandle=false (LOCAL 모드)일 때 원격 분사 거부
async startSpray(siteId, ...) {
    if (!plc_state.canHandle) {
        await this.logger.logPLCInstruction({ tags: ['local'], ... })
        return  // 거부
    }
    // 분사 진행
}
```

---

## WebSocket 메시지 흐름

```mermaid
graph LR
    subgraph Services["발신 서비스"]
        PLC_C["PLC Controller"]
        SCHED["Scheduler"]
        AUTH_C["Auth Controller"]
        MAINT_C["Maintenance"]
    end

    subgraph Kafka["Kafka\nwebsocket_messages 토픽"]
        K["message.key로\n메시지 타입 구분"]
    end

    subgraph WS["WebSocket Controller"]
        CONSUMER["KafkaConsumerHelper\nBATCH_SIZE=5"]
        ROUTER["메시지 라우팅\n구독 토픽 매칭"]
    end

    subgraph Clients["클라이언트"]
        C1["User A"]
        C2["User B"]
        C3["Admin"]
    end

    PLC_C -->|"plc_coil_data\nplc_register_data\nspray_status"| K
    SCHED -->|"weather_update"| K
    AUTH_C -->|"email_verification"| K
    MAINT_C -->|"maintenance_alert"| K
    K --> CONSUMER --> ROUTER
    ROUTER -->|"구독 토픽 필터링"| C1 & C2 & C3
```

**구현된 WebSocket 메시지 타입 (6가지)**:

| Key | Topic | 설명 |
|-----|-------|------|
| `plc_coil_data` | `ws.plc.coil.data` | PLC 코일 상태 + 고장코드 (~5초) |
| `plc_register_data` | `ws.plc.register.data` | 센서 데이터 (~1분) |
| `weather_update` | `ws.weather.update` | 날씨 데이터 (~1시간) |
| `spray_status` | `ws.spray.status` | 분사 상태 변경 (이벤트) |
| `maintenance_alert` | `ws.maintenance.alert` | 유지보수 완료 알림 (이벤트) |
| `stt_response` | `ws.stt.response` | 음성 명령 결과 (이벤트) |

---

## 보안 아키텍처

### 인증 흐름

```mermaid
sequenceDiagram
    participant C as Client
    participant B as Backend
    participant M as MySQL
    participant MFA as MFA Service

    C->>B: 로그인 요청 (Auth API)
    B->>M: 이메일+역할로 사용자 조회
    M-->>B: 사용자 정보
    B->>B: bcrypt 비밀번호 검증
    alt MFA 활성화된 계정
        B->>C: MFA 요구 응답
        C->>B: TOTP 코드 입력
        B->>MFA: TOTP 검증 (타이밍 공격 방지)
        MFA-->>B: 검증 결과
    end
    B->>M: jwt_token_version 증가
    B->>B: JWT HS512 발급
    B-->>C: JWT Token
```

### MongoDB 로그 수집 구조

```typescript
// MongoLogger — LogBuffer 배치 처리 (200건 or 1초마다)
class MongoLogger {
    logRequest(channel: LogChannel, log: RequestLog)   // API 요청 로그
    logError(payload: ErrorLogDocument)                 // 에러 로그
    logWS(payload: WebsocketLogDocument)               // WebSocket 로그
    logScheduler(payload: SchedulerLogDocument)        // 스케줄러 로그
    logPLCInstruction(payload: PLCLogDocument)         // PLC 명령 로그
}
```

---

## 성능 최적화 전략

### 1. Cursor 기반 페이지네이션

```typescript
// limit+1 패턴 — 추가 COUNT 쿼리 없이 다음 페이지 여부 판단
const result = await repo.readByCondition(
    (t) => cursor ? lte(t.id, cursor) : undefined,
    { column: 'id', direction: 'desc' },
    limit  // DB에는 limit+1 전달
)
const hasmore = result.data.length > limit
if (hasmore) result.data.pop()
```

### 2. Redis 캐싱 전략

| 캐시 키 | 내용 | TTL |
|---------|------|-----|
| `weather:{siteId}` | 기상청 API 응답 | 1시간 |
| `session:{userId}` | JWT 세션 | 24시간 |
| `spray_site:{id}` | 분사 중인 사이트 (멱등성) | 분사 시간 + 5분 |

### 3. FFmpeg 이미지 파이프라인

```
Before: RTSP → FFmpeg → PNG buffer → Sharp → WebP  (2단계, 중간 버퍼 있음)
After:  RTSP → FFmpeg → WebP 직접 출력             (1단계)

효과: 메모리 40% 감소, 처리 속도 30% 향상, Sharp 의존성 제거
```

### 4. MongoDB 인덱스 전략

```javascript
// 시간순 조회 최적화
db.web_request_logs.createIndex({ timestamp: -1 })
db.web_request_logs.createIndex({ userid: 1, timestamp: -1 })

// 태그별 에러 필터링
db.error_logs.createIndex({ tags: 1, timestamp: -1 })

// PLC 사이트별 이벤트
db.plc_logs.createIndex({ siteId: 1, timestamp: -1 })
```

---

[← README](../README.md)

**Last Updated**: 2026-03-04 | **Version**: 3.8.0