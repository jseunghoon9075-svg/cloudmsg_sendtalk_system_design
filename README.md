# Message Provider System Design

CloudMsg 단일 메시지 시스템을  
멀티 Provider 구조로 확장하기 위한 서버 설계 문서입니다.

## Part A 서버 설계

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
   ├── organizations
   ├── organization_message_provider
   ├── cloudmsg_config
   └── sendtalk_config
   │
   ▼
Provider Router
   │
   ├── CloudMsg Provider → CloudMsg API
   └── SendTalk Provider → SendTalk API
   │
   ▼
message_log 저장
```

---

# 3. Database Design

![erd](erd.png)

주요 테이블

organizations  
→ 고객사(테넌트) 정보

organization_message_provider  
→ 조직이 사용하는 메시지 Provider 설정

cloudmsg_config  
→ CloudMsg API 인증 정보

sendtalk_config  
→ SendTalk API 인증 정보

message_log  
→ 메시지 발송 이력

provider_audit  
→ Provider 변경 이력

---

# 4. Message Flow

1. 메시지 발송 요청
2. organization_message_provider 조회
3. Provider 선택
4. Provider API 호출
5. message_log 저장
   (발송 provider, 외부 메시지 ID, 상태 기록)

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

### 중복 메시지

대응

- idempotency_key 도입

---

# 7. AI Usage

본 과제 수행 과정에서 ChatGPT를 활용하여

- 메시지 Provider 아키텍처 설계
- ERD 검토
- API 구조 분석

을 진행하였습니다.

---

## Part B. 화면 설계

# 1. 목표

조직 관리자가 자신의 조직에서 사용할 메시지 전송 Provider를 설정하고 관리할 수 있는 UI를 설계한다.

지원되는 Provider

- CloudMsg

- SendTalk

UI 설계 목표

- Provider 설정을 직관적으로 관리

- API 설정 오류 방지

- 운영자의 실수 방지

- Provider 변경 이력 관리

---

# 2. 화면 흐름 (User Flow)

관리자는 다음 흐름으로 Provider를 설정합니다.

Organization Admin
      │
      ▼
Settings 메뉴 진입
      │
      ▼
Message Provider 설정 화면
      │
      ▼
현재 Provider 확인
      │
      ▼
Provider 선택
      │
      ▼
Provider 설정 입력
      │
      ▼
API 연결 테스트
      │
      ▼
  설정 저장

---
  
# 3. Provider Settings UI

```json
┌──────────────────────────────────────────────┐
│ Message Provider Settings                    │
├──────────────────────────────────────────────┤

Current Provider
[ CloudMsg ]

------------------------------------------------

Select Provider

(●) CloudMsg
( ) SendTalk

------------------------------------------------

Provider Configuration

Access Key        [ ********************* ]
Secret Key        [ ********************* ]
Sender Phone      [ 01012345678 ]
Kakao Profile ID  [ my-kakao-channel ]

[ Test Connection ]

------------------------------------------------

[ Cancel ]                     [ Save Settings ]

