# 🌡️ Smart Road Watering System - Backend Architecture

[![TypeScript](https://img.shields.io/badge/TypeScript-5.0+-blue.svg)](https://www.typescriptlang.org/)
[![Bun](https://img.shields.io/badge/Bun-1.0+-orange.svg)](https://bun.sh/)
[![Architecture](https://img.shields.io/badge/Architecture-Microservices-green.svg)]()
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**도로 살수 시스템을 위한 고성능 IoT 백엔드 아키텍처**

> 이 문서는 실무에서 설계하고 구현한 프로덕션 레벨 시스템의 아키텍처와 핵심 설계 패턴을 다룹니다.

---

## 📋 목차

- [프로젝트 개요](#-프로젝트-개요)
- [시스템 아키텍처](#-시스템-아키텍처)
- [핵심 설계 패턴](#-핵심-설계-패턴)
- [기술적 의사결정](#-기술적-의사결정)
- [성능 최적화](#-성능-최적화)
- [보안 설계](#-보안-설계)

---

## 🎯 프로젝트 개요

### 비즈니스 문제

도시의 도로 표면 온도 상승과 미세먼지 문제를 해결하기 위한 **지능형 도로 살수 시스템**이 필요했습니다.

**요구사항:**
- PLC 장비를 통한 실시간 살수 제어
- 기상 데이터 기반 자동 살수 판단
- 다중 사이트 관리 (10+ 지역)
- 실시간 모니터링 및 알림
- 99.9% 가용성 보장

### 기술적 챌린지

1. **실시간성**: PLC 장비와 5초 간격 데이터 동기화
2. **확장성**: 다중 사이트 동시 제어 및 모니터링
3. **안정성**: 네트워크 불안정 환경에서도 안정적 운영
4. **보안**: 산업용 IoT 장비 접근 제어

---

## 🏗️ 시스템 아키텍처

### 전체 시스템 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                             │
│                  (Web Dashboard / Mobile App)                    │
└────────────────────────┬────────────────────────────────────────┘
                         │ HTTPS/WSS
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway (Nginx)                         │
│                  - Load Balancing (Round Robin)                  │
│                  - SSL/TLS Termination                           │
│                  - Rate Limiting                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Backend #1  │  │  Backend #2  │  │  Backend #N  │
│              │  │              │  │              │
│  Bun.js      │  │  Bun.js      │  │  Bun.js      │
│  ElysiaJS    │  │  ElysiaJS    │  │  ElysiaJS    │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └────────┬────────┴────────┬────────┘
                │                 │
    ┌───────────┴──┐         ┌────┴─────────┐
    │              │         │              │
    ▼              ▼         ▼              ▼
┌────────┐   ┌─────────┐  ┌──────┐    ┌─────────┐
│ MySQL  │   │ MongoDB │  │Redis │    │  Kafka  │
│        │   │         │  │      │    │ Cluster │
│ Master │   │ Replica │  │Cache │    └────┬────┘
│   │    │   │   Set   │  │      │         │
│ Slave  │   │         │  │      │         │
└────────┘   └─────────┘  └──────┘         │
                                            │
                                            ▼
                                    ┌───────────────┐
                                    │ PLC Adapter   │
                                    │ (Modbus TCP)  │
                                    └───────┬───────┘
                                            │
                        ┌───────────────────┼───────────────────┐
                        │                   │                   │
                        ▼                   ▼                   ▼
                  ┌─────────┐         ┌─────────┐         ┌─────────┐
                  │ PLC #1  │         │ PLC #2  │   ...   │ PLC #N  │
                  │ Site A  │         │ Site B  │         │ Site N  │
                  └─────────┘         └─────────┘         └─────────┘
```

### 아키텍처 특징

#### 1. 계층화된 구조 (Layered Architecture)

```
┌─────────────────────────────────────┐
│      Presentation Layer             │  ← API Endpoints
├─────────────────────────────────────┤
│      Business Logic Layer           │  ← Controllers & Services
├─────────────────────────────────────┤
│      Data Access Layer              │  ← Repositories & ORM
├─────────────────────────────────────┤
│      Infrastructure Layer           │  ← DB, Cache, Message Queue
└─────────────────────────────────────┘
```

**설계 이유:**
- 각 계층의 독립적 변경 가능
- 단위 테스트 용이성
- 명확한 책임 분리

#### 2. 마이크로서비스 지향 아키텍처

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Auth       │     │  Cooling     │     │    Admin     │
│   Service    │────▶│    Road      │────▶│   Service    │
│              │     │   Service    │     │              │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
                    ┌───────▼────────┐
                    │  Kafka Message │
                    │      Bus       │
                    └────────────────┘
```

**독립적인 모듈:**
- **Auth Module**: 인증/인가 처리
- **Cooling Road Module**: 살수 제어 로직
- **WebSocket Module**: 실시간 통신
- **PLC Module**: 장비 통신 추상화
- **AI Module**: 자동 살수 의사결정

---

## 🎨 핵심 설계 패턴

### 1. Adapter Pattern - PLC 통신 추상화

**문제:** 
- 개발 환경에 실제 PLC 장비가 없어 테스트 불가
- 다양한 PLC 제조사별 프로토콜 차이
- 프로덕션/개발 환경 분리 필요

**해결책:**

```typescript
// 추상화 인터페이스
interface IPLCReader {
    readCoils(address: number, count: number): Promise<boolean[]>
    readHoldingRegisters(address: number, count: number): Promise<number[]>
}

interface IPLCWriter {
    writeCoils(address: number, data: boolean[]): Promise<void>
    writeHoldingRegisters(address: number, data: number[]): Promise<void>
}

// 실제 PLC 구현
class ModbusPLCAdapter implements IPLCReader, IPLCWriter {
    constructor(private connection: ModbusConnection) {}
    
    async readCoils(address: number, count: number): Promise<boolean[]> {
        // Modbus TCP 프로토콜로 실제 PLC 통신
        const result = await this.connection.readCoils(address, count)
        return result.data
    }
    
    async writeCoils(address: number, data: boolean[]): Promise<void> {
        await this.connection.writeCoils(address, data)
    }
}

// 테스트용 가짜 PLC
class FakePLCAdapter implements IPLCReader, IPLCWriter {
    private simulatedData: Map<number, boolean[]> = new Map()
    
    async readCoils(address: number, count: number): Promise<boolean[]> {
        // 시뮬레이션된 데이터 반환
        return Array.from({ length: count }, () => Math.random() > 0.5)
    }
    
    async writeCoils(address: number, data: boolean[]): Promise<void> {
        // 메모리에만 저장
        this.simulatedData.set(address, data)
    }
}

// 팩토리 패턴으로 어댑터 선택
class PLCAdapterFactory {
    static create(config: PLCConfig): IPLCReader & IPLCWriter {
        if (config.mode === 'PRODUCTION') {
            return new ModbusPLCAdapter(
                new ModbusConnection(config.host, config.port)
            )
        } else {
            return new FakePLCAdapter()
        }
    }
}
```

**결과:**
- 환경 변수 하나로 실제/가짜 PLC 전환
- PLC 없이도 전체 시스템 개발/테스트 가능
- 새로운 PLC 제조사 추가 시 새 어댑터만 구현

### 2. Repository Pattern - 데이터 접근 추상화

**문제:**
- ORM 의존성으로 인한 테스트 어려움
- 비즈니스 로직에 SQL 쿼리 혼재
- 데이터베이스 변경 시 전체 코드 수정 필요

**해결책:**

```typescript
// 레포지토리 인터페이스 (일반화된 엔티티 예시)
interface IEntityRepository<T> {
    findById(id: number): Promise<T | null>
    findByField(field: string, value: any): Promise<T | null>
    create(data: CreateDTO): Promise<T>
    update(id: number, data: UpdateDTO): Promise<T>
    delete(id: number): Promise<void>
}

// Drizzle ORM 구현체
class DrizzleEntityRepository implements IEntityRepository<Entity> {
    constructor(private db: DrizzleDB) {}
    
    async findById(id: number): Promise<Entity | null> {
        const result = await this.db
            .select()
            .from(entities)
            .where(eq(entities.id, id))
            .limit(1)
        
        return result[0] || null
    }
    
    async findByField(field: string, value: any): Promise<Entity | null> {
        const result = await this.db
            .select()
            .from(entities)
            .where(eq(entities[field], value))
            .limit(1)
        
        return result[0] || null
    }
    
    // ... 기타 메서드
}

// 서비스 계층에서 사용
class BusinessService {
    constructor(private entityRepo: IEntityRepository) {}
    
    async authenticate(identifier: string, credential: string) {
        // 레포지토리를 통한 데이터 접근 (ORM 숨김)
        const entity = await this.entityRepo.findByField('identifier', identifier)
        
        if (!entity) {
            throw new UnauthorizedException()
        }
        
        const isValid = await this.verifyCredential(credential, entity.credential)
        
        if (!isValid) {
            throw new UnauthorizedException()
        }
        
        return this.generateToken(entity)
    }
}
```

**결과:**
- 비즈니스 로직과 데이터 접근 계층 분리
- Mock 레포지토리로 단위 테스트 가능
- ORM 교체 시 레포지토리만 수정

### 3. Dependency Injection - 느슨한 결합

**문제:**
- 클래스 간 강한 결합으로 테스트 어려움
- 의존성 관리의 복잡성
- 싱글톤 패턴의 한계

**해결책:**

```typescript
import { container, injectable, inject } from 'tsyringe'

// 서비스 등록
@injectable()
class MySQLService {
    private connection: Connection
    
    async connect() {
        this.connection = await createConnection(config)
    }
    
    getConnection() {
        return this.connection
    }
}

@injectable()
class RedisService {
    private client: RedisClient
    
    async connect() {
        this.client = await createClient(config)
    }
    
    async get(key: string): Promise<string | null> {
        return await this.client.get(key)
    }
}

@injectable()
class KafkaService {
    private producer: Producer
    
    async connect() {
        this.producer = kafka.producer()
        await this.producer.connect()
    }
    
    async send(topic: string, message: any) {
        await this.producer.send({
            topic,
            messages: [{ value: JSON.stringify(message) }]
        })
    }
}

// 의존성 주입
@injectable()
class BusinessController {
    constructor(
        @inject('MySQLService') private db: MySQLService,
        @inject('RedisService') private cache: RedisService,
        @inject('KafkaService') private messageQueue: KafkaService
    ) {}
    
    async executeOperation(resourceId: number) {
        // 1. 캐시 확인
        const cached = await this.cache.get(`resource:${resourceId}`)
        if (cached) {
            return JSON.parse(cached)
        }
        
        // 2. DB 조회
        const resource = await this.db
            .getConnection()
            .query('SELECT * FROM resources WHERE id = ?', [resourceId])
        
        // 3. Kafka로 이벤트 전송
        await this.messageQueue.send('resource.operation', {
            resourceId,
            command: 'EXECUTE_OPERATION'
        })
        
        return resource
    }
}

// 컨테이너 설정
container.register('MySQLService', { useClass: MySQLService })
container.register('RedisService', { useClass: RedisService })
container.register('KafkaService', { useClass: KafkaService })

// 의존성 자동 주입
const controller = container.resolve(BusinessController)
```

**결과:**
- 테스트 시 Mock 객체 주입 가능
- 서비스 교체 용이 (예: Redis → Memcached)
- 순환 참조 방지

### 4. Event-Driven Architecture - Kafka 메시지 큐

**문제:**
- 서비스 간 직접 통신으로 인한 강한 결합
- 동기 통신으로 인한 성능 저하
- 장애 전파 (한 서비스 장애가 전체 시스템 영향)

**해결책:**

```typescript
// 이벤트 타입 정의
enum EventType {
    OPERATION_STARTED = 'operation.started',
    OPERATION_STOPPED = 'operation.stopped',
    DATA_UPDATED = 'data.updated',
    EXTERNAL_DATA_RECEIVED = 'external.data.received'
}

// 이벤트 발행자 (Producer)
class EventPublisher {
    constructor(private kafka: KafkaProducer) {}
    
    async publish(event: EventType, payload: any) {
        await this.kafka.send({
            topic: event,
            messages: [{
                key: payload.siteId?.toString(),
                value: JSON.stringify({
                    type: event,
                    payload,
                    timestamp: new Date()
                })
            }]
        })
    }
}

// 이벤트 구독자 (Consumer)
class EventSubscriber {
    constructor(private kafka: KafkaConsumer) {}
    
    async subscribe(
        event: EventType, 
        handler: (payload: any) => Promise<void>
    ) {
        await this.kafka.subscribe({ 
            topic: event,
            fromBeginning: false 
        })
        
        await this.kafka.run({
            eachMessage: async ({ message }) => {
                const event = JSON.parse(message.value.toString())
                await handler(event.payload)
            }
        })
    }
}

// 사용 예시: 작업 시작 이벤트 처리
class OperationService {
    constructor(
        private publisher: EventPublisher,
        private subscriber: EventSubscriber
    ) {
        this.setupEventHandlers()
    }
    
    private setupEventHandlers() {
        // 작업 시작 이벤트 구독
        this.subscriber.subscribe(
            EventType.OPERATION_STARTED, 
            async (payload) => {
                await this.logOperationHistory(payload)
                await this.captureBeforeSnapshot(payload.resourceId)
            }
        )
        
        // 작업 중지 이벤트 구독
        this.subscriber.subscribe(
            EventType.OPERATION_STOPPED,
            async (payload) => {
                await this.captureAfterSnapshot(payload.resourceId)
                await this.calculateMetrics(payload)
            }
        )
    }
    
    async startOperation(resourceId: number) {
        // 제어 명령 전송
        await this.sendControlCommand(resourceId, 'START')
        
        // 이벤트 발행 (비동기)
        await this.publisher.publish(
            EventType.OPERATION_STARTED,
            { resourceId, timestamp: new Date() }
        )
    }
}
```

**메시지 큐 토픽 설계:**

```
device.control           → 장비 제어 명령
device.data.updated      → 장비 데이터 업데이트
operation.started        → 작업 시작
operation.stopped        → 작업 중지
external.data.received   → 외부 데이터 수신
websocket.broadcast      → WebSocket 브로드캐스트
ai.decision              → AI 판단 결과
```

**결과:**
- 서비스 간 느슨한 결합
- 비동기 처리로 응답 속도 향상
- 이벤트 재처리 가능 (장애 복구)
- 새로운 구독자 추가 용이

### 5. Semaphore Pattern - 동시성 제어

**문제:**
- 다중 CCTV에서 동시 이미지 캡처 시 CPU/메모리 과부하
- FFmpeg 프로세스 과다 생성
- 파일 I/O 경합

**해결책:**

```typescript
class Semaphore {
    private permits: number
    private queue: Array<() => void> = []
    
    constructor(permits: number) {
        this.permits = permits
    }
    
    async acquire<T>(task: () => Promise<T>): Promise<T> {
        // 허가 대기
        await this.waitForPermit()
        
        try {
            // 작업 실행
            return await task()
        } finally {
            // 허가 반환
            this.release()
        }
    }
    
    private async waitForPermit(): Promise<void> {
        if (this.permits > 0) {
            this.permits--
            return
        }
        
        // 대기 큐에 추가
        return new Promise(resolve => {
            this.queue.push(resolve)
        })
    }
    
    private release(): void {
        const next = this.queue.shift()
        
        if (next) {
            next()
        } else {
            this.permits++
        }
    }
}

// 사용 예시
class ImageCaptureService {
    // 최대 3개의 동시 캡처만 허용
    private captureSemaphore = new Semaphore(3)
    
    async captureFromMultipleSites(siteIds: number[]) {
        // 모든 사이트의 이미지를 병렬로 캡처하되,
        // 동시에 3개까지만 실행
        const promises = siteIds.map(siteId =>
            this.captureSemaphore.acquire(async () => {
                return await this.captureImage(siteId)
            })
        )
        
        return await Promise.all(promises)
    }
    
    private async captureImage(siteId: number): Promise<Buffer> {
        // FFmpeg로 RTSP 스트림 캡처
        const rtspUrl = await this.getRTSPUrl(siteId)
        
        return new Promise((resolve, reject) => {
            const ffmpeg = spawn('ffmpeg', [
                '-i', rtspUrl,
                '-frames:v', '1',
                '-f', 'image2pipe',
                '-'
            ])
            
            const chunks: Buffer[] = []
            
            ffmpeg.stdout.on('data', chunk => chunks.push(chunk))
            ffmpeg.on('close', () => resolve(Buffer.concat(chunks)))
            ffmpeg.on('error', reject)
        })
    }
}
```

**결과:**
- CPU 사용률 70% → 30% 감소
- 메모리 안정화 (OOM 에러 제거)
- 응답 시간 예측 가능

---

## 💡 기술적 의사결정

### 1. Bun.js를 선택한 이유

**비교 분석:**

| 항목 | Node.js | Deno | Bun.js |
|------|---------|------|--------|
| **시작 시간** | 100ms | 80ms | **30ms** |
| **번들 크기** | 큰 편 | 중간 | **작음** |
| **타입스크립트** | 별도 빌드 | 네이티브 | **네이티브** |
| **패키지 속도** | npm: 느림 | 중간 | **3-5배 빠름** |
| **생태계** | 매우 풍부 | 제한적 | **npm 호환** |

**선택 이유:**
```typescript
// Node.js: 별도 빌드 필요
// 1. tsconfig.json 설정
// 2. tsc 또는 ts-node 사용
// 3. node dist/index.js 실행

// Bun.js: 즉시 실행
bun run src/index.ts  // 빌드 불필요!
```

**실측 성능:**
- 콜드 스타트: Node.js 1.2초 → Bun.js 0.4초
- API 응답 시간: 평균 20% 향상
- 메모리 사용: 약 30% 감소

### 2. ElysiaJS를 선택한 이유

**Express vs ElysiaJS 비교:**

```typescript
// Express (Node.js)
app.get('/api/sites/:id', async (req, res) => {
    try {
        const id = parseInt(req.params.id)
        const site = await db.query('SELECT * FROM sites WHERE id = ?', [id])
        res.json({ success: true, data: site })
    } catch (error) {
        res.status(500).json({ error: error.message })
    }
})

// ElysiaJS (Bun.js)
app.get('/api/sites/:id', async ({ params }) => {
    const id = parseInt(params.id)
    const site = await db.query('SELECT * FROM sites WHERE id = ?', [id])
    return { success: true, data: site }
}, {
    params: t.Object({
        id: t.String()
    })
})
```

**장점:**
- **타입 안전성**: TypeBox 기반 런타임 검증
- **자동 문서화**: OpenAPI 스펙 자동 생성
- **성능**: Express 대비 10배 빠른 라우팅
- **간결함**: 보일러플레이트 코드 최소화

### 3. Drizzle ORM을 선택한 이유

**Prisma vs TypeORM vs Drizzle 비교:**

```typescript
// Prisma: 스키마 별도 파일
// schema.prisma
model User {
  id    Int    @id @default(autoincrement())
  email String @unique
}

// TypeORM: 데코레이터 기반
@Entity()
class User {
  @PrimaryGeneratedColumn()
  id: number
  
  @Column({ unique: true })
  email: string
}

// Drizzle: SQL-like TypeScript
const users = mysqlTable('users', {
  id: int('id').primaryKey().autoincrement(),
  email: varchar('email', { length: 255 }).unique()
})

// 타입 자동 추론
type User = typeof users.$inferSelect  // { id: number, email: string }
```

**선택 이유:**
- **경량**: Prisma 대비 10배 작은 번들 사이즈
- **SQL 친화적**: 복잡한 쿼리 작성 용이
- **타입 안전성**: 컴파일 타임 검증
- **마이그레이션**: Git-friendly SQL 파일

### 4. MySQL + MongoDB + Redis 조합

**데이터 저장소 선택 전략:**

```typescript
// MySQL: 트랜잭션이 중요한 데이터
// - 사용자/엔티티 정보
// - 리소스 정보
// - 작업 이력 (정규화된 데이터)

const resourceRepository = {
    async createResource(data: ResourceData) {
        return await db.transaction(async (tx) => {
            const resource = await tx.insert(resources).values(data)
            await tx.insert(resourceSettings).values({
                resourceId: resource.id,
                ...defaultSettings
            })
            return resource
        })
    }
}

// MongoDB: 비정형 로그 데이터
// - 시스템 로그
// - 에러 로그
// - 이벤트 히스토리

const logger = {
    async log(level: string, message: string, metadata: any) {
        await mongoDb.collection('logs').insertOne({
            level,
            message,
            metadata,
            timestamp: new Date(),
            hostname: os.hostname()
        })
    }
}

// Redis: 캐싱 및 세션
// - 사용자 세션
// - API 응답 캐시
// - Rate Limiting 카운터

const cache = {
    async getResourceInfo(resourceId: number) {
        const key = `resource:${resourceId}`
        const cached = await redis.get(key)
        
        if (cached) {
            return JSON.parse(cached)
        }
        
        const resource = await db.query('SELECT * FROM resources WHERE id = ?', [resourceId])
        await redis.setex(key, 3600, JSON.stringify(resource))
        
        return resource
    }
}
```

**분산 데이터 관리:**
- MySQL: ACID 보장이 필요한 핵심 데이터
- MongoDB: 스키마 유연성이 필요한 로그
- Redis: 빠른 읽기가 필요한 캐시

---

## ⚡ 성능 최적화

### 1. 데이터베이스 쿼리 최적화

**N+1 문제 해결:**

```typescript
// ❌ N+1 문제 발생
async function getResourcesWithRelations() {
    const resources = await db.select().from(resources)  // 1 query
    
    for (const resource of resources) {
        // N queries (리소스 개수만큼)
        resource.relations = await db
            .select()
            .from(relations)
            .where(eq(relations.parentId, resource.parentId))
    }
    
    return resources
}

// ✅ JOIN으로 해결
async function getResourcesWithRelations() {
    return await db
        .select({
            resource: resources,
            relation: relations
        })
        .from(resources)
        .leftJoin(relations, eq(resources.parentId, relations.parentId))
        // 1 query로 모든 데이터 조회
}
```

**인덱스 전략:**

```typescript
// 복합 인덱스 설계
const operationHistory = mysqlTable('operation_history', {
    id: int('id').primaryKey(),
    resourceId: int('resource_id'),
    startTime: datetime('start_time'),
    endTime: datetime('end_time')
}, (table) => ({
    // 자주 함께 조회되는 컬럼에 복합 인덱스
    resourceTimeIdx: index('idx_resource_time')
        .on(table.resourceId, table.startTime)
}))

// 쿼리 최적화
const history = await db
    .select()
    .from(operationHistory)
    .where(
        and(
            eq(operationHistory.resourceId, resourceId),      // 인덱스 활용
            gte(operationHistory.startTime, startDate) // 인덱스 활용
        )
    )
    .orderBy(desc(operationHistory.startTime))
```

### 2. 캐싱 전략

**다층 캐싱 (Multi-level Caching):**

```typescript
class CacheManager {
    private memoryCache = new Map<string, CacheEntry>()
    
    async get<T>(key: string): Promise<T | null> {
        // L1: 메모리 캐시 (가장 빠름)
        const memCached = this.memoryCache.get(key)
        if (memCached && !this.isExpired(memCached)) {
            return memCached.value as T
        }
        
        // L2: Redis 캐시
        const redisCached = await this.redis.get(key)
        if (redisCached) {
            const value = JSON.parse(redisCached)
            // Redis에서 가져온 데이터를 메모리에도 캐시
            this.memoryCache.set(key, {
                value,
                expiry: Date.now() + 60000 // 1분
            })
            return value as T
        }
        
        // L3: 데이터베이스
        return null
    }
    
    async set<T>(key: string, value: T, ttl: number): Promise<void> {
        // 메모리와 Redis 둘 다 저장
        this.memoryCache.set(key, {
            value,
            expiry: Date.now() + Math.min(ttl, 60000)
        })
        
        await this.redis.setex(key, ttl, JSON.stringify(value))
    }
}
```

**Cache Invalidation:**

```typescript
// 이벤트 기반 캐시 무효화
class ResourceService {
    async updateResource(resourceId: number, data: UpdateResourceDTO) {
        await db.update(resources)
            .set(data)
            .where(eq(resources.id, resourceId))
        
        // 관련 캐시 즉시 삭제
        await cache.delete(`resource:${resourceId}`)
        await cache.delete(`resource:${resourceId}:settings`)
        await cache.delete(`parent:${data.parentId}:resources`)
        
        // Kafka로 캐시 무효화 이벤트 발행 (다른 서버들도 삭제)
        await kafka.send({
            topic: 'cache.invalidate',
            messages: [{
                value: JSON.stringify({
                    pattern: `resource:${resourceId}*`
                })
            }]
        })
    }
}
```

### 3. WebSocket 최적화

**Selective Broadcasting:**

```typescript
class WebSocketManager {
    private connections = new Map<string, WebSocket>()
    private subscriptions = new Map<string, Set<string>>()
    
    // 클라이언트가 특정 토픽 구독
    subscribe(connectionId: string, topic: string) {
        if (!this.subscriptions.has(topic)) {
            this.subscriptions.set(topic, new Set())
        }
        this.subscriptions.get(topic)!.add(connectionId)
    }
    
    // 토픽을 구독한 클라이언트에게만 전송
    broadcast(topic: string, message: any) {
        const subscribers = this.subscriptions.get(topic)
        if (!subscribers) return
        
        const payload = JSON.stringify(message)
        let sent = 0
        
        for (const connectionId of subscribers) {
            const ws = this.connections.get(connectionId)
            if (ws && ws.readyState === WebSocket.OPEN) {
                ws.send(payload)
                sent++
            }
        }
        
        console.log(`Broadcast to ${sent}/${subscribers.size} subscribers`)
    }
}

// 사용 예시
wsManager.subscribe('user123', 'resource:1:data')  // 리소스 1만 구독
wsManager.subscribe('user456', 'resource:2:data')  // 리소스 2만 구독

// 리소스 1 데이터 업데이트 → user123에게만 전송
wsManager.broadcast('resource:1:data', {
    metric1: 25.5,
    metric2: 60
})
```

### 4. 이미지 처리 최적화

```typescript
class ImageProcessor {
    // Sharp 라이브러리로 이미지 최적화
    async optimizeImage(buffer: Buffer): Promise<Buffer> {
        return await sharp(buffer)
            .resize(1920, 1080, {
                fit: 'inside',
                withoutEnlargement: true
            })
            .webp({
                quality: 80,
                effort: 4  // 압축 수준 (0-6)
            })
            .toBuffer()
    }
    
    // 썸네일 생성
    async createThumbnail(buffer: Buffer): Promise<Buffer> {
        return await sharp(buffer)
            .resize(320, 180)
            .webp({ quality: 60 })
            .toBuffer()
    }
    
    // 병렬 처리
    async processImages(buffers: Buffer[]) {
        return await Promise.all(
            buffers.map(buffer => 
                this.semaphore.acquire(() => 
                    this.optimizeImage(buffer)
                )
            )
        )
    }
}
```

**결과:**
- 원본 JPEG (2.5MB) → WebP (800KB): 68% 감소
- 처리 시간: 평균 300ms

---

## 🔐 보안 설계

### 1. 인증 시스템 (JWT + MFA)

**JWT 토큰 구조:**

```typescript
interface JWTPayload {
    userId: number
    email: string
    role: UserRole
    organizationId: number
    iat: number  // Issued At
    exp: number  // Expiration
}

class AuthService {
    generateToken(user: User): string {
        const payload: JWTPayload = {
            userId: user.id,
            email: user.email,
            role: user.role,
            organizationId: user.organizationId,
            iat: Math.floor(Date.now() / 1000),
            exp: Math.floor(Date.now() / 1000) + 86400  // 24시간
        }
        
        return jwt.sign(payload, JWT_SECRET, {
            algorithm: 'HS512'
        })
    }
    
    verifyToken(token: string): JWTPayload {
        try {
            return jwt.verify(token, JWT_SECRET) as JWTPayload
        } catch (error) {
            throw new UnauthorizedException('Invalid token')
        }
    }
}
```

**MFA (TOTP) 구현:**

```typescript
import * as OTPAuth from 'otpauth'

class MFAService {
    // MFA 등록: QR 코드 생성
    async setupMFA(userId: number): Promise<{
        secret: string
        qrCode: string
    }> {
        // 사용자별 시크릿 생성
        const secret = OTPAuth.Secret.generate()
        
        const totp = new OTPAuth.TOTP({
            issuer: 'SmartRoad',
            label: `user_${userId}`,
            algorithm: 'SHA1',
            digits: 6,
            period: 30,
            secret: secret
        })
        
        // QR 코드 생성
        const qrCode = await QRCode.toDataURL(totp.toString())
        
        // DB에 암호화하여 저장
        await this.saveMFASecret(userId, secret.base32)
        
        return {
            secret: secret.base32,
            qrCode
        }
    }
    
    // MFA 검증: 타이밍 공격 방지
    async verifyMFA(userId: number, token: string): Promise<boolean> {
        const secret = await this.getMFASecret(userId)
        if (!secret) return false
        
        const totp = new OTPAuth.TOTP({
            secret: OTPAuth.Secret.fromBase32(secret)
        })
        
        // 시간 창 허용 (±1 period = ±30초)
        const delta = totp.validate({
            token,
            window: 1
        })
        
        // 타이밍 공격 방지: 항상 일정 시간 소요
        await this.constantTimeDelay()
        
        return delta !== null
    }
    
    // 상수 시간 지연 (타이밍 공격 방지)
    private async constantTimeDelay(): Promise<void> {
        const start = Date.now()
        const targetDuration = 100  // 100ms
        
        // 실제 검증 로직 실행 후 남은 시간만큼 대기
        const elapsed = Date.now() - start
        const remaining = Math.max(0, targetDuration - elapsed)
        
        await new Promise(resolve => setTimeout(resolve, remaining))
    }
}
```

### 2. Rate Limiting

```typescript
class RateLimiter {
    constructor(
        private redis: RedisClient,
        private windowMs: number = 60000,      // 1분
        private maxRequests: number = 100      // 최대 100 요청
    ) {}
    
    async checkLimit(key: string): Promise<{
        allowed: boolean
        remaining: number
        resetAt: Date
    }> {
        const now = Date.now()
        const windowKey = `ratelimit:${key}:${Math.floor(now / this.windowMs)}`
        
        // Redis에서 현재 윈도우의 요청 수 조회
        const current = await this.redis.incr(windowKey)
        
        // 첫 요청이면 TTL 설정
        if (current === 1) {
            await this.redis.expire(windowKey, Math.ceil(this.windowMs / 1000))
        }
        
        const allowed = current <= this.maxRequests
        const remaining = Math.max(0, this.maxRequests - current)
        const resetAt = new Date(
            Math.floor(now / this.windowMs + 1) * this.windowMs
        )
        
        return { allowed, remaining, resetAt }
    }
}

// ElysiaJS 미들웨어
app.use(async ({ request, set }) => {
    const ip = request.headers.get('x-forwarded-for') || 'unknown'
    const result = await rateLimiter.checkLimit(ip)
    
    // 응답 헤더에 Rate Limit 정보 추가
    set.headers['X-RateLimit-Limit'] = '100'
    set.headers['X-RateLimit-Remaining'] = result.remaining.toString()
    set.headers['X-RateLimit-Reset'] = result.resetAt.toISOString()
    
    if (!result.allowed) {
        set.status = 429
        return { error: 'Too many requests' }
    }
})
```

### 3. 역할 기반 접근 제어 (RBAC)

```typescript
enum UserRole {
    USER = 'USER',                    // 일반 사용자
    MAINTENANCE = 'MAINTENANCE',      // 유지보수 담당자
    DEVELOPER = 'DEVELOPER',          // 개발자
    ORGANIZE = 'ORGANIZE'             // 조직 관리자
}

enum Permission {
    READ_SITE = 'site:read',
    WRITE_SITE = 'site:write',
    CONTROL_PLC = 'plc:control',
    MANAGE_USERS = 'users:manage',
    VIEW_LOGS = 'logs:view'
}

const RolePermissions: Record<UserRole, Permission[]> = {
    [UserRole.USER]: [
        Permission.READ_SITE
    ],
    [UserRole.MAINTENANCE]: [
        Permission.READ_SITE,
        Permission.WRITE_SITE,
        Permission.CONTROL_PLC,
        Permission.VIEW_LOGS
    ],
    [UserRole.DEVELOPER]: [
        Permission.READ_SITE,
        Permission.WRITE_SITE,
        Permission.CONTROL_PLC,
        Permission.VIEW_LOGS
    ],
    [UserRole.ORGANIZE]: [
        Permission.READ_SITE,
        Permission.WRITE_SITE,
        Permission.CONTROL_PLC,
        Permission.MANAGE_USERS,
        Permission.VIEW_LOGS
    ]
}

class AuthGuard {
    checkPermission(user: User, required: Permission): boolean {
        const permissions = RolePermissions[user.role]
        return permissions.includes(required)
    }
}

// 데코레이터로 권한 검사
function RequirePermission(permission: Permission) {
    return function (
        target: any,
        propertyKey: string,
        descriptor: PropertyDescriptor
    ) {
        const originalMethod = descriptor.value
        
        descriptor.value = async function (...args: any[]) {
            const user = args[0]  // 첫 번째 인자가 user
            
            if (!authGuard.checkPermission(user, permission)) {
                throw new ForbiddenException()
            }
            
            return originalMethod.apply(this, args)
        }
    }
}

// 사용 예시
class CoolingRoadController {
    @RequirePermission(Permission.CONTROL_PLC)
    async startWatering(user: User, siteId: number) {
        // PLC 제어 로직
    }
    
    @RequirePermission(Permission.MANAGE_USERS)
    async createUser(user: User, userData: CreateUserDTO) {
        // 사용자 생성 로직
    }
}
```

### 4. SQL Injection 방지

```typescript
// ❌ 취약한 코드
async function getSiteByName(name: string) {
    const query = `SELECT * FROM sites WHERE name = '${name}'`
    return await db.execute(query)
}
// 공격 예시: name = "' OR '1'='1"
// 결과 쿼리: SELECT * FROM sites WHERE name = '' OR '1'='1'

// ✅ Prepared Statement 사용
async function getSiteByName(name: string) {
    return await db
        .select()
        .from(sites)
        .where(eq(sites.name, name))
    // Drizzle ORM이 자동으로 파라미터 바인딩 처리
}

// ✅ 직접 쿼리 시 Prepared Statement
async function rawQuery(name: string) {
    return await db.execute(
        sql`SELECT * FROM sites WHERE name = ${name}`
    )
}
```

---

## 📊 모니터링 및 로깅

### Structured Logging

```typescript
enum LogLevel {
    DEBUG = 'debug',
    INFO = 'info',
    WARN = 'warn',
    ERROR = 'error'
}

interface LogEntry {
    level: LogLevel
    message: string
    timestamp: Date
    service: string
    userId?: number
    requestId?: string
    metadata?: Record<string, any>
    error?: {
        message: string
        stack: string
        code?: string
    }
}

class Logger {
    async log(entry: Omit<LogEntry, 'timestamp' | 'service'>) {
        const logEntry: LogEntry = {
            ...entry,
            timestamp: new Date(),
            service: 'backend'
        }
        
        // 콘솔 출력 (개발 환경)
        if (IS_DEVELOPMENT) {
            console.log(JSON.stringify(logEntry, null, 2))
        }
        
        // MongoDB에 비동기로 저장 (프로덕션)
        if (IS_PRODUCTION) {
            await logBuffer.push(logEntry)
        }
    }
    
    info(message: string, metadata?: any) {
        return this.log({ level: LogLevel.INFO, message, metadata })
    }
    
    error(message: string, error: Error, metadata?: any) {
        return this.log({
            level: LogLevel.ERROR,
            message,
            metadata,
            error: {
                message: error.message,
                stack: error.stack || '',
                code: (error as any).code
            }
        })
    }
}

// 사용 예시
logger.info('Watering started', {
    siteId: 1,
    userId: 123,
    duration: 300
})

logger.error('PLC connection failed', new Error('Timeout'), {
    plcHost: '192.168.1.100',
    attemptCount: 3
})
```

---

**이 문서는 실무 프로젝트의 아키텍처와 설계 패턴을 포트폴리오용으로 정리한 것입니다.**
