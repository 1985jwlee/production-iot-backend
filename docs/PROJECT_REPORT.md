# 📊 프로젝트 현황 보고서

[← README](../README.md)

**프로젝트명**: Production IoT Backend — 쿨링로드 시스템  
**현재 버전**: v4.4.0  
**상태**: ✅ Production Ready  
**작성일**: 2026-03-19

---

## 프로젝트 개요

### 목적

도로 살수 장비(PLC)를 원격으로 제어하고, 센서·기상 데이터·AI 판단을 결합해 살수 시점을 자동 결정하는 백엔드 플랫폼. 모놀리식 TypeScript 서비스로 시작해, 실제 운영에서 발생한 문제를 기반으로 Kafka 이벤트 버스 기반 언어 폴리글랏 MSA로 점진적 전환 중.

### 포트폴리오 관점 핵심 메시지

```
✓ 모놀리식 → MSA 점진적 전환 — 문제를 측정하고 분리 시점 결정
✓ 언어 폴리글랏 (TypeScript · Go · C#) — 문제 특성 기반 언어 선택
✓ Kafka 이벤트 버스 — 서비스 간 비동기 통신, 장애 격리
✓ Polyglot Persistence — 데이터 특성별 저장소 선택
✓ 프로덕션 보안 설계 — JWT 이중 무효화, MFA/TOTP, RBAC
✓ 코드 리뷰 기반 Critical 버그 발견 — RBAC 우회, 평문 비밀번호 비교 등
```

---

## 시스템 통계 (v4.4.0)

### 코드베이스

```
TypeScript 파일:   44개+  (v4.4.0: PLC 4파일 C# 이관)
총 코드 라인:    ~13,000 lines
Go 서비스:       coolroad-image-service (별도 리포)
C# 서비스:       coolroad-plc-service (별도 리포)
문서 파일:         27개+ Markdown
```

### 서비스 구성

```
MSA 서비스:        3개 (TypeScript API · Go Image · C# PLC)
Controllers:       9개  (Presentation Layer)
Services:          9개  (Business Logic — @singleton DI)
Core Modules:     13개  (Data Access Layer)
Repositories:     15개  (MySQL 13 + TimescaleDB 2 — 전체 DI 등록)
DI 등록 총계:     44개  (Controller 9 + Service 9 + Repo 15 + Infra 7 + Snowflake ID 생성기 4)
```

### 데이터베이스

```
MySQL 테이블:          13개  (관계형 데이터)
TimescaleDB hypertables: 2개  (시계열 날씨·센서 — v4.0.0)
MongoDB Collections:    5개  (로그)
Redis Keys:           다수  (캐시·세션·멱등성)
MinIO Buckets:          2개  (prod / dev)
```

---

## 아키텍처 구성

### 레이어 구조

```mermaid
graph TB
    subgraph Client["Client Layer"]
        WEB["Web / Mobile"]
        ADMIN["Admin Dashboard"]
    end

    subgraph Gateway["Gateway Layer"]
        NGINX["Nginx :443\nRate Limiting 12 Zone\nSSL/TLS"]
    end

    subgraph Presentation["Presentation Layer (9 Controllers)"]
        C1["Auth"] C2["CoolingRoad"] C3["WebSocket"]
        C4["Scheduler"] C5["AI"] C6["CCTV"]
        C7["Admin"] C8["Maintenance"] C9["Notice"]
    end

    subgraph Business["Business Logic (9 Services — @singleton DI)"]
        S1["인증 서비스\nEmailService\nMFAService"]
        S2["분사 제어 서비스\nSchedulerService"]
        S3["WebSocket 서비스\nAIService\nCctvService"]
        S4["관리자 서비스\nMaintenanceService\nNoticeService"]
    end

    subgraph Data["Data Access Layer"]
        MYSQL_R["MySQL 베이스 레포지토리\n13개 Repository"]
        TSDB_R["TimescaleDB 베이스 레포지토리\n2개 Repository"]
        MONGO_M["MongoDB 로거"]
        REDIS_M["Redis 캐시 매니저"]
        KAFKA_M["Kafka 프로듀서\n+ Kafka 컨슈머"]
    end

    subgraph MSA["MSA Services"]
        PLC_S["coolroad-plc-service\nC# .NET 10"]
        IMG_S["coolroad-image-service\nGo 1.26"]
    end

    subgraph DB["Storage"]
        MY[("MySQL")] TSDB[("TimescaleDB")] MG[("MongoDB")]
        RD[("Redis")] MN[("MinIO")]
    end

    WEB & ADMIN --> NGINX --> Presentation
    Presentation --> Business
    Business --> Data
    Data --> MY & TSDB & MG & RD & MN
    KAFKA_M --> PLC_S & IMG_S

    style PLC_S fill:#4A90E2,color:#fff
    style IMG_S fill:#F5A623,color:#000
    style TSDB fill:#4A90E2,color:#fff
```

