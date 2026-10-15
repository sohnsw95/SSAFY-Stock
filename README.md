# 싸피증권 · SSAFY-Stock

### 실시간 시세로 경험하는 주식 모의투자

실시간 시세와 주문·체결, 투자 커뮤니티, AI 자동매매를 제공하는 모의투자 서비스입니다.<br/>
KIS API의 실제 시세를 활용하며, 주문은 서비스 내부의 모의 계좌에서 처리합니다.

[포트폴리오](https://sukwoo-portfolio.vercel.app/) · [개발자 소개](https://github.com/sohnsw95)

![싸피증권 시세 및 주문 화면](assets/stock.png)

<br/>

## 프로젝트 개요

| 항목 | 내용 |
|---|---|
| 과정 | SSAFY 14기 특화 프로젝트 · 금융특화 |
| 공식 프로젝트 기간 | 2026.02.19 ~ 2026.03.30 |
| 팀 구성 | 6명 |
| 내 역할 | 팀장 · 백엔드 · 인프라 · CI/CD |
| 수상 | 특화 프로젝트 우수상 |

## 어떤 서비스를 만들었나요?

사용자가 시세를 살펴보고 직접 투자하거나, AI가 관리하는 모의 계좌의 매매를 경험할 수 있는 서비스입니다.
시세 수신, 주문·체결, 계좌 상태, AI 기능을 각각의 서비스로 나누고 메시지로 연결했습니다.

| 기능 | 설명 |
|---|---|
| 실시간 시세 | KIS API에서 받은 종목 시세·호가를 화면에 전달 |
| 모의 주문·체결 | 주문을 기록하고 매칭 결과를 계좌·포지션에 반영 |
| 자산·랭킹 | 모의 계좌의 보유 자산과 투자 성과 확인 |
| AI 자동매매 | AI가 종목과 매수·매도를 판단해 모의 주문 실행 |
| 커뮤니티 | 투자 관련 정보를 나누는 게시판 |
| 회원·관심종목 | 인증, 투자성향 및 관심종목 관리 |

사용자 직접 매매 계좌와 AI 관리 계좌를 구분했습니다.

<br/>

## 시스템 구조

아래는 주요 서비스의 책임과 이벤트 흐름을 간추린 구조입니다.

```mermaid
flowchart TB
    UI[React 화면] --> Proxy[Nginx · HTTPS]
    Proxy --> Gateway[Spring Cloud Gateway]
    Gateway --> Core[Core · 회원 및 거래 상태]
    Gateway --> Market[Market · 시세]
    Gateway --> AI[AI · 모의 자동매매]
    KIS[KIS 시세 API] --> Market
    Market --> Cache[(Redis)]
    Market <--> Events[Kafka 이벤트]
    Core <--> Events
    Matcher[Matcher · 주문 체결] <--> Events
    AI <--> Events
    Core --> DB[(MySQL)]
    AI --> Vector[(ChromaDB)]
    Market -->|WebSocket| UI
    Core -->|SSE 알림| UI
```

| 영역 | 프로젝트 기술 |
|---|---|
| 화면 | React |
| 서버 | Spring Boot · Spring Cloud Gateway · Go · FastAPI |
| 메시징·실시간 통신 | Kafka · Redis · WebSocket · SSE |
| 데이터·외부 연동 | MySQL · ChromaDB · S3 · KIS Open API |
| 배포 | Docker Compose · Jenkins · GitLab Webhook · Nginx · AWS |
| 상태 점검 | Prometheus · Grafana · 서비스별 health endpoint |
| 협업 | GitLab · Jira · Notion · Mattermost |

<br/>

## 제가 맡은 부분

| 영역 | 구현·수행 내용 |
|---|---|
| 회원·인증 | 회원가입·투자성향, 로그인·JWT 발급, 인증 경로 조정 |
| Gateway | 인증·라우팅 구성, 요청 경로와 오류 확인 |
| 관심종목 | 등록·조회·삭제 API와 오류 코드 처리 |
| 설계·협업 | DB/ERD와 API·공통 응답·예외 정책 정리, 팀 일정 조율 |
| 배포 | Compose 레이어 분리, Jenkins 기반 변경 서비스 선별 배포 |
| 프록시 | Nginx HTTPS, Jenkins·WebSocket·SSE 경로 구성 |
| 데이터 보존 | AI 서버 ChromaDB 볼륨·마운트 추가 |

체결엔진과 AI 매매 판단·자동 주문 기능은 팀의 구현입니다.
저는 회원·인증 API와 서비스 연동, 배포·운영 구성을 중심으로 참여했습니다.

<br/>

## 배포 구성

### 서비스 변경 범위에 맞춰 배포하기

여러 서버가 함께 동작하는 환경에서 매번 전체 스택을 재시작하는 대신,
변경된 서비스에 맞춰 빌드·배포 범위를 나누었습니다.

- Docker Compose를 `web`, `server`, `infra`, `monitoring`의 4개 레이어로 분리했습니다.
- Git diff를 이용해 변경된 서비스를 식별하고 Jenkins에서 선별 배포하도록 구성했습니다.
- Push·MR 이벤트에서 빌드·테스트를 실행하고, 특화 당시 `dev` 브랜치에서 배포하도록 구성했습니다.
- Jenkins Credentials로 환경 파일을 주입했습니다.

```mermaid
flowchart LR
    Git[GitLab Push · MR] --> Jenkins[Jenkins]
    Jenkins --> Check[변경 범위 확인]
    Check --> Build[대상 서비스 빌드·테스트]
    Build --> Image[Docker 이미지 빌드]
    Image --> Deploy[Compose 선택 재배포]
```

이 배포 흐름은 변경 서비스의 재배포 방식이며 무중단 배포를 의미하지 않습니다.

## 대표 문제 해결

### Jenkins 하위 경로와 Nginx 프록시 경로 맞추기

**문제**<br/>
Jenkins를 Nginx의 하위 경로로 제공하는 과정에서 crumb 403 오류가 발생했습니다.

**수정**

- Jenkins 실행 prefix를 `/jenkins`로 설정했습니다.
- Nginx의 `/jenkins/` 요청을 같은 경로로 전달하도록 맞췄습니다.
- 불필요한 cookie/path 재작성을 제거했습니다.
- `/jenkins` 요청을 `/jenkins/`로 연결하는 리다이렉트를 추가했습니다.

애플리케이션의 기준 경로와 프록시가 전달하는 경로를 함께 확인해야 한다는 점을 배웠습니다.

<br/>

## Theseus와의 후속 연동

특화 프로젝트 종료 후, Theseus의 Remote Workspace에서 싸피증권 EC2를 연결해 후속 개발과 운영에 활용했습니다.

| 비공개 코드 브랜치 | 용도 |
|---|---|
| `main` · `dev` | 특화 프로젝트 당시 개발 이력 |
| `stock_dev` | Theseus 연동 후속 개발 이력 |
| `stock-prod` | 후속 배포 관련 이력 |

후속 작업 중에는 마켓 서버가 실행 중인데도 시세 화면이 갱신되지 않는 상황을 다뤘습니다.
수신 로그와 Redis·Kafka·WebSocket 구간을 나눠 점검하고, 마켓 서버 재시작 후 시세 수신과 화면 갱신을 확인했습니다.
이 사례는 특화 기간의 성과와 구분합니다.

[연동 플랫폼: Theseus](https://github.com/sohnsw95/Theseus)

## 회고

실시간 서비스에서는 개별 API가 동작하는 것만큼 서비스 사이의 경로와 이벤트 흐름을 맞추는 일이 중요했습니다.
배포 대상을 나누고 인증·프록시 설정을 점검하면서, 기능 개발과 운영 환경을 함께 이해하는 경험을 쌓았습니다.

---

이 저장소에는 프로젝트 소개와 공개용 자료를 정리했습니다.
