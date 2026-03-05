# Message Provider System Design

CloudMsg 단일 메시지 시스템을  
멀티 Provider 구조로 확장하기 위한 서버 설계 문서입니다.

---

# 1. Problem

기존 시스템은 CloudMsg 메시지 서비스만 사용하고 있습니다.

신규 요구사항

- SendTalk 메시지 서비스 추가
- 조직별 메시지 Provider 선택 가능
- 메시지 발송 이력 관리
- 설정 변경 이력 관리

---

# 2. Architecture

```text
Client
   │
   ▼
Message API
   │
   ▼
Message Service
   │
   ▼
Provider Router
   │
   ├── CloudMsg Provider
   └── SendTalk Provider
```

---

# 3. Database Design

![erd](docs/erd.png)

주요 테이블

- organizations
- organization_message_provider
- cloudmsg_config
- sendtalk_config
- message_log
- provider_audit

---

# 4. Message Flow

1. 메시지 발송 요청
2. organization_message_provider 조회
3. Provider 선택
4. Provider API 호출
5. message_log 저장

---

# 5. Technical Decisions

## Provider Abstraction

멀티 Provider 지원을 위해 Provider Router 구조를 사용하였습니다.

## Provider Config 분리

Provider별 API 구조가 다르기 때문에

- cloudmsg_config
- sendtalk_config

로 분리하였습니다.

---

# 6. 리스크

### 외부 API 장애

대응

- message_log 상태 기록
- 재시도 전략

### 중복 메세지

대응

- idempotency_key 도입

---

# 7. AI Usage

본 과제 수행 과정에서 ChatGPT를 활용하여

- 메시지 Provider 아키텍처 설계
- ERD 검토
- API 구조 분석

을 진행하였습니다.