---

## MSA 전환 현황

### 전환 완료

| 서비스 | 언어 | 버전 | 상태 |
|--------|------|------|------|
| PLC 제어 (Modbus TCP + 분사) | C# .NET 10 | v4.4.0 | ✅ 운영 중 |
| 이미지 캡처 (RTSP → MinIO) | Go 1.26 | v4.3.0 | ✅ 운영 중 |

### 전환 예정

| 서비스 | 언어 | 시기 | 분리 이유 |
|--------|------|------|---------|
| 날씨 수집 (기상청 API) | C# ASP.NET Core | 2026 Q3 | 수집 주기 관리 독립화 |
| WebSocket 실시간 통신 | TypeScript/ElysiaJS | 2026 Q4 | 10,000+ 연결 수평 확장 |

---

## 보안 설계 현황

### 구현 완료

| 보안 요소 | 구현 내용 | 버전 |
|---------|---------|------|
| JWT 이중 무효화 | jwt_token_version + TrashboxJWT 블랙리스트 | v3.3.16~19 |
| 단일 세션 정책 | 로그인마다 version 증가 → 이전 기기 즉시 무효화 | v3.3.x |
| MFA/TOTP | 타이밍 공격 방지 + 재시도 제한 | v3.3.x |
| RBAC | USER / DEVELOPER / MAINTENANCE / ORGANIZE 역할 | v3.x |
| 이메일 인증 | 미인증 계정 로그인 차단 (v3.9.2 보안 강화) | v3.9.2 |
| Rate Limiting | Nginx 12개 Zone 독립 설정 | v3.5.x |
| bcrypt 전환 | 평문 비밀번호 SQL 비교 → bcrypt.verify (BUG-11) | v3.6.7 |

### 발견·수정한 Critical 버그

| ID | 내용 | 발견 방법 |
|----|------|---------|
| — | RBAC 우회 (async checkRole — Promise truthy 오판정) | 코드 리뷰 |
| BUG-11 | 평문 비밀번호 SQL WHERE 비교 | 코드 리뷰 |
| BUG-41 | 자동 분사 조건 자기비교 (항상 조건 충족) | 코드 리뷰 |
| BUG-58 | jwt_token_version mutation 경쟁 조건 | 코드 리뷰 |
| BUG-59 | MFA setup 인증 없음 — 타인 TOTP 시크릿 생성 가능 | 코드 리뷰 |
| BUG-34 | 이메일 미인증 계정 JWT 발급 | 코드 리뷰 |

모든 Critical 버그는 기능 테스트로는 발견 불가능한 종류. 코드 리뷰에서만 발견됐다.

---

## 성능 최적화 현황

| 최적화 항목 | 방법 | 버전 |
|-----------|------|------|
| Kafka 벌크 전송 | 0.3초 버퍼 배치 전송 | v3.3.8 |
| WebSocket O(1) 라우팅 | Map<userId, WebSocket> 보조 인덱스 | v4.2.1 |
| Redis Cache-Aside | 사용자·사이트·프로필 URL 캐싱 | v3.9.0 |
| Redis SET 인덱스 | 사이트-사용자 관계 인덱싱 → WS 라우팅 DB 쿼리 제거 | v3.9.9 |
| TimescaleDB hypertable | 시계열 쿼리 청크 인덱스 | v4.0.0 |
| Go goroutine Worker Pool | ffmpeg CPU 바운드 작업 격리 | v4.3.0 |
| C# 주기 타이머 | Modbus 폴링 OS 레벨 정밀 주기 | v4.4.0 |