└──────────────────────────────────────────────┘
```

---

# 4. Provider 설정 방식

Provider마다 필요한 설정 값이 다르기 때문에
Provider 선택에 따라 입력 폼이 동적으로 변경된다.

CloudMsg 선택 시

입력 필드

- Access Key

- Secret Key

- Sender Phone

- Kakao Profile ID

---

SendTalk 선택 시

입력 필드

- API Key

- API Secret

- Sender ID

---

# 5. UI 설계 결정 사항
Provider 선택 방식

대안

- Dropdown

- Radio Button

선택

Radio Button

이유

- Provider 수가 적음

- 현재 선택 상태를 명확히 표시

- 운영자 실수 방지

---

API 연결 검증

대안

1. 저장 후 검증

2. 저장 전 검증

선택

Test Connection 버튼 제공

이유

- 잘못된 API Key 저장 방지

- 운영자의 설정 오류 감소

---

Provider 변경 확인

Provider 변경은 운영에 영향을 줄 수 있으므로
확인 팝업을 제공한다.

You are about to change the message provider.

Existing messages will continue using the previous provider.
New messages will use the selected provider.

Continue?

[ Cancel ] [ Confirm ]

---

# 6. 예외 / 엣지 케이스
잘못된 API Key 입력

상황

관리자가 잘못된 API 인증 정보를 입력

대응

- Test Connection 실패

- 오류 메시지 표시

예

```text
Connection failed
Invalid API credentials
```

---

필수 입력 누락

상황

API Key 또는 Secret Key 미입력

대응

입력 검증

```text
API Key is required
```

---

Provider 변경 중 메시지 발송

상황

Provider 변경 시 기존 메시지 발송 중

대응

- 기존 메시지는 기존 Provider 유지

- 신규 메시지부터 변경된 Provider 사용

---

Provider API 장애

상황

Provider API가 응답하지 않는 경우

대응

- 연결 테스트 실패

- 사용자에게 오류 메시지 표시

```text
Provider connection failed
Please try again later
```

---

# 7. 설정 변경 이력 관리

Provider 설정이 변경될 경우
다음 정보가 기록된다.

```text
organization_message_provider_audit
```

기록 정보

- organization_id

- before_provider

- after_provider

- changed_by

- changed_at

이를 통해 운영자가 설정 변경 이력을 확인할 수 있다.

---

# 8. 사용자 피드백

설정 저장 성공

```text
Provider settings updated successfully
```

API 연결 실패

```text
Connection failed.
Please verify API credentials.
```

---

# 9. 기대 효과

이 UI 설계를 통해

- 조직별 메시지 Provider 관리 가능

- API 설정 오류 방지

- 운영 실수 최소화

- Provider 변경 이력 추적 가능


---


# AI 활용

본 과제 수행 과정에서 화면 설계 및 예외 상황을 검토하기 위해 AI를 활용하였다.
AI는 UI 설계 방향과 예외 상황을 탐색하는 참고 도구로 사용되었으며, 최종 설계 결정은 요구사항 분석을 기반으로 직접 판단하였다.

## 1. Provider 설정 화면 구조 설계
사용 프롬프트

```text
B2B SaaS 서비스에서 조직 관리자가 메시지 전송 provider를 설정하는
관리자 UI를 설계하려고 합니다.

지원 provider는 CloudMsg와 SendTalk 두 가지이며
API key, sender 정보 등 provider별 설정이 필요합니다.

운영자가 실수할 가능성을 고려하여
provider 설정 화면의 UI 구조와 주요 구성 요소를 제안해주세요.
```

### AI 응답 활용

AI는 다음과 같은 UI 구성 요소를 제안하였다.

- 현재 Provider 표시

- Provider 선택 영역

- Provider 설정 입력 영역

- 연결 테스트 기능

- 설정 저장 기능

이를 참고하여 Provider 설정 화면을 다음과 같은 구조로 설계하였다.

- Current Provider 표시

- Provider 선택 (Radio Button)

- Provider 설정 입력

- Test Connection

- Save Settings

---

## 2. Provider 설정 UI 방식 결정
사용 프롬프트

```text
관리자 설정 화면에서 메시지 provider를 선택할 때
Dropdown과 Radio Button 중 어떤 UI 방식이 더 적절한지
장단점을 비교해주세요.

Provider는 CloudMsg와 SendTalk 두 가지입니다.
```

AI 응답 활용

AI는 다음과 같은 의견을 제시하였다.

- Provider 수가 적은 경우 Radio Button이 가시성이 높음

- 현재 선택 상태를 명확하게 표시 가능

- 운영 UI에서 실수 방지에 유리

이를 참고하여 Provider 선택 UI를 Radio Button 방식으로 설계하였다.

---

## 3. 예외 상황 검토
사용 프롬프트

```text
메시지 provider 설정을 변경하는 관리자 UI에서
발생할 수 있는 운영 실수나 예외 상황에는 어떤 것들이 있을까요?

예를 들어 API key 오류, provider 변경 중 메시지 발송 등
운영 환경에서 발생 가능한 문제를 정리해주세요.
```

AI 응답 활용

AI는 다음과 같은 예외 상황을 제안하였다.

- 잘못된 API Key 입력

- 필수 설정 값 누락

- Provider 변경 중 메시지 발송

- Provider API 장애

이를 참고하여 다음 대응 방안을 UI 설계에 반영하였다.

- API 연결 테스트 기능

- 필수 입력 값 validation

- Provider 변경 confirmation

- 오류 메시지 표시

---

AI는 UI 설계 아이디어와 예외 상황 탐색을 위해 활용되었으며
최종 설계는 요구사항 분석을 기반으로 직접 결정하였다.
