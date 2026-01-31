# 🌡️ Smart Road Watering System - Backend Architecture

[![TypeScript](https://img.shields.io/badge/TypeScript-5.0+-blue.svg)](https://www.typescriptlang.org/)
[![Bun](https://img.shields.io/badge/Bun-1.0+-orange.svg)](https://bun.sh/)
[![Architecture](https://img.shields.io/badge/Architecture-Event--Driven-green.svg)]()
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**도로 살수 시스템을 위한 고성능 IoT 백엔드 아키텍처**

> 이 문서는 실무에서 설계하고 구현한 프로덕션 레벨 시스템의 아키텍처와 핵심 설계 패턴을 다룹니다.

---

## 📋 목차

- [프로젝트 개요](#-프로젝트-개요)
- [시스템 아키텍처](#-시스템-아키텍처)
- [핵심 설계 패턴](#-핵심-설계-패턴)
- [기술적 의사결정](#-기술적-의사결정)
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

```mermaid
graph TB
    subgraph "기술적 도전 과제"
        C1[실시간성<br/>PLC 5초 간격<br/>데이터 동기화]
        C2[확장성<br/>다중 사이트<br/>동시 제어]
        C3[안정성<br/>네트워크 불안정<br/>환경 대응]
        C4[보안<br/>산업용 IoT<br/>접근 제어]
    end
    
    subgraph "설계 해결 방안"
        S1[Event-Driven<br/>Architecture]
        S2[Adapter Pattern<br/>PLC 추상화]
        S3[WebSocket +<br/>Kafka]
        S4[JWT + MFA<br/>RBAC]
    end
    
    C1 --> S1
    C2 --> S2
    C3 --> S3
    C4 --> S4
    
    style C1 fill:#ffebee,stroke:#d32f2f
    style C2 fill:#ffebee,stroke:#d32f2f
    style C3 fill:#ffebee,stroke:#d32f2f
    style C4 fill:#ffebee,stroke:#d32f2f
    style S1 fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style S2 fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style S3 fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style S4 fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
```

> **Note**: 구현 과정의 기술적 챌린지와 성능 최적화 경험은 [TECHNICAL_CHALLENGES.md](TECHNICAL_CHALLENGES.md)에서 확인하실 수 있습니다.

---

## 🏗️ 시스템 아키텍처

### 전체 시스템 구조

```mermaid
graph TB
    subgraph "Client Layer"
        WEB[Web Dashboard]
        MOBILE[Mobile App]
    end
    
    subgraph "API Gateway"
        NGINX[Nginx<br/>- Load Balancing<br/>- SSL/TLS<br/>- Rate Limiting]
    end
    
    subgraph "Backend Cluster"
        BE1[Backend #1<br/>Bun.js + ElysiaJS]
        BE2[Backend #2<br/>Bun.js + ElysiaJS]
        BE3[Backend #N<br/>Bun.js + ElysiaJS]
    end
    
    subgraph "Message Queue"
        KAFKA[Kafka Cluster<br/>Event Stream]
    end
    
    subgraph "Data Layer"
        MYSQL[(MySQL<br/>Master-Slave<br/>ACID 보장)]
        MONGO[(MongoDB<br/>Replica Set<br/>로그 저장)]
        REDIS[(Redis<br/>Cache<br/>Session)]
    end
    
    subgraph "Device Layer"
        PLC1[PLC #1<br/>Site A]
        PLC2[PLC #2<br/>Site B]
        PLCN[PLC #N<br/>Site N]
    end
    
    WEB --> NGINX
    MOBILE --> NGINX
    
    NGINX --> BE1
    NGINX --> BE2
    NGINX --> BE3
    
    BE1 <--> KAFKA
    BE2 <--> KAFKA
    BE3 <--> KAFKA
    
    BE1 --> MYSQL
    BE1 --> MONGO
    BE1 --> REDIS
    
    BE1 <-.->|Modbus TCP| PLC1
    BE2 <-.->|Modbus TCP| PLC2
    BE3 <-.->|Modbus TCP| PLCN
    
    style WEB fill:#e1f5ff,stroke:#2196f3
    style MOBILE fill:#e1f5ff,stroke:#2196f3
    style NGINX fill:#fff9c4,stroke:#fbc02d,stroke-width:2px
    style BE1 fill:#fff4e1,stroke:#ff9800
    style BE2 fill:#fff4e1,stroke:#ff9800
    style BE3 fill:#fff4e1,stroke:#ff9800
    style KAFKA fill:#f0e1ff,stroke:#9c27b0,stroke-width:3px
    style MYSQL fill:#ffe1e1,stroke:#f44336
    style MONGO fill:#ffe1e1,stroke:#f44336
    style REDIS fill:#ffe1e1,stroke:#f44336
    style PLC1 fill:#ffebee,stroke:#d32f2f
    style PLC2 fill:#ffebee,stroke:#d32f2f
    style PLCN fill:#ffebee,stroke:#d32f2f
```

### 계층화된 구조 (Layered Architecture)

```mermaid
graph TB
    subgraph "Presentation Layer"
        API[API Endpoints<br/>REST / WebSocket]
    end
    
    subgraph "Business Logic Layer"
        CTRL[Controllers<br/>비즈니스 로직]
        SVC[Services<br/>도메인 로직]
    end
    
    subgraph "Data Access Layer"
        REPO[Repositories<br/>데이터 접근]
        ORM[Drizzle ORM<br/>타입 안전]
    end
    
    subgraph "Infrastructure Layer"
        DB[Databases<br/>MySQL/MongoDB/Redis]
        MQ[Message Queue<br/>Kafka]
        CACHE[Caching<br/>Redis]
    end
    
    API --> CTRL
    CTRL --> SVC
    SVC --> REPO
    REPO --> ORM
    ORM --> DB
    
    SVC -.-> MQ
    SVC -.-> CACHE
    
    style API fill:#e1f5ff,stroke:#2196f3,stroke-width:2px
    style CTRL fill:#fff4e1,stroke:#ff9800
    style SVC fill:#fff4e1,stroke:#ff9800
    style REPO fill:#e8f5e9,stroke:#4caf50
    style ORM fill:#e8f5e9,stroke:#4caf50
    style DB fill:#ffe1e1,stroke:#f44336
    style MQ fill:#f0e1ff,stroke:#9c27b0
    style CACHE fill:#ffe1e1,stroke:#f44336
```

**설계 이유:**
- ✅ 각 계층의 독립적 변경 가능
- ✅ 단위 테스트 용이성
- ✅ 명확한 책임 분리
- ✅ 유지보수성 향상

### 마이크로서비스 지향 아키텍처

```mermaid
graph LR
    subgraph "Independent Modules"
        AUTH[Auth Module<br/>인증/인가]
        COOLING[Cooling Road<br/>살수 제어]
        WS[WebSocket<br/>실시간 통신]
        PLC_MOD[PLC Module<br/>장비 통신]
        AI[AI Module<br/>자동 판단]
    end
    
    subgraph "Event Bus"
        KAFKA_BUS[Kafka Message Bus]
    end
    
    AUTH --> KAFKA_BUS
    COOLING --> KAFKA_BUS
    WS --> KAFKA_BUS
    PLC_MOD --> KAFKA_BUS
    AI --> KAFKA_BUS
    
    KAFKA_BUS -.->|Subscribe| AUTH
    KAFKA_BUS -.->|Subscribe| COOLING
    KAFKA_BUS -.->|Subscribe| WS
    KAFKA_BUS -.->|Subscribe| PLC_MOD
    KAFKA_BUS -.->|Subscribe| AI
    
    style AUTH fill:#e1f5ff,stroke:#2196f3
    style COOLING fill:#fff4e1,stroke:#ff9800
    style WS fill:#e8f5e9,stroke:#4caf50
    style PLC_MOD fill:#ffe1e1,stroke:#f44336
    style AI fill:#f0e1ff,stroke:#9c27b0
    style KAFKA_BUS fill:#fff9c4,stroke:#fbc02d,stroke-width:3px
```

---

## 🎨 핵심 설계 패턴

### 1. Adapter Pattern - PLC 통신 추상화

**문제:** 
- 개발 환경에 실제 PLC 장비가 없어 테스트 불가
- 다양한 PLC 제조사별 프로토콜 차이
- 프로덕션/개발 환경 분리 필요

**해결책:**

```mermaid
graph TB
    subgraph "Application Layer"
        BL[Business Logic<br/>장비 제어 로직]
    end
    
    subgraph "Interface Layer"
        IFACE["IPLCReader / IPLCWriter<br/>(추상화된 계약)"]
    end
    
    subgraph "Development Adapters"
        FAKE[Fake PLC Adapter<br/>시뮬레이션 데이터<br/>네트워크 불필요]
    end
    
    subgraph "Production Adapters"
        MODBUS[Modbus Adapter<br/>실제 PLC 통신<br/>Modbus TCP]
        SIEMENS[Siemens Adapter<br/>S7 Protocol]
        MITSU[Mitsubishi Adapter<br/>MC Protocol]
    end
    
    subgraph "Factory Pattern"
        FACTORY[PLC Adapter Factory<br/>환경별 자동 선택]
    end
    
    BL --> IFACE
    IFACE -.->|implements| FAKE
    IFACE -.->|implements| MODBUS
    IFACE -.->|implements| SIEMENS
    IFACE -.->|implements| MITSU
    
    FACTORY -->|NODE_ENV=dev| FAKE
    FACTORY -->|PLC_TYPE=MODBUS| MODBUS
    FACTORY -->|PLC_TYPE=SIEMENS| SIEMENS
    FACTORY -->|PLC_TYPE=MITSU| MITSU
    
    style BL fill:#e1f5ff,stroke:#2196f3,stroke-width:2px
    style IFACE fill:#fff9c4,stroke:#fbc02d,stroke-width:3px
    style FAKE fill:#f3e5f5,stroke:#9c27b0,stroke-width:2px
    style MODBUS fill:#e8f5e9,stroke:#4caf50
    style SIEMENS fill:#e8f5e9,stroke:#4caf50
    style MITSU fill:#e8f5e9,stroke:#4caf50
    style FACTORY fill:#fff4e1,stroke:#ff9800,stroke-width:2px
```

**구현 예시:**

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
    async readCoils(address: number, count: number): Promise<boolean[]> {
        const result = await this.connection.readCoils(address, count)
        return result.data
    }
}

// 테스트용 가짜 PLC
class FakePLCAdapter implements IPLCReader, IPLCWriter {
    async readCoils(address: number, count: number): Promise<boolean[]> {
        return Array.from({ length: count }, () => Math.random() > 0.5)
    }
}

// 팩토리 패턴
class PLCAdapterFactory {
    static create(config: PLCConfig): IPLCReader & IPLCWriter {
        if (config.mode === 'PRODUCTION') {
            return new ModbusPLCAdapter(config)
        } else {
            return new FakePLCAdapter()
        }
    }
}
```

**결과:**
- ✅ 환경 변수 하나로 실제/가짜 PLC 전환
- ✅ PLC 없이도 전체 시스템 개발/테스트 가능
- ✅ 새로운 PLC 제조사 추가 시 새 어댑터만 구현
- ✅ 단위 테스트 작성 가능

---

### 2. Repository Pattern - 데이터 접근 추상화

```mermaid
graph TB
    subgraph "Service Layer"
        SVC[Business Service<br/>비즈니스 로직]
    end
    
    subgraph "Repository Interface"
        IFACE[IRepository<br/>데이터 접근 계약]
    end
    
    subgraph "Implementations"
        DRIZZLE[Drizzle Repository<br/>실제 ORM 구현]
        MOCK[Mock Repository<br/>테스트용 구현]
    end
    
    subgraph "Database"
        DB[(MySQL<br/>실제 데이터)]
        MEM[(In-Memory<br/>테스트 데이터)]
    end
    
    SVC --> IFACE
    IFACE -.->|implements| DRIZZLE
    IFACE -.->|implements| MOCK
    
    DRIZZLE --> DB
    MOCK --> MEM
    
    style SVC fill:#e1f5ff,stroke:#2196f3,stroke-width:2px
    style IFACE fill:#fff9c4,stroke:#fbc02d,stroke-width:3px
    style DRIZZLE fill:#e8f5e9,stroke:#4caf50
    style MOCK fill:#f3e5f5,stroke:#9c27b0
    style DB fill:#ffe1e1,stroke:#f44336
    style MEM fill:#ffe1e1,stroke:#f44336
```

**결과:**
- ✅ 비즈니스 로직과 데이터 접근 계층 분리
- ✅ Mock 레포지토리로 단위 테스트 가능
- ✅ ORM 교체 시 레포지토리만 수정

---

### 3. Event-Driven Architecture - Kafka 메시지 큐

```mermaid
graph LR
    subgraph "Event Producers"
        P1[Operation Service<br/>작업 이벤트]
        P2[PLC Service<br/>장비 이벤트]
        P3[External API<br/>외부 데이터]
    end
    
    subgraph "Kafka Topics"
        T1[device.control<br/>장비 제어 명령]
        T2[device.data.updated<br/>장비 데이터 업데이트]
        T3[operation.started<br/>작업 시작]
        T4[operation.stopped<br/>작업 중지]
        T5[external.data.received<br/>외부 데이터 수신]
        T6[websocket.broadcast<br/>실시간 브로드캐스트]
    end
    
    subgraph "Event Consumers"
        C1[History Logger<br/>이력 기록]
        C2[WebSocket Server<br/>실시간 전송]
        C3[AI Service<br/>분석 및 판단]
        C4[Notification<br/>알림 발송]
    end
    
    P1 --> T3
    P1 --> T4
    P2 --> T1
    P2 --> T2
    P3 --> T5
    
    T3 --> C1
    T3 --> C2
    T4 --> C1
    T4 --> C3
    T2 --> C2
    T5 --> C3
    
    style P1 fill:#e1f5ff,stroke:#2196f3
    style P2 fill:#fff4e1,stroke:#ff9800
    style P3 fill:#e8f5e9,stroke:#4caf50
    style T1 fill:#f0e1ff,stroke:#9c27b0
    style T2 fill:#f0e1ff,stroke:#9c27b0
    style T3 fill:#f0e1ff,stroke:#9c27b0
    style T4 fill:#f0e1ff,stroke:#9c27b0
    style T5 fill:#f0e1ff,stroke:#9c27b0
    style T6 fill:#f0e1ff,stroke:#9c27b0
    style C1 fill:#fff9c4,stroke:#fbc02d
    style C2 fill:#fff9c4,stroke:#fbc02d
    style C3 fill:#fff9c4,stroke:#fbc02d
    style C4 fill:#fff9c4,stroke:#fbc02d
```

**결과:**
- ✅ 서비스 간 느슨한 결합
- ✅ 비동기 처리로 응답 속도 향상
- ✅ 이벤트 재처리 가능 (장애 복구)
- ✅ 새로운 구독자 추가 용이

---

### 4. Semaphore Pattern - 동시성 제어

```mermaid
sequenceDiagram
    participant R as Requests<br/>(10개 사이트)
    participant S as Semaphore<br/>(permits=3)
    participant F as FFmpeg Pool
    participant Q as Wait Queue
    
    Note over R,Q: 초기 상태: 10개 요청 동시 도착
    
    R->>S: Request 1-3
    S->>F: Execute #1, #2, #3
    
    R->>S: Request 4-10
    S->>Q: Enqueue #4-10 (대기)
    
    Note over F: FFmpeg 실행 중<br/>(최대 3개만)
    
    Note over R,Q: 5초 후: Request #1 완료
    
    F-->>S: Complete #1
    S->>Q: Dequeue #4
    S->>F: Execute #4
    
    Note over S: 동시 실행 수 유지<br/>(항상 ≤ 3)
    
    Note over R,Q: 순차적으로 처리<br/>CPU/메모리 안정화
```

**결과:**
- ✅ CPU 사용률 100% → 35%
- ✅ 메모리 안정화 (OOM 에러 제거)
- ✅ 응답 시간 예측 가능

---

## 💡 기술적 의사결정

### 1. Bun.js를 선택한 이유

```mermaid
graph TB
    subgraph "Node.js"
        N1[시작 시간: 느림]
        N2[번들 크기: 큰 편]
        N3[TS 지원: 별도 빌드]
        N4[패키지: npm 느림]
    end
    
    subgraph "Deno"
        D1[시작 시간: 보통]
        D2[번들 크기: 중간]
        D3[TS 지원: 네이티브]
        D4[패키지: 제한적]
    end
    
    subgraph "Bun.js ✅"
        B1[시작 시간: 빠름]
        B2[번들 크기: 작음]
        B3[TS 지원: 네이티브]
        B4[패키지: npm 호환]
        B5[개발 경험: 우수]
    end
    
    style N1 fill:#ffebee,stroke:#d32f2f
    style N2 fill:#ffebee,stroke:#d32f2f
    style N3 fill:#ffebee,stroke:#d32f2f
    style N4 fill:#ffebee,stroke:#d32f2f
    style D1 fill:#fff4e1,stroke:#ff9800
    style D2 fill:#fff4e1,stroke:#ff9800
    style D3 fill:#fff4e1,stroke:#ff9800
    style D4 fill:#ffebee,stroke:#d32f2f
    style B1 fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style B2 fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style B3 fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style B4 fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style B5 fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
```

**선택 이유:**
- TypeScript 네이티브 지원으로 빌드 과정 불필요
- npm 생태계 완전 호환
- 빠른 개발 사이클 (Hot reload)
- 경량화된 런타임

---

### 2. Polyglot Persistence 전략

```mermaid
graph TB
    subgraph "MySQL - ACID 보장"
        M1[사용자/계정 정보]
        M2[리소스 정보]
        M3[작업 이력<br/>정규화된 데이터]
        M4[트랜잭션 필수]
    end
    
    subgraph "MongoDB - 유연한 스키마"
        MG1[시스템 로그]
        MG2[에러 로그]
        MG3[이벤트 히스토리]
        MG4[비정형 데이터]
    end
    
    subgraph "Redis - 빠른 읽기"
        R1[사용자 세션]
        R2[API 응답 캐시]
        R3[Rate Limiting]
        R4[실시간 카운터]
    end
    
    APP[Application]
    
    APP -->|CRUD| M1
    APP -->|CRUD| M2
    APP -->|Transaction| M3
    APP -->|Logging| MG1
    APP -->|Logging| MG2
    APP -->|Cache| R1
    APP -->|Cache| R2
    
    style M1 fill:#ffe1e1,stroke:#f44336
    style M2 fill:#ffe1e1,stroke:#f44336
    style M3 fill:#ffe1e1,stroke:#f44336
    style M4 fill:#ffe1e1,stroke:#f44336
    style MG1 fill:#e8f5e9,stroke:#4caf50
    style MG2 fill:#e8f5e9,stroke:#4caf50
    style MG3 fill:#e8f5e9,stroke:#4caf50
    style MG4 fill:#e8f5e9,stroke:#4caf50
    style R1 fill:#e1f5ff,stroke:#2196f3
    style R2 fill:#e1f5ff,stroke:#2196f3
    style R3 fill:#e1f5ff,stroke:#2196f3
    style R4 fill:#e1f5ff,stroke:#2196f3
    style APP fill:#f0e1ff,stroke:#9c27b0,stroke-width:3px
```

---

## 🔐 보안 설계

### 1. JWT + MFA 인증

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Auth Service
    participant MFA as MFA Service
    participant DB as Database
    
    Note over C,DB: 1단계: 기본 인증
    
    C->>A: Login (email, password)
    A->>DB: Verify credentials
    DB-->>A: User data
    
    Note over A: Password 검증 성공
    
    A->>C: MFA Challenge
    
    Note over C,MFA: 2단계: MFA 인증
    
    C->>MFA: TOTP Token
    MFA->>MFA: Verify TOTP<br/>(±30초 허용)
    
    alt MFA Success
        MFA-->>A: Verified
        A->>A: Generate JWT<br/>(24시간 유효)
        A-->>C: Access Token + Refresh Token
    else MFA Failed
        MFA-->>C: 401 Unauthorized
    end
```

---

### 2. Rate Limiting

```mermaid
graph TB
    REQUEST[Client Request]
    
    subgraph "Rate Limiter"
        EXTRACT[Extract IP/User ID]
        CHECK[Redis Counter<br/>Check]
        DECISION{Allowed?}
    end
    
    subgraph "Redis"
        COUNTER[Request Counter<br/>Key: ratelimit:IP:timestamp<br/>TTL: 1분]
    end
    
    subgraph "Response"
        ALLOW[200 OK<br/>X-RateLimit-Remaining: N]
        DENY[429 Too Many Requests<br/>X-RateLimit-Reset: timestamp]
    end
    
    REQUEST --> EXTRACT
    EXTRACT --> CHECK
    CHECK <--> COUNTER
    CHECK --> DECISION
    
    DECISION -->|≤100 requests| ALLOW
    DECISION -->|>100 requests| DENY
    
    COUNTER -.->|Increment| COUNTER
    
    style REQUEST fill:#e1f5ff,stroke:#2196f3
    style EXTRACT fill:#fff4e1,stroke:#ff9800
    style CHECK fill:#fff4e1,stroke:#ff9800
    style DECISION fill:#fff9c4,stroke:#fbc02d,stroke-width:2px
    style COUNTER fill:#ffe1e1,stroke:#f44336
    style ALLOW fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    style DENY fill:#ffebee,stroke:#d32f2f,stroke-width:2px
```

---

### 3. RBAC (Role-Based Access Control)

```mermaid
graph TB
    subgraph "User Roles"
        USER[USER<br/>일반 사용자]
        MAINT[MAINTENANCE<br/>유지보수 담당자]
        DEV[DEVELOPER<br/>개발자]
        ORG[ORGANIZE<br/>조직 관리자]
    end
    
    subgraph "Permissions"
        P1[site:read<br/>사이트 조회]
        P2[site:write<br/>사이트 수정]
        P3[plc:control<br/>PLC 제어]
        P4[users:manage<br/>사용자 관리]
        P5[logs:view<br/>로그 조회]
    end
    
    USER --> P1
    
    MAINT --> P1
    MAINT --> P2
    MAINT --> P3
    MAINT --> P5
    
    DEV --> P1
    DEV --> P2
    DEV --> P3
    DEV --> P5
    
    ORG --> P1
    ORG --> P2
    ORG --> P3
    ORG --> P4
    ORG --> P5
    
    style USER fill:#e1f5ff,stroke:#2196f3
    style MAINT fill:#fff4e1,stroke:#ff9800
    style DEV fill:#e8f5e9,stroke:#4caf50
    style ORG fill:#f0e1ff,stroke:#9c27b0,stroke-width:2px
    style P1 fill:#fff9c4,stroke:#fbc02d
    style P2 fill:#fff9c4,stroke:#fbc02d
    style P3 fill:#ffe1e1,stroke:#f44336
    style P4 fill:#ffe1e1,stroke:#f44336
    style P5 fill:#fff9c4,stroke:#fbc02d
```

---

## 📚 관련 포트폴리오

이 설계 원칙은 다른 도메인에도 적용 가능합니다:

### 🎨 [Main Game Architecture](https://github.com/1985jwlee/portpolio_main)

**동일한 원칙의 게임 도메인 적용**

| 원칙 | IoT Backend | Game Server |
|------|------------|-------------|
| **외부 격리** | PLC 장애 시 서비스 유지 | DB 장애 시 게임 진행 |
| **이벤트 기반** | Kafka Event Stream | Kafka Event Stream |
| **계약 안정성** | API 스키마 불변 | 운영 API 불변 |
| **비동기 처리** | WebSocket + Kafka | Command → Event |

### 📊 [Coin Data API](https://github.com/1985jwlee/portpolio_coindataapi)

**외부 API 격리 패턴**

| 원칙 | IoT Backend | Coin API |
|------|------------|----------|
| **외부 격리** | PLC 프로토콜 추상화 | 거래소 API 추상화 |
| **정규화** | Modbus → Internal Schema | External API → Internal Schema |
| **캐싱** | Redis Multi-tier | In-Memory Cache |

> **핵심 메시지**: "설계 원칙은 도메인을 넘어 일반화 가능합니다"

---

## 📧 Contact

**GitHub**: [@1985jwlee](https://github.com/1985jwlee)  
**Email**: leejae.w.jl@icloud.com

---

## 📝 License

이 문서는 설계 포트폴리오로, 학습 및 평가 목적으로 공개되었습니다.

---

**Last Updated**: 2025-01-30

**Note**: 이 프로젝트는 실무 프로덕션 시스템의 아키텍처와 설계 판단력을 증명하기 위한 자료입니다.