---

## 코드 품질 현황

### 아키텍처 구조 개선

| 항목 | 이전 상태 | 현재 상태 | 버전 |
|------|---------|---------|------|
| Controller/Service 분리 | 동일 파일 1000줄+ | 10개 모듈 전체 분리 | v4.2.1 |
| Repository DI 등록 | 서비스마다 `new` 생성 | 15개 전체 @singleton DI | v4.2.2 |
| 로그 보일러플레이트 | 6개 컨트롤러 × 11그룹 반복 | 팩토리 메서드 2줄 | v4.2.3 |
| PLC 코드 | TS 모놀리식 ~1,135줄 | C# MSA 독립 서비스 | v4.4.0 |

### 잔여 기술 부채

| 항목 | 우선순위 | 내용 |
|------|---------|------|
| BUG-19 | 🔴 P1 | signInOrganizer 평문 비밀번호 비교 미수정 |
| BUG-35 | 🔴 P1 | API 응답에 password_hash/mfa_secret 노출 |
| H9 | 🟡 P2 | WebSocket 연결 시 JWT 검증 부재 |
| ARCH-7 | 🟡 P2 | SetSiteSettings 트랜잭션 미적용 |

---

## 기능 현황

### 핵심 기능 완성도

| 기능 | 상태 | 버전 |
|------|------|------|
| PLC Modbus TCP 제어 | ✅ C# MSA | v4.4.0 |
| 분사 자동화 (AI + 스케줄) | ✅ | v3.3.6 |
| 실시간 WebSocket | ✅ | v3.3.7 |
| JWT/MFA/RBAC 보안 | ✅ | v3.3.x |
| Kafka 이벤트 버스 + DLQ | ✅ | v3.3.7~3.3.8 |
| TimescaleDB 시계열 | ✅ | v4.0.0 |
| Redis 캐시 계층 | ✅ | v3.9.0 |
| CCTV MediaMTX WebRTC | ✅ | v4.2.0 |
| 이미지 캡처 Go MSA | ✅ | v4.3.0 |
| PLC C# MSA | ✅ | v4.4.0 |
| Nginx Rate Limiting + SSL | ✅ | v3.5.x |

### 남은 P1 작업

```
🔴 BUG-19: signInOrganizer bcrypt 전환 (보안)
🔴 BUG-35: API 응답 민감 필드 제거 (보안)
```

---

## 버전 히스토리 요약

| 버전 | 날짜 | 주요 내용 |
|------|------|---------|
| **v4.4.0** | 2026-03-17 | PLC MSA — C# .NET 10 백그라운드 서비스 |
| **v4.3.0** | 2026-03-17 | Image MSA — Go Worker Pool + Kafka |
| **v4.2.3** | 2026-03-15 | 베이스 컨트롤러 팩토리 메서드, BUG-69 IP 감지 수정 |
| **v4.2.2** | 2026-03-14 | Repository 15개 전체 DI 등록 (ARCH-3) |
| **v4.2.1** | 2026-03-13 | Controller/Service 분리 10/10, BUG-58/59, CQ-32 |
| **v4.2.0** | 2026-03-12 | CCTV MediaMTX WebRTC 전환 |
| **v4.0.0** | 2026-03-10 | TimescaleDB 도입, 시계열 데이터 이관 |
| **v3.9.x** | 2026-03-05~09 | Redis 캐시 계층, 보안 강화, 페이지네이션 |
| **v3.8.0** | 2026-03-04 | 에러 코드 분화 |
| **v3.5.x** | 2026-02-11~25 | HTTPS/SSL, Nginx 구조 개편, 환경 분리 |
| **v3.3.x** | 2026-01-27~02-10 | WebSocket + Kafka DLQ + 자동 분사 시스템 |

---

**Last Updated**: 2026-03-19 | **Version**: 4.4.0
