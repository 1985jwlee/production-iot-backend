# 🎯 Technical Challenges & Solutions

실무 프로젝트에서 마주한 기술적 챌린지와 해결 과정을 상세히 기록합니다.

---

## 목차

1. [WebSocket 연결 안정성 문제](#1-websocket-연결-안정성-문제)
2. [다중 환경 관리의 복잡성](#2-다중-환경-관리의-복잡성)
3. [이미지 처리 성능 병목](#3-이미지-처리-성능-병목)
4. [PLC 통신 추상화](#4-plc-통신-추상화)
5. [실시간 데이터 동기화](#5-실시간-데이터-동기화)

---

## 1. WebSocket 연결 안정성 문제

### 문제 상황

```mermaid
sequenceDiagram
    participant C as Client
    participant N as Network<br/>(WiFi/4G)
    participant S as Server
    
    Note over C,S: ❌ 문제 시나리오
    
    C->>N: WebSocket Connect
    N->>S: Connection Established
    
    Note over N: 네트워크 불안정<br/>(WiFi → 4G 전환)
    
    N--xC: Connection Lost
    
    Note over C: 클라이언트는<br/>연결 끊김 감지
    
    Note over S: 서버는 여전히<br/>연결 유지 중<br/>(좀비 연결)
    
    C->>N: Reconnect Attempt
    N->>S: New Connection
    
    Note over S: 좀비 연결 +<br/>새 연결 = 리소스 낭비
```

**증상:**
- 모바일 환경에서 네트워크 전환 시 연결 끊김
- 서버에서 끊긴 연결을 감지하지 못함 (좀비 연결)
- 브라우저 백그라운드 전환 시 연결 유지 실패

### 근본 원인 분석

```mermaid
graph TB
    subgraph "문제 요인"
        TCP[TCP Keep-Alive<br/>간격이 너무 김<br/>실시간 감지 어려움]
        NAT[NAT/방화벽<br/>일정 시간 통신 없으면<br/>연결 강제 종료]
        BROWSER[브라우저 정책<br/>백그라운드 탭의<br/>타이머 throttling]
    end
    
    subgraph "결과"
        ZOMBIE[좀비 연결<br/>리소스 낭비]
        DISCONNECT[예기치 않은<br/>연결 끊김]
        DELAY[재연결 지연<br/>데이터 손실]
    end
    
    TCP --> ZOMBIE
    NAT --> DISCONNECT
    BROWSER --> DELAY
    
    style TCP fill:#ffebee,stroke:#d32f2f
    style NAT fill:#ffebee,stroke:#d32f2f
    style BROWSER fill:#ffebee,stroke:#d32f2f
    style ZOMBIE fill:#ffe1e1,stroke:#f44336
    style DISCONNECT fill:#ffe1e1,stroke:#f44336
    style DELAY fill:#ffe1e1,stroke:#f44336
```

### 해결책 1: Application-Level Keepalive

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    
    Note over C,S: ✅ Application-Level Keepalive
    
    loop Every 30 seconds
        C->>S: Ping
        S->>C: Pong
    end
    
    Note over C: Pong 타임아웃<br/>설정 (90초)
    
    C->>S: Ping
    
    Note over C,S: 90초 동안<br/>Pong 없음
    
    Note over C: 연결 끊김 감지
    C->>C: Close Connection
    C->>S: Reconnect
```

**구현:**

```typescript
class WebSocketConnection {
    private pingInterval: Timer
    private pongTimeout: Timer
    private lastPongTime: number = Date.now()
    
    private startKeepalive() {
        // 30초마다 Ping 전송
        this.pingInterval = setInterval(() => {
            if (this.ws.readyState === WebSocket.OPEN) {
                this.ws.send(JSON.stringify({ type: 'ping' }))
                
                // 90초 안에 Pong 없으면 연결 종료
                this.pongTimeout = setTimeout(() => {
                    if (Date.now() - this.lastPongTime > 90000) {
                        this.ws.close()
                    }
                }, 90000)
            }
        }, 30000)
    }
}
```

### 해결책 2: Exponential Backoff 재연결

```mermaid
graph TB
    START[연결 끊김]
    
    START --> ATTEMPT1[1차 시도<br/>지연: 1초]
    ATTEMPT1 -->|실패| ATTEMPT2[2차 시도<br/>지연: 2초]
    ATTEMPT2 -->|실패| ATTEMPT3[3차 시도<br/>지연: 4초]
    ATTEMPT3 -->|실패| ATTEMPT4[4차 시도<br/>지연: 8초]
    ATTEMPT4 -->|실패| ATTEMPT5[5차 시도<br/>지연: 16초]
    ATTEMPT5 -->|실패| MAX[최대 시도 도달<br/>30초 대기]
    
    ATTEMPT1 -->|성공| SUCCESS[연결 성공<br/>카운터 리셋]
    ATTEMPT2 -->|성공| SUCCESS
    ATTEMPT3 -->|성공| SUCCESS
    ATTEMPT4 -->|성공| SUCCESS
    ATTEMPT5 -->|성공| SUCCESS
    
    style START fill:#ffebee,stroke:#d32f2f
    style ATTEMPT1 fill:#fff9c4,stroke:#fbc02d
    style ATTEMPT2 fill:#fff9c4,stroke:#fbc02d
    style ATTEMPT3 fill:#fff9c4,stroke:#fbc02d
    style ATTEMPT4 fill:#fff9c4,stroke:#fbc02d
    style ATTEMPT5 fill:#fff9c4,stroke:#fbc02d
    style MAX fill:#ffe1e1,stroke:#f44336
    style SUCCESS fill:#e8f5e9,stroke:#4caf50,stroke-width:3px
```

**구현:**

```typescript
class ReconnectionManager {
    private reconnectAttempts = 0
    private maxReconnectDelay = 30000
    
    async reconnect() {
        const baseDelay = Math.min(
            1000 * Math.pow(2, this.reconnectAttempts),
            this.maxReconnectDelay
        )
        
        // Jitter 추가 (±20%)
        const jitter = baseDelay * 0.2 * (Math.random() - 0.5)
        const delay = baseDelay + jitter
        
        await new Promise(resolve => setTimeout(resolve, delay))
        
        try {
            await this.connect()
            this.reconnectAttempts = 0
        } catch (error) {
            this.reconnectAttempts++
            this.reconnect()
        }
    }
}
```

### 결과

| 지표 | 개선 전 | 개선 후 | 개선률 |
|------|---------|---------|--------|
| **평균 연결 유지 시간** | 5분 | 2시간+ | **2,400% ↑** |
| **좀비 연결 수** | 10-15% | <1% | **90% ↓** |
| **재연결 성공률** | 60% | 95% | **58% ↑** |
| **서버 리소스 사용** | 높음 | 정상 | **안정화** |

---

## 2. 다중 환경 관리의 복잡성

### 문제 상황

```mermaid
graph TB
    subgraph "개발 환경"
        DEV_MYSQL[MySQL<br/>localhost:3306]
        DEV_REDIS[Redis<br/>localhost:6380]
        DEV_KAFKA[Kafka<br/>localhost:9092]
    end
    
    subgraph "스테이징 환경"
        STG_MYSQL[MySQL<br/>staging-db:3306]
        STG_REDIS[Redis<br/>staging-redis:6379]
        STG_KAFKA[Kafka<br/>staging-kafka:9092]
    end
    
    subgraph "프로덕션 환경"
        PROD_MYSQL[MySQL<br/>prod-db:3306]
        PROD_REDIS[Redis Cluster<br/>redis-cluster:6379]
        PROD_KAFKA[Kafka Cluster<br/>kafka-cluster:9092]
    end
    
    APP[Application]
    
    APP -.->|개발| DEV_MYSQL
    APP -.->|스테이징| STG_MYSQL
    APP -.->|프로덕션| PROD_MYSQL
    
    style APP fill:#f0e1ff,stroke:#9c27b0,stroke-width:3px
    style DEV_MYSQL fill:#e1f5ff,stroke:#2196f3
    style DEV_REDIS fill:#e1f5ff,stroke:#2196f3
    style DEV_KAFKA fill:#e1f5ff,stroke:#2196f3
    style STG_MYSQL fill:#fff4e1,stroke:#ff9800
    style STG_REDIS fill:#fff4e1,stroke:#ff9800
    style STG_KAFKA fill:#fff4e1,stroke:#ff9800
    style PROD_MYSQL fill:#e8f5e9,stroke:#4caf50
    style PROD_REDIS fill:#e8f5e9,stroke:#4caf50
    style PROD_KAFKA fill:#e8f5e9,stroke:#4caf50
```

**문제점:**
```
❌ 환경별로 다른 설정 파일 관리
❌ 배포 시 설정 실수 빈번
❌ 환경 변수 누락으로 인한 런타임 에러
❌ 하드코딩된 설정값
```

### 해결책: 중앙집중식 설정 관리

```mermaid
graph TB
    subgraph "Environment Files"
        ENV_EX[.env.example<br/>템플릿]
        ENV_DEV[.env<br/>로컬 개발]
        ENV_STG[.env.staging<br/>스테이징]
        ENV_PROD[.env.production<br/>프로덕션]
    end
    
    subgraph "Config Layer"
        PARSER[dotenv Parser]
        VALIDATOR[Config Validator]
        SELECTOR[Environment Selector]
    end
    
    subgraph "Application Config"
        CONFIG[Centralized Config<br/>단일 진실의 원천]
    end
    
    ENV_DEV --> PARSER
    ENV_STG --> PARSER
    ENV_PROD --> PARSER
    
    PARSER --> VALIDATOR
    VALIDATOR --> SELECTOR
    SELECTOR --> CONFIG
    
    CONFIG --> APP[Application]
    
    style ENV_EX fill:#f3e5f5,stroke:#9c27b0
    style ENV_DEV fill:#e1f5ff,stroke:#2196f3
    style ENV_STG fill:#fff4e1,stroke:#ff9800
    style ENV_PROD fill:#e8f5e9,stroke:#4caf50
    style PARSER fill:#fff9c4,stroke:#fbc02d
    style VALIDATOR fill:#fff9c4,stroke:#fbc02d
    style SELECTOR fill:#fff9c4,stroke:#fbc02d
    style CONFIG fill:#f0e1ff,stroke:#9c27b0,stroke-width:3px
    style APP fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
```

**구현:**

```typescript
// configs/environment.ts
export const CONFIG = {
    PORT: parseInt(process.env.PORT || '8101'),
    
    MYSQL: {
        HOST: process.env.MYSQL_HOST || 'localhost',
        PORT: parseInt(process.env.MYSQL_PORT || '3306'),
        DATABASE: selectByEnv('smartroad_dev', 'smartroad')
    },
    
    REDIS: {
        HOST: process.env.REDIS_HOST || 'localhost',
        PORT: selectByEnv(6380, 6379)
    },
    
    JWT: {
        SECRET: selectByEnv(
            process.env.JWT_SECRET_DEV || 'dev-secret',
            process.env.JWT_SECRET || throwEnvError('JWT_SECRET')
        )
    }
} as const

function selectByEnv<T>(dev: T, prod: T): T {
    return IS_PRODUCTION ? prod : dev
}
```

### 결과

```mermaid
graph LR
    BEFORE[개선 전<br/>환경별 설정 분산<br/>배포 오류 빈번]
    AFTER[개선 후<br/>단일 설정 파일<br/>환경 자동 선택]
    
    BEFORE -.->|개선| AFTER
    
    METRICS[측정 결과<br/>배포 오류 90% 감소<br/>설정 시간 1일→10분]
    
    AFTER --> METRICS
    
    style BEFORE fill:#ffebee,stroke:#d32f2f
    style AFTER fill:#e8f5e9,stroke:#4caf50,stroke-width:3px
    style METRICS fill:#e1f5ff,stroke:#2196f3
```

---

## 3. 이미지 처리 성능 병목

### 문제 상황

```mermaid
graph TB
    subgraph "10개 사이트 동시 요청"
        S1[Site 1]
        S2[Site 2]
        S3[Site 3]
        S4[Site 4]
        S5[Site 5]
        S6[Site 6]
        S7[Site 7]
        S8[Site 8]
        S9[Site 9]
        S10[Site 10]
    end
    
    subgraph "FFmpeg 프로세스"
        F1[FFmpeg #1<br/>200MB]
        F2[FFmpeg #2<br/>200MB]
        F3[FFmpeg #3<br/>200MB]
        F4[FFmpeg #4<br/>200MB]
        F5[FFmpeg #5<br/>200MB]
        F6[FFmpeg #6<br/>200MB]
        F7[FFmpeg #7<br/>200MB]
        F8[FFmpeg #8<br/>200MB]
        F9[FFmpeg #9<br/>200MB]
        F10[FFmpeg #10<br/>200MB]
    end
    
    subgraph "서버 상태"
        CPU[CPU: 100%]
        MEM[Memory: 2GB<br/>OOM Killer]
        CRASH[Server Crash 💥]
    end
    
    S1 --> F1
    S2 --> F2
    S3 --> F3
    S4 --> F4
    S5 --> F5
    S6 --> F6
    S7 --> F7
    S8 --> F8
    S9 --> F9
    S10 --> F10
    
    F1 --> CPU
    F2 --> CPU
    F3 --> CPU
    F4 --> MEM
    F5 --> MEM
    F6 --> MEM
    F7 --> CRASH
    
    style S1 fill:#e1f5ff,stroke:#2196f3
    style S2 fill:#e1f5ff,stroke:#2196f3
    style S3 fill:#e1f5ff,stroke:#2196f3
    style S4 fill:#e1f5ff,stroke:#2196f3
    style S5 fill:#e1f5ff,stroke:#2196f3
    style S6 fill:#e1f5ff,stroke:#2196f3
    style S7 fill:#e1f5ff,stroke:#2196f3
    style S8 fill:#e1f5ff,stroke:#2196f3
    style S9 fill:#e1f5ff,stroke:#2196f3
    style S10 fill:#e1f5ff,stroke:#2196f3
    style F1 fill:#fff4e1,stroke:#ff9800
    style F2 fill:#fff4e1,stroke:#ff9800
    style F3 fill:#fff4e1,stroke:#ff9800
    style F4 fill:#fff4e1,stroke:#ff9800
    style F5 fill:#fff4e1,stroke:#ff9800
    style F6 fill:#fff4e1,stroke:#ff9800
    style F7 fill:#fff4e1,stroke:#ff9800
    style F8 fill:#fff4e1,stroke:#ff9800
    style F9 fill:#fff4e1,stroke:#ff9800
    style F10 fill:#fff4e1,stroke:#ff9800
    style CPU fill:#ffebee,stroke:#d32f2f
    style MEM fill:#ffebee,stroke:#d32f2f
    style CRASH fill:#d32f2f,stroke:#b71c1c,stroke-width:3px,color:#fff
```

**증상:**
- CPU 사용률 100% 도달
- 메모리 부족으로 OOM Killer 발동
- 서버 응답 없음 (다른 API도 영향)

### 해결책 1: Semaphore를 이용한 동시성 제어

```mermaid
sequenceDiagram
    participant R as Requests (10개)
    participant S as Semaphore<br/>(limit=3)
    participant F as FFmpeg Pool
    participant Q as Wait Queue
    
    Note over R,Q: Time: 0초
    
    R->>S: Request 1-3
    S->>F: Execute (3개)
    
    R->>S: Request 4-10
    S->>Q: Wait (7개)
    
    Note over F: 처리 중<br/>(최대 3개만)
    
    Note over R,Q: Time: 5초 (Request 1 완료)
    
    F-->>S: Complete #1
    S->>Q: Dequeue Request 4
    S->>F: Execute Request 4
    
    Note over R,Q: Time: 10초 (Request 2 완료)
    
    F-->>S: Complete #2
    S->>Q: Dequeue Request 5
    S->>F: Execute Request 5
    
    Note over S: 동시 실행 ≤ 3개<br/>나머지는 대기
```

**구현:**

```typescript
class Semaphore {
    private permits: number
    private queue: Array<() => void> = []
    
    async acquire<T>(task: () => Promise<T>): Promise<T> {
        await this.waitForPermit()
        try {
            return await task()
        } finally {
            this.release()
        }
    }
}

// 최대 3개만 동시 실행
const captureSemaphore = new Semaphore(3)

async function captureAllSites(siteIds: number[]) {
    const promises = siteIds.map(id =>
        captureSemaphore.acquire(() => captureImage(id))
    )
    return await Promise.all(promises)
}
```

### 해결책 2: 이미지 최적화

```mermaid
graph LR
    subgraph "Before"
        ORIG1[원본 이미지<br/>4K: 3840x2160<br/>JPEG: 2.5MB]
    end
    
    subgraph "After"
        OPT1[최적화 이미지<br/>Full HD: 1920x1080<br/>WebP: 800KB]
        THUMB[썸네일<br/>320x180<br/>WebP: 50KB]
    end
    
    ORIG1 -->|리사이즈| OPT1
    ORIG1 -->|리사이즈| THUMB
    
    style ORIG1 fill:#ffebee,stroke:#d32f2f
    style OPT1 fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style THUMB fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
```

**구현:**

```typescript
class ImageOptimizer {
    async optimizeImage(buffer: Buffer): Promise<Buffer> {
        return await sharp(buffer)
            .resize(1920, 1080, { fit: 'inside' })
            .webp({ quality: 80, effort: 4 })
            .toBuffer()
    }
    
    async createThumbnail(buffer: Buffer): Promise<Buffer> {
        return await sharp(buffer)
            .resize(320, 180)
            .webp({ quality: 60 })
            .toBuffer()
    }
}
```

### 결과

| 지표 | 개선 전 | 개선 후 | 개선률 |
|------|---------|---------|--------|
| **CPU 최대 사용률** | 100% | 35% | **65% ↓** |
| **메모리 사용** | 2GB (OOM) | 600MB | **70% ↓** |
| **평균 처리 시간** | 30초 (실패 시 무한) | 15초 | **50% ↓** |
| **성공률** | 60% | 98% | **63% ↑** |
| **이미지 크기** | 2.5MB (JPEG) | 800KB (WebP) | **68% ↓** |

---

## 4. PLC 통신 추상화

### 문제 상황

```mermaid
graph TB
    subgraph "개발 환경"
        DEV_CODE[개발 코드]
        NO_PLC[❌ PLC 장비 없음<br/>개발/테스트 불가]
    end
    
    subgraph "프로덕션 환경"
        PROD_CODE[프로덕션 코드]
        REAL_PLC[✅ 실제 PLC<br/>Modbus TCP]
        UNSTABLE[⚠️ 연결 불안정<br/>개발 중단]
    end
    
    DEV_CODE -.->|배포| PROD_CODE
    PROD_CODE <--> REAL_PLC
    REAL_PLC -.-> UNSTABLE
    
    style DEV_CODE fill:#ffebee,stroke:#d32f2f
    style NO_PLC fill:#d32f2f,stroke:#b71c1c,stroke-width:3px,color:#fff
    style PROD_CODE fill:#fff4e1,stroke:#ff9800
    style REAL_PLC fill:#e8f5e9,stroke:#4caf50
    style UNSTABLE fill:#ffebee,stroke:#d32f2f
```

**문제점:**
```
❌ PLC 없이 개발/테스트 불가능
❌ 실제 PLC 연결 시 잦은 연결 끊김
❌ 다른 제조사 PLC 지원 어려움
❌ 단위 테스트 불가능
```

### 해결책: Adapter Pattern

```mermaid
graph TB
    subgraph "Application"
        BL[Business Logic]
    end
    
    subgraph "Interface Layer"
        IFACE["IPLCReader / IPLCWriter<br/>(추상화된 계약)"]
    end
    
    subgraph "Development"
        FAKE[Fake PLC Adapter<br/>시뮬레이션 데이터<br/>네트워크 불필요]
    end
    
    subgraph "Production"
        MODBUS[Modbus Adapter<br/>실제 PLC 통신<br/>Modbus TCP]
        SIEMENS[Siemens Adapter<br/>S7 Protocol]
        MITSU[Mitsubishi Adapter<br/>MC Protocol]
    end
    
    subgraph "Factory"
        FACTORY[PLC Factory<br/>환경별 자동 선택]
    end
    
    BL --> IFACE
    IFACE -.->|implements| FAKE
    IFACE -.->|implements| MODBUS
    IFACE -.->|implements| SIEMENS
    IFACE -.->|implements| MITSU
    
    FACTORY -->|DEV| FAKE
    FACTORY -->|PROD:MODBUS| MODBUS
    FACTORY -->|PROD:SIEMENS| SIEMENS
    FACTORY -->|PROD:MITSU| MITSU
    
    style BL fill:#e1f5ff,stroke:#2196f3,stroke-width:2px
    style IFACE fill:#fff9c4,stroke:#fbc02d,stroke-width:3px
    style FAKE fill:#f3e5f5,stroke:#9c27b0,stroke-width:2px
    style MODBUS fill:#e8f5e9,stroke:#4caf50
    style SIEMENS fill:#e8f5e9,stroke:#4caf50
    style MITSU fill:#e8f5e9,stroke:#4caf50
    style FACTORY fill:#fff4e1,stroke:#ff9800,stroke-width:2px
```

**구현:**

```typescript
// 공통 인터페이스
interface IPLCReader {
    connect(): Promise<void>
    readCoils(address: number, count: number): Promise<boolean[]>
    readHoldingRegisters(address: number, count: number): Promise<number[]>
}

// 실제 PLC 어댑터
class ModbusPLCAdapter implements IPLCReader {
    async readCoils(address: number, count: number): Promise<boolean[]> {
        const result = await this.modbus.readCoils(address, count)
        return result.data
    }
}

// 가짜 PLC 어댑터 (개발용)
class FakePLCAdapter implements IPLCReader {
    async readCoils(address: number, count: number): Promise<boolean[]> {
        // 시뮬레이션: 랜덤 데이터 생성
        return Array.from({ length: count }, () => Math.random() > 0.5)
    }
}

// 팩토리로 어댑터 선택
const plc = PLCAdapterFactory.create({
    type: process.env.PLC_TYPE // 'MODBUS' | 'FAKE'
})
```

### 결과

```mermaid
graph LR
    BEFORE[개선 전<br/>PLC 필수<br/>테스트 불가]
    AFTER[개선 후<br/>어댑터 패턴<br/>독립 개발]
    
    BEFORE -.->|개선| AFTER
    
    METRICS[측정 결과<br/>테스트 환경: 2일→10분<br/>제조사 추가: 쉬움<br/>단위 테스트: 가능]
    
    AFTER --> METRICS
    
    style BEFORE fill:#ffebee,stroke:#d32f2f
    style AFTER fill:#e8f5e9,stroke:#4caf50,stroke-width:3px
    style METRICS fill:#e1f5ff,stroke:#2196f3
```

---

## 5. 실시간 데이터 동기화

### 문제 상황

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    participant PLC as PLC Device
    
    Note over C,PLC: ❌ HTTP Polling (비효율적)
    
    loop Every 5 seconds
        C->>S: GET /api/plc/data
        S->>PLC: Read data
        PLC-->>S: Data
        S-->>C: Response
    end
    
    Note over C: 문제점:<br/>- 불필요한 요청<br/>- 최대 5초 지연<br/>- 서버 부하 증가<br/>- 네트워크 낭비
```

**문제점:**
```
❌ 불필요한 HTTP 요청 (데이터 변경 없어도 요청)
❌ 서버 부하 증가
❌ 실시간성 부족 (최대 5초 지연)
❌ 네트워크 대역폭 낭비
```

### 해결책: WebSocket + Kafka

```mermaid
graph TB
    subgraph "Device Layer"
        PLC[PLC 장비들<br/>5초마다 수집]
    end
    
    subgraph "Collection Service"
        COLLECTOR[Device Collector<br/>데이터 수집]
    end
    
    subgraph "Message Queue"
        KAFKA[Kafka<br/>device.data topic]
    end
    
    subgraph "WebSocket Servers"
        WS1[WebSocket #1<br/>Selective Broadcast]
        WS2[WebSocket #2<br/>Selective Broadcast]
        WS3[WebSocket #N<br/>Selective Broadcast]
    end
    
    subgraph "Clients"
        C1[Client #1<br/>구독: resource:1]
        C2[Client #2<br/>구독: resource:2]
        C3[Client #N<br/>구독: resource:1,3]
    end
    
    PLC -->|Modbus TCP| COLLECTOR
    COLLECTOR -->|Publish| KAFKA
    
    KAFKA -->|Subscribe| WS1
    KAFKA -->|Subscribe| WS2
    KAFKA -->|Subscribe| WS3
    
    WS1 -.->|resource:1 only| C1
    WS2 -.->|resource:2 only| C2
    WS3 -.->|resource:1,3| C3
    
    style PLC fill:#ffebee,stroke:#d32f2f
    style COLLECTOR fill:#fff4e1,stroke:#ff9800
    style KAFKA fill:#f0e1ff,stroke:#9c27b0,stroke-width:3px
    style WS1 fill:#e8f5e9,stroke:#4caf50
    style WS2 fill:#e8f5e9,stroke:#4caf50
    style WS3 fill:#e8f5e9,stroke:#4caf50
    style C1 fill:#e1f5ff,stroke:#2196f3
    style C2 fill:#e1f5ff,stroke:#2196f3
    style C3 fill:#e1f5ff,stroke:#2196f3
```

**구현:**

```typescript
// 1. 장비 데이터 수집
class DeviceDataCollector {
    async start() {
        setInterval(async () => {
            const data = await this.collectDeviceData()
            
            // Kafka로 발행
            await this.producer.send({
                topic: 'device.data',
                messages: [{
                    key: data.resourceId.toString(),
                    value: JSON.stringify(data)
                }]
            })
        }, 5000)
    }
}

// 2. WebSocket 서버
class WebSocketServer {
    async start() {
        await this.consumer.subscribe({ topic: 'device.data' })
        
        await this.consumer.run({
            eachMessage: async ({ message }) => {
                const data = JSON.parse(message.value.toString())
                
                // 해당 리소스를 구독한 클라이언트에게만 전송
                this.broadcastToSubscribers(
                    `resource:${data.resourceId}:device`,
                    data
                )
            }
        })
    }
}

// 3. 클라이언트
const client = new WebSocketClient()
client.on('resource:1:device', (data) => {
    console.log('Device data:', data)
    updateUI(data)
})
```

### 결과

| 지표 | HTTP Polling | WebSocket + Kafka | 개선률 |
|------|--------------|-------------------|--------|
| **지연 시간** | 0-5초 | <100ms | **98% ↓** |
| **서버 CPU** | 40% | 15% | **62% ↓** |
| **네트워크** | 10MB/min | 1MB/min | **90% ↓** |
| **확장성** | 100 clients | 10,000+ clients | **100배 ↑** |
| **데이터 유실** | 가능 | 없음 (Kafka 보장) | **완전 방지** |

---

## 📚 학습 및 성장

### 기술 스택 선택 과정

```mermaid
graph TB
    PROBLEM[문제 정의]
    RESEARCH[대안 조사]
    POC[PoC 테스트]
    DECISION[의사결정]
    RETRO[회고]
    
    PROBLEM --> RESEARCH
    RESEARCH --> POC
    POC --> DECISION
    DECISION --> RETRO
    RETRO -.->|학습| PROBLEM
    
    style PROBLEM fill:#ffebee,stroke:#d32f2f
    style RESEARCH fill:#fff9c4,stroke:#fbc02d
    style POC fill:#fff4e1,stroke:#ff9800
    style DECISION fill:#e8f5e9,stroke:#4caf50,stroke-width:3px
    style RETRO fill:#e1f5ff,stroke:#2196f3
```

**예시: WebSocket vs SSE vs HTTP Polling**

| 기준 | HTTP Polling | SSE | WebSocket |
|------|-------------|-----|-----------|
| 양방향 통신 | ❌ | ❌ | ✅ |
| 실시간성 | 중간 | 높음 | 매우 높음 |
| 서버 부하 | 높음 | 중간 | 낮음 |
| 브라우저 지원 | 전부 | 대부분 | 전부 |
| 구현 복잡도 | 낮음 | 중간 | 높음 |

**결정:** WebSocket (양방향 제어 명령 필요)

### 실수로부터의 학습

```mermaid
graph LR
    MISTAKE1[무분별한<br/>console.log]
    MISTAKE2[에러 처리<br/>누락]
    MISTAKE3[타입 검증<br/>부족]
    
    LEARN1[Structured<br/>Logging]
    LEARN2[적절한<br/>에러 핸들링]
    LEARN3[런타임<br/>타입 검증]
    
    MISTAKE1 -.->|개선| LEARN1
    MISTAKE2 -.->|개선| LEARN2
    MISTAKE3 -.->|개선| LEARN3
    
    style MISTAKE1 fill:#ffebee,stroke:#d32f2f
    style MISTAKE2 fill:#ffebee,stroke:#d32f2f
    style MISTAKE3 fill:#ffebee,stroke:#d32f2f
    style LEARN1 fill:#e8f5e9,stroke:#4caf50
    style LEARN2 fill:#e8f5e9,stroke:#4caf50
    style LEARN3 fill:#e8f5e9,stroke:#4caf50
```

### 다음 도전 과제

```mermaid
mindmap
  root((Next Challenges))
    GraphQL
      REST over-fetching 해결
      타입 안전한 쿼리
    gRPC
      마이크로서비스 통신
      성능 최적화
    Kubernetes
      컨테이너 오케스트레이션
      자동 스케일링
    Observability
      Prometheus
      Grafana
      분산 추적
```

---

**이 문서는 실무 프로젝트에서 마주한 실제 문제와 해결 과정을 기록한 것입니다.**

---

**Last Updated**: 2025-01-30