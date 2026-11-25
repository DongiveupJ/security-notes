---
layout: post
title: "IDS/IPS 정리와 Suricata 실습"
date: 2026-11-26 00:15:00 +0900
category: technical-writing
description: IDS와 IPS의 정의·배치·탐지 방식·유형·장단점을 NIST 기준으로 정리하고, Suricata로 같은 룰을 탐지(alert)와 차단(drop)으로 바꿔 차이를 직접 확인한다.
---

## 0. 한눈에 보기: IDS vs IPS

본문을 읽기 전에 두 방식의 차이를 먼저 요약했다. 자세한 내용은 2절(정의와 배치), 6절(장단점), 8절(실습)에서 다룬다.

| 구분 | IDS | IPS |
| --- | --- | --- |
| 배치 | 보통 패시브. SPAN/TAP 등으로 트래픽의 복사본을 받는다. | 인라인. 트래픽 경로 한가운데에 놓인다. |
| 동작 | 탐지하고 경보를 남긴다. 트래픽은 그대로 통과한다. (8절: `alert.action`이 `allowed`, 서버 응답을 받음) | 탐지에 더해 패킷을 버리는 등의 방법으로 차단한다. (8절: `alert.action`이 `blocked`, 서버 응답 없음) |
| 장점 | 트래픽 복사본만 보므로 장애가 나도 네트워크는 그대로 흐른다. 공격의 출처를 찾을 포렌식 정보를 제공한다. | 공격이 피해로 이어지기 전에 막는다. 다른 위치와 솔루션이 놓친 악성 행위도 막을 수 있다. |
| 단점 | 막을 수 없다(차단 시도는 신뢰하기 어렵다). 부하로 패킷을 놓치면 그 공격을 탐지하지 못할 수 있고, IDS가 무력화되면 공격이 탐지 없이 통과할 수 있다. | 오탐이 정상 트래픽을 막을 수 있다. 트래픽 경로에 홉이 하나 더 생겨 지연이 생기고, fail-closed로 동작하는 장비가 고장 나면 그 구간이 끊긴다(단일 장애점). |

오탐·미탐, 암호화 트래픽, 튜닝 부담 같은 한계는 두 방식이 공통으로 갖는다(6절). 설정 실수로 방화벽이 막았어야 할 트래픽이 통과했을 때 이를 잡아내는 안전망 역할도 둘 다 한다. IPS는 IDS에 '막는 능력'이 더해진 것이라서, 둘의 차이는 막을 수 있느냐와 그 대가(오탐으로 인한 정상 트래픽 차단, 지연, 장애 시 가용성)에 있다.

## 1. 들어가며

요즘 많은 기업들의 채용 공고를 찾아보면서, 인프라 보안/보안 솔루션 운영 직무를 많이 접했다.

그중에서 IDS/IPS 같은 솔루션을 포함하여 운영하는 직무들이 꽤나 보였다.

한 기업의 보안 솔루션 직무에 지원해 과제 테스트를 하며 보안 솔루션에 관한 PT 자료를 만들었는데, 그 경험을 통해 보안 솔루션에 관심을 가지게 되었다.

또, 졸업 작품인 RevoCheck를 만들며 권한을 회수한 뒤에도 접근이 남는지 검증하는 문제를 다뤘다. 막혀야 할 접근이 실제로 막혔는지 확인하는 일이었다.

그러다 반대쪽이 궁금해졌다. 막혔는지 확인하는 일이 아니라, 이상을 감지하고 막는 일이다. 찾아보니 이것이 IDS/IPS의 주된 역할이었다.

그래서 IDS/IPS의 동작 원리, 유형, 장단점, 차이를 정리하고 실습까지 진행하게 됐다.

## 2. IDS와 IPS 정의, 배치 위치

**IDS**(Intrusion Detection System)는 말 그대로 침입을 탐지하는 솔루션이다. 시스템과 네트워크에서 발생하는 이벤트를 모니터링하며, 침입 징후가 있는지 분석하는 소프트웨어다.

**IPS**(Intrusion Prevention System)는 IDS의 탐지 기능에 더해, 탐지한 위협이 피해로 이어지지 않도록 막으려 시도하는 소프트웨어다.

두 제품군은 기능이 거의 같다. 관리자가 IPS의 방지 기능을 꺼서 IPS를 IDS처럼 운영할 수도 있다. 그래서 "IDS와 IPS는 완전히 다른 장비다"라고 하면 곤란하다. IPS는 IDS에 '막는 능력'이 더해진 것이라고 생각하면 좋다.

**IDPS**: IDS와 IPS를 모두 가리키는 약어다.

### 핵심 기능

IDPS는 공통으로 관측한 이벤트를 기록하고, 중요한 이벤트는 보안 담당자에게 경보로 알리며, 요약 보고서를 만든다. 경보는 이메일, SNMP 트랩, syslog 메시지 등으로 전달된다.

### IPS가 위협을 막는 방법

1. 공격 자체를 중단시킨다 (연결 종료, 패킷 차단 등)
2. 보안 환경을 바꾼다 (방화벽 설정 변경 등)
3. 공격의 내용을 바꾼다 (이메일에서 악성 첨부 파일 제거 등)

탐지 방식이나 유형은 뒤에서 다룬다.

### 인라인 vs 패시브(TAP/SPAN) 배치

배치에 대해 이해하려면, 인라인과 패시브부터 정리하고 가는 게 좋겠다.

| 구분 | 인라인(inline) | 패시브(passive) |
| --- | --- | --- |
| 위치 | 트래픽 경로 한가운데 | 트래픽의 복사본을 받는 곳 |
| 특징 | 감시하는 트래픽이 반드시 센서를 통과해야 함 | 실제 트래픽은 센서를 거치지 않음 |
| 차단 | 가능. 주 목적이 트래픽을 막아 공격을 중단시키는 것 | 트래픽을 멈출 믿을 만한 방법이 없음 |
| 장애 시 | 부하가 심해 패킷을 버리면 네트워크 가용성에 지장 | 부하로 패킷을 놓치면 사고를 탐지하지 못할 수 있음 |

한 줄로 정리하면, 인라인은 트래픽이 센서를 직접 통과하는 곳이라서 막을 수 있고, 패시브는 트래픽의 복사본만 보기 때문에 원칙적으로 탐지와 경보를 위한 것이고, 차단을 시도할 수는 있어도 신뢰하기 어렵다.

패시브로 트래픽 복사본을 얻는 방법은 아래와 같고, 이 글에서는 각 방식을 자세히 다루지 않는다.  
스패닝 포트(SPAN)/네트워크 탭(TAP)/IDS 로드 밸런서

### 배치 위치와 선택 기준

인라인 센서는 방화벽 같은 네트워크 보안 장비가 놓이는 자리에 둔다. 더 안전한 쪽에 두면 검사할 트래픽이 줄어들고, 덜 안전한 쪽에 두면 방화벽을 보호하며 방화벽의 부담을 덜어 준다. 다만 차단 능력의 대가로 장애 지점이 될 수 있다.

패시브 센서는 네트워크 경계 같은 핵심 지점과 DMZ 서브넷 같은 핵심 구간을 감시한다. 다만 트래픽 복사본만 보기 때문에, 부하로 패킷을 놓치면 그 공격은 탐지하지 못할 수 있다.

방지 기능(Prevention)을 쓸 계획이면 인라인으로, 아니면 패시브로 두는 것이 일반적이다. 반드시 그래야 하는 것은 아니다.

## 3. 왜 필요한가: 방화벽만으로는 부족한 이유

여기서 방화벽은 전통적 방화벽(패킷 필터링, 상태 기반 방식)을 기준으로 한다. NGFW는 7절에서 다룬다.

#### 전통적 방화벽이 보는 것: IP, 포트, 프로토콜(주로 L3\~L4), 세션 상태

방화벽은 패킷의 출발지·목적지 IP, 포트, 프로토콜을 확인하고, 이를 미리 정한 정책과 비교해 허용하거나 차단한다. 고급 방화벽은 더 깊은 패킷 검사까지 하기도 한다.  

방화벽이 패킷 필터링을 하는 방식은 IP 주소와 포트로 허용 여부를 정한다. 상태 기반 방식은 이미 열린 연결을 추적해 그 연결에 속한 트래픽인지 판단한다.  

그래서 방화벽의 메인 역할은 "누가 어디로 가도 되는가"를 정하는 것이다. 네트워크 경계에서 내부와 외부 사이를 막고, 허용된 통신만 통과시키는 장치다.  

#### 못 보는 것: 허용된 포트(80/443)로 들어오는 악성 페이로드

전통 방화벽은 대체로 패킷 헤더를 기준으로 판단하지만, IPS는 페이로드(데이터 부분)까지 살펴 위협을 찾아낸다.  
웹 포트 같은 허용된 통로 안으로 들어오는 악성 내용(예: SQL 인젝션 등)이 막아야 할 위협에 해당된다. 웹 공격은 L7에서 동작하는 WAF가 다루는 영역과도 겹친다.  

방화벽은 출입증 검사, IDS는 CCTV와 경보, IPS는 보안요원이라고 생각하면 좋다.  
정리하면 방화벽은 규칙에 따라 통신을 허용하거나 막고, IDS/IPS는 허용된 통신의 내용까지 살핀다.  

배치 위치도 다르다. 방화벽이 외부와 내부 네트워크 사이의 경계(출입문)에 있다면, IPS는 인라인(트래픽 경로)에 직접 놓이고, 방화벽의 안쪽과 바깥쪽 모두에 둘 수 있다(2절 참고). 방화벽 뒤에 두면 출입증 검사를 통과한 사람을 보안요원이 한 번 더 보는 구조가 된다.  

중요한 점은, IDS/IPS가 방화벽을 대체하는 존재가 아니라는 것이다. 이 둘은 방화벽과 협업하는 관계이다. 역할이 다른 기술을 공격 경로마다 겹쳐 놓아, 한 기술이 놓친 공격을 다른 기술이 잡게 하는 방식을 심층 방어(Defense in Depth)라고 한다(NIST 용어집, CNSSI 4009 인용). 방화벽과 IDS/IPS는 서로 다른 지점에서 트래픽을 살펴 이 구조를 이룬다.  

또 설정 실수로 방화벽이 막았어야 할 트래픽이 통과하더라도, IDS/IPS가 이를 감지해 경보를 낼 수 있다. 방화벽의 빈틈을 확인하는 안전망 역할도 하는 셈이다.

## 4. 탐지 방식 (동작 원리)

IDPS가 침입을 탐지하는 판단 기준의 종류로는 3가지가 있다. (NIST 기준)

#### 1. 시그니처 기반

알려진 위협에 대응하는 패턴(시그니처라고 한다. 예: 알려진 악성 코드)과 관측된 이벤트를 비교한다. 예를 들어, 악성 파일 이름(freepics.exe)을 시그니처로 삼으면, 이름이 freepics2.exe로 바뀐 변형은 탐지하지 못한다.  
\- 강점: 알려진 위협을 탐지하는 데 매우 효과적이다.  
\- 약점: 모르는 위협과 변형에 약하다. 여러 이벤트에 걸친 공격은 이어서 보지 못한다.  
8절에서 직접 써본다.  

#### 2. 이상 행위 기반

정상 활동의 프로파일과 관측값을 비교한다. 프로파일은 보통 며칠\~몇 주의 기간 동안의 활동을 분석해 평소 활동(정상 활동)으로 만든다.  
예를 들어, 평소의 웹 트래픽, 프로그램이 사용하는 도구나 호출하는 명령 등을 정상 활동의 기준(프로파일)으로 잡아둔다.  
정적 프로파일은 다시 만들기 전까지 고정적으로 사용되고, 동적 프로파일은 이벤트가 관측될 때마다 계속 조정된다.  
\- 강점: 이전에 알려지지 않은 위협도 탐지할 수 있다.  
\- 약점: 악의적인 활동이 프로파일에 흡수될 수 있고, 정적 프로파일은 새로운 활동에 대비하기 어려우며, 오탐이 흔하다. 경보가 나면 이유를 바로 알기 어렵다.  

#### 3. 상태 기반 프로토콜 분석

프로토콜 상태마다 허용되는 동작의 프로파일과 관측된 활동을 비교한다.  
이상 행위 기반은 네트워크나 호스트별 프로파일을 쓰지만, 상태 기반은 벤더가 만든 범용 프로파일을 쓴다.  
예를 들어, 로그인(인증)하지 않은 FTP 세션에서 인증 후에만 허용되는 명령이 들어오면, 프로토콜이 그 상태에서 허용하지 않는 동작이므로 이상으로 판단한다.  
\- 강점: 프로토콜 상태를 추적해, 규격에서 벗어난 동작을 탐지한다.  
\- 약점: 자원을 많이 쓰고, 프로토콜이 허용하는 범위 안에서 이뤄지는 공격은 탐지할 수 없다.

세 방식을 한눈에 비교하면 다음과 같다.

| 구분 | 시그니처 기반 | 이상 행위 기반 | 상태 기반 프로토콜 분석 |
| --- | --- | --- | --- |
| 비교 대상 | 알려진 위협의 패턴(시그니처) | 정상 활동의 프로파일(해당 네트워크·호스트 기준) | 프로토콜 상태별로 허용되는 동작의 프로파일(벤더 범용) |
| 강점 | 알려진 위협 탐지에 매우 효과적 | 이전에 알려지지 않은 위협도 탐지 가능 | 프로토콜 상태를 추적해, 그 상태에서 허용되지 않는 동작을 잡아냄 |
| 약점 | 모르는 위협·변형에 약하고, 여러 이벤트에 걸친 공격은 이어서 보지 못함 | 악의적 활동이 프로파일에 흡수될 수 있고, 오탐이 흔하며 경보 이유를 바로 알기 어려움 | 자원을 많이 쓰고, 프로토콜이 허용하는 범위 안의 공격은 탐지하지 못함 |
| 예시 | 악성 파일 이름이 freepics.exe에서 freepics2.exe로 바뀌면 이름 기반 시그니처를 피함 | 평소의 웹 트래픽, 프로그램이 호출하는 명령을 정상 기준으로 삼음 | 인증하지 않은 FTP 세션에서 인증 후에만 허용되는 명령이 들어옴 |

실제 제품은 이 방식 중 하나만 쓰기보다, 여러 방식을 따로 또는 통합해서 쓰는 경우가 대부분이다. 한 방식이 놓친 약점을 다른 방식이 보완하기 때문이다.

## 5. 유형

NIST는 무엇을 감시하느냐로 네트워크 기반, 호스트 기반, 무선, 네트워크 행위 분석(NBA)의 네 유형을 나눈다.

인라인/패시브는 배치 방식이라서 IDPS 유형과는 별개의 축이다. 5절의 유형은 감시 대상, 배치는 센서 위치를 분류한 것이다.  
NIDS/HIDS의 2분류가 흔하지만, 이 글은 NIST 4분류를 따른다.

#### 1. 네트워크 기반 (NIDS)

\- 네트워크 트래픽을 감시한다. 애플리케이션(메인), 전송/네트워크 계층을 분석한다.  
\- 센서, 관리 서버, 콘솔로 구성되어 있고, 데이터베이스 서버는 선택적으로 구성할 수 있다. 센서는 인라인과 패시브로 나뉜다(2절 참고).  
\- 탐지 대상: 정찰과 공격, 예상치 못한 응용 서비스, 정책 위반. 4절의 세 방식을 조합해 탐지한다.  
\- 한계: 암호화된 트래픽 안의 공격은 탐지하지 못한다(HTTPS). 트래픽이 많은 고부하에서는 전체 분석이 어려울 수 있고, DDoS나 경보 폭주를 노린 트래픽으로 센서가 공격받을 수 있다.

#### 2. 호스트 기반 (HIDS)

\- 단일 호스트를 감시한다. 시스템 로그, 실행 중인 프로세스, 파일 접근과 변경 등을 본다.  
\- 호스트에 설치하는 에이전트가 대부분이고, 일부는 어플라이언스를 쓴다.  
\- 탐지 대상: 호스트 내부의 파일·프로세스·설정 변경 등  
\- NIDS는 네트워크 구간에서 암호문 상태의 트래픽을 보지만, HIDS는 복호화가 끝난 호스트 안에서 활동을 보기 때문에 암호화의 영향을 덜 받는다.  
\- 한계: 호스트 자원을 많이 쓰고, 튜닝 부담이 크며, 알림이 15\~60분 단위 배치로 전달될 수 있다.

#### 3. 무선 (WIDS)

\- WLAN 프로토콜 트래픽을 감시한다. IP 주소나 애플리케이션 계층 내용은 보지 않아 범위가 좁고, 그만큼 정확도는 높은 편이다.  
\- 탐지 대상: 비인가 WLAN, 장치, 무선 스캐너, DoS, 위장/중간자 공격 등  
\- 한계: 센서가 한 번에 한 채널만 볼 수 있어 채널을 돌아가며 감시한다.&#32;

#### 4. 네트워크 행위 분석 (NBA)

\- 네트워크 흐름과 트래픽 통계를 감시한다. 소수 구간의 패킷을 직접 보거나, 라우터의 플로우 데이터를 받는다.  
\- 주로 이상 기반이고, 시그니처 기반 기능은 거의 없다(4절 참고).  
\- 탐지 대상: DoS/DDoS, 스캔, 웜 등  
\- 한계: 플로우 데이터가 배치로 전달되어 지연되고, 소규모 공격 탐지는 덜 정확하다. 오탐 때문에 차단은 제한적으로만 쓰거나 아예 쓰지 않는다.

NIST의 네 유형을 한눈에 비교하면 다음과 같다.

| 구분 | 네트워크 기반(NIDS) | 호스트 기반(HIDS) | 무선(WIDS) | 네트워크 행위 분석(NBA) |
| --- | --- | --- | --- | --- |
| 감시 대상 | 특정 구간·장비의 네트워크 트래픽(주로 응용 계층 분석) | 단일 호스트의 시스템 로그, 실행 중인 프로세스, 파일 접근·변경 | WLAN(IEEE 802.11 계열) 트래픽 | 네트워크 흐름과 트래픽 통계 |
| 센서·배치 | 센서(인라인 또는 패시브), 관리 서버, 콘솔 | 호스트에 설치하는 에이전트(일부는 어플라이언스) | 전용 센서, AP·무선 스위치 내장 센서, 노트북 등 호스트 기반 센서 | 소수 구간의 패킷을 직접 보거나 라우터의 플로우 데이터를 수집 |
| 주요 탐지 대상 | 정찰·공격, 예상치 못한 응용 서비스, 정책 위반 | 호스트 내부의 파일·프로세스·설정 변경 등 | 비인가 WLAN·장치, 무선 스캐너, DoS, 위장·중간자 공격 | DoS/DDoS, 스캔, 웜 |
| 탐지 방식 | 시그니처, 이상 행위, 상태 기반 프로토콜 분석을 조합 | 코드 분석, 파일시스템·로그 모니터링 등 | 시그니처, 이상 행위, 상태 기반 프로토콜 분석 기법을 조합 | 주로 이상 기반(시그니처 기능은 거의 없음) |
| 한계 | 암호화된 트래픽 안의 공격은 탐지 불가, 고부하 시 분석 한계, 센서가 DDoS 등의 공격 대상이 될 수 있음 | 호스트 자원 소모, 튜닝 부담, 알림이 15\~60분 단위 배치로 전달될 수 있음 | 한 번에 한 채널만 감시, 일부 무선 공격(오프라인 키 크래킹 등)은 탐지 불가, 센서 재밍·DoS 위험 | 플로우 데이터 전달 지연, 소규모 공격 탐지는 덜 정확, 오탐 때문에 차단은 제한적 |

EDR(Endpoint Detection and Response), 클라우드, 컨테이너 같은 최근 환경의 IDPS 유형까지 다루면 글이 너무 방대해지므로, 후속 학습을 위해 간단한 분류표만 남긴다.

| 환경 | 대표 예 | 감시 대상 | NIST 유형과의 관계(해석) |
| --- | --- | --- | --- |
| 엔드포인트 | EDR | 엔드포인트 활동 텔레메트리 | 호스트 기반 + 조사·대응 기능 |
| 클라우드 네트워크 | Google Cloud IDS | 미러링된 트래픽 | 네트워크 기반(패시브) |
| 클라우드 로그/API | AWS GuardDuty | CloudTrail, 플로우, DNS 로그 | NBA와 비슷하지만 입력이 로그 메타데이터 |
| 컨테이너 | Falco | 커널 syscall + K8s 메타데이터 | 호스트 기반 계열 |

## 6. 장단점과 한계

### 장점

방화벽 같은 다른 보안 기술이 놓친 부분을 보완한다.

또한, 공격의 출처를 찾을 포렌식 정보를 제공하고, 공격자가 자기 행위에 책임을 지게 하는 억제 효과도 어느 정도 있다.

IPS의 차단 기능은 다른 위치와 솔루션이 놓친 악성 행위를 찾아 '막는' 효과까지 있다.

### 단점

#### 1. 오탐과 미탐, 트레이드오프 관계

오탐(False Positive)은 정상 활동을 이상 행위로 잘못 탐지하는 것이다.  
미탐(False Negative)은 이상 활동을 정상 활동으로 착각해 놓치는 것이다.  
이 두 개념이 처음엔 너무 헷갈렸다...  

오탐과 미탐은 한쪽을 줄이면 다른 쪽이 늘어나는 트레이드오프 관계라 모두 없애는 것은 불가능하다.  

많은 조직이 미탐을 줄이는 대신 오탐 증가를 감수한다. 위협을 놓치는 게 더 큰 리스크이기 때문이라고 해석할 수 있다.  
하지만, 그만큼 분석가의 부담이 커진다.  

특히 이상 행위 기반은, 실제로는 정상적인 활동이지만 평소와 다른 활동 때문에 오탐이 많이 나온다. (4절 비교표 참고)  

NIST는 대부분의 IDPS가 탐지 정확도를 높이려면 어느 정도의 튜닝과 맞춤 설정이 필요하고, 임계값 및 허용/차단 목록은 주기적으로 검토해야 한다고 한다.  
설정 없이 쓰면 오래되어 식별하기 쉬운 소수의 공격만 탐지하는 경우도 있다.  

차단을 켜기 전에 학습·시뮬레이션 모드를 쓰면, 오탐으로 정상 활동을 실수로 막을 위험을 줄일 수 있다.  

#### 2. 암호화 트래픽

IDS는 볼 수 있는 것만 분석하므로, 암호화된 연결 안으로 터널링된 트래픽은 분석할 수 없다. (5절의 NIDS 한계 참고)  

암호화 트래픽을 검사하는 방법으로는 다음 4가지가 있다.  
① 서버 자체에서 검사 ② 프록시 서버 ③ IDS가 개인키로 직접 복호화 ④ 독립 복호화 도구  
이 중 ①은 서버 위의 HIDS로 복호화된 트래픽을 보는 방식이고, ②는 프록시가 복호화한 트래픽을 IDS가 검사하는 방식이다. ③은 일부 IDS 장비(예: Juniper IDP)가 개인키로 직접 복호화하는 방식이고, ④는 Viewssld 같은 별도 도구로 복호화하는 방식이다.  

하지만 복호화에는 성능 비용이 따른다. 연산 자원이 많이 들어서, 일부 검사 장비는 복호화를 생략하거나 성능 저하를 감수한다는 것이 복호화 솔루션 벤더(F5)의 설명이다.  

#### 3. 회피

탐지를 피하려고 악성 활동의 형식이나 타이밍을 바꾸는 것이다.  
핵심 원리는 IDS가 패킷을 해석하는 방식과 실제 대상 호스트가 해석하는 방식이 다를 때 틈이 생긴다는 것이다.  
대응책으로는 IDPS가 대상 호스트가 하는 특수 처리를 그대로 따라 하는 것이 있다.

#### 4. 성능 한계, IDS 자체를 표적으로 한 공격

DDoS나 대규모 악성코드 사태 같은 불리한 상황에서 IDPS의 가용성과 성능이 떨어질 수 있다.  
별도 관리망 없이 VLAN으로 관리 트래픽을 기존 네트워크 장비와 공유하면, 그 장비가 완전히 포화되어 IDPS의 가용성과 성능에 영향을 줄 수 있다.  

상태 기반 프로토콜 분석은 복잡하고 상태를 추적해야 해서 자원을 많이 쓴다.  

단계적 배치가 확장성 문제를 미리 파악하는 데 도움이 된다.  
센서를 한꺼번에 켜면 관리 서버와 콘솔이 과부하될 수 있다.  

IDPS 자체가 공격자의 표적이 되는 경우도 많다. 패시브 IDS는 트래픽 복사본만 보기 때문에, 장애가 나도 네트워크는 그대로 흐른다(fail-open, 아래 5번 참고). 이건 장점이면서 약점이다. IDS가 무력화돼도 네트워크는 멈추지 않지만, 그동안 공격도 탐지되지 않은 채 통과한다. 그래서 공격자가 먼저 DoS로 IDS를 무력화한 뒤 공격하면 탐지 없이 통과할 수 있다.

#### 5. 인라인 IPS의 지연과 단일 장애점(SPOF)

인라인 IPS는 트래픽 경로 한가운데에 놓이므로 홉이 하나 더해져 지연이 생긴다.  

장애가 나면 두 가지 중 하나로 동작한다. 트래픽을 검사 없이 통과시키면(fail-open) 네트워크는 유지되지만 그 순간 검사와 차단이 사라지고, 트래픽을 막으면(fail-closed) 보호는 유지되지만 해당 구간이 끊긴다. fail-closed로 동작하는 장비가 고장 나면 그 인라인을 지나는 트래픽이 모두 막히므로 단일 장애점(SPOF)이 된다. 어느 쪽으로 동작할지는 제품과 설정에 따라 다르다.

완화책으로는 전원이 꺼져도 인라인 인터페이스 쌍 사이로 트래픽을 통과시키는 하드웨어 바이패스가 있다. 그래서 바이패스, 이중화, 장애 모드(fail-open/closed)를 설계 단계에서 정해야 한다.

## 7. 활용 방법

IDS/IPS를 단독 장비로 보지 않고, 다른 시스템과 결합해서 사용하는 방식으로 활용할 수 있다.

#### 1. SIEM·SOC와 연계: IDS 경보를 출발점으로

SIEM은 서버, 네트워크 장비, 애플리케이션, 클라우드 서비스의 로그를 모아 상관분석 규칙을 적용하고, 로그 하나로는 보이지 않는 의심 패턴을 찾아낸다.  

SOC는 SIEM, IDS/IPS, EDR 등을 써서 위협을 탐지하고 조사하고 대응하는 팀과 절차다.  
즉 IDS는 SOC의 여러 센서 중 하나이다.  

상관분석의 예시로 로그인 실패 한 번은 잡음이지만, 실패 뒤 성공, 새 프로세스, 외부 연결이 이어지면 공격 사슬이 된다.  
흐름은 다음과 같다. IDS가 탐지하면 이벤트 로그와 경보를 만들고, SIEM이 이를 모아 정규화한 뒤 상관분석해서 SecOps(보안 운영) 팀으로 넘긴다. 정규화(서로 다른 장비의 로그 형식을 통일하는 작업)가 상관분석보다 먼저라는 점이 실무 포인트다.  

여기서 운영상의 문제는 경보의 질이다. 6절의 오탐과 관련이 있는데, 경보만으로는 검증에 필요한 맥락이 부족해서, SOC가 노이즈 많은 IDS 경보를 분류하는 데 병목이 생긴다는 지적이 있다.  

연계는 입력과 출력 두 방향이다. 입력으로는 위협 인텔리전스를 IDS 시그니처와 SIEM 상관 규칙에 함께 넣어 최신 위협을 탐지하고, 출력으로는 IDS 경보를 SOAR에 연결해 격리/차단 같은 대응을 자동화할 수 있다.  

*아래는 궁금한 사람들을 위한 추가 정보다.*  

SOAR(Security Orchestration, Automation and Response)는 보안 도구들을 연결(orchestration)하고 반복 대응 절차를 자동화(automation)해서 사고 대응(response)을 빠르게 하는 플랫폼이다. 미리 정의한 대응 절차를 플레이북이라고 부르고, 경보가 들어오면 이 플레이북이 자동으로 실행된다.  

SIEM은 로그를 모아 상관분석하고 경보를 내는 탐지/가시성 특화이고, SOAR는 그 경보를 받아 조치까지 실행하는 대응 쪽이다.  

#### 2. NGFW 통합: 하나의 장비 vs 독립 IPS

NGFW는 전통적 패킷 필터링에 심층 패킷 검사, IPS, 애플리케이션 인식을 결합한 방화벽이다. 3절의 전통 방화벽(L3\~L4)이 IP·포트·프로토콜 중심이었다면, NGFW는 애플리케이션 수준까지 검사한다는 점이 다르다.  

그렇다고 단순히 IPS 기능이 들어 있다고 모두 NGFW가 아니다. IPS 기능과 방화벽이 긴밀히 통합되어야 NGFW라고 할 수 있다.  

통합의 이점은 연동이다. IPS가 악성 트래픽을 탐지하면 방화벽이 사람 개입 없이 보안 정책을 자동 갱신해 내려보낼 수 있다.   
2절의 IPS가 방화벽 설정을 바꾼다는 방지 방법과 이어진다.  

독립 IPS를 유지하는 조직은 성능과 검사 능력을, NGFW로 옮기는 조직은 관리 간소화를 중시하는 경향이 있다.  
NGFW에서 IPS를 켜면 처리량이 크게 떨어질 때, 차단 품질과 성능이 필수일 때, NetOps(네트워크 운영)와 SecOps의 책임을 분리해야 할 때는, 독립 NGIPS(Next-Generation IPS, IPS에 애플리케이션과 사용자 맥락을 더한 차세대 IPS)가 필요할 수 있다.  
이 비교는 NGFW와 NGIPS를 판매하는 벤더(Fortinet, Cisco)의 설명에 기대고 있어서, 벤더마다 강조점이 다를 수 있다는 점을 유의해야 한다.  

#### 3. 차량 IDS: IT 환경과 무엇이 다른가

**ECU**(Electronic Control Unit)는 엔진, 브레이크, 에어백 등 차량의 각 기능을 제어하는 소형 컴퓨터이고, **CAN**(Controller Area Network)은 ECU들이 서로 메시지를 주고받는 차량 내부 통신 버스이다.  
현대 차량은 많은 ECU가 CAN, 자동차 이더넷 등으로 연결된 구조이다.  
CAN은 보안을 고려해 설계되지 않아 송신자 인증과 암호화가 없다.  
그래서 수신 노드가 메시지의 출처를 확인할 수 없고, 스니핑, 퍼징, 스푸핑, 재전송, DoS에 취약하다.  

CAN에 인증을 넣으려면 ECU의 연산 능력과 통신 용량에 부담이 커서(아래 제약 참고), 탐지가 현실적인 방어책이 된다. 한 논문은 IDS가 CAN 버스를 보호하는 가장 실용적인 도구라고 한다.  

동작 방식: 차량 IDS는 버스의 트래픽을 감시하고 ECU 간 메시지를 분석해 의심스럽거나 이상한 행위를 찾는다.  
보통 ECU에 설치해서 들어오는 메시지를 분석한다.  
탐지는 이상 행위 기반이 중심이고, 한 서베이는 이를 빈도 기반, 머신러닝 기반, 통계 기반, 하이브리드로 나눈다.  

핵심은 IT와 다른 제약이다. 전통 네트워크용 IDS 기술은 자원 제약과 자동차의 실시간 요구 때문에 차량에 그대로 적용할 수 없다.  
ECU는 연산 능력이 제한적이고 CAN의 전송 용량도 한계가 있으며, 구형 차량에 대규모 보안 업데이트를 하기도 어렵다.  
일부 연구는 탐지가 자원을 많이 쓰므로 연산 능력이 있는 안드로이드 헤드유닛에서 수행하는 구성을 제안한다.  

감시 대상은 버스 메시지라서 네트워크 기반에 가깝고, ECU에 올라간다는 점은 호스트 기반에 가깝다고 해석할 수 있다.

## 8. 실습: Suricata

### 실습 환경

실습은 Apple Silicon 맥북 위에 Multipass로 띄운 Ubuntu VM에서 진행했다. Suricata의 인라인 IPS 설정 문서는 Linux와 Windows용만 있고, 이 글은 Linux의 NFQUEUE 방식으로 차단을 실습하기 때문에 맥북에 바로 설치하지 않고 Linux VM을 썼다.

| 구분 | 내용 |
| --- | --- |
| 호스트 | Apple Silicon 맥북 |
| 가상화 | Multipass (Ubuntu VM, RAM 4GB, CPU 2개) |
| 게스트 OS | Ubuntu 26.04.1 LTS (resolute), 커널 7.0.0-43-generic |
| 아키텍처 | aarch64 (ARM64) |
| Suricata | 8.0.8 RELEASE (Ubuntu 패키지 1:8.0.8-0ubuntu3) |
| 캡처·차단 지원 | AF\_PACKET, NFQueue 지원 |
| 경로 | 설정 /etc/suricata/, 로그 /var/log/suricata/ |

지원 여부는 suricata --build-info에서 NFQueue support가 yes로 나오는 것을 확인했다.

### 설치와 설정

Suricata는 apt로 설치했다. 설정 파일은 `/etc/suricata/suricata.yaml`이고, `af-packet` 항목의 `interface`가 캡처할 인터페이스를 정한다. 기본값은 `eth0`라서, VM의 실제 인터페이스 이름과 맞는지 확인해야 한다.

```yaml
af-packet:
  - interface: eth0
```

### 룰 받기

`suricata-update`로 ET Open 룰셋을 받았다. 로그에는 룰 69,581개를 불러와 53,604개를 활성화했다고 나왔다.

```bash
sudo suricata-update
```

### 문제 1: 룰 업데이트 중 검증 실패

처음 `suricata-update`를 실행했을 때 룰은 정상적으로 받았는데, 마지막에 아래 에러가 나왔다.

```
<Error> -- Suricata test failed, aborting.
<Error> -- Restoring previous rules.
```

`suricata-update`는 새 룰을 적용하기 전에 `suricata -T`로 검증한다. 같은 검증을 직접 실행하고 `dmesg`를 확인하니, 룰 53,604개를 불러온 뒤 메모리 부족으로 Suricata가 강제 종료된 기록이 세 번 남아 있었다. VM의 총 메모리는 943MiB에 스왑도 없었다.

VM을 중지하고 맥에서 메모리를 4GB, CPU를 2개로 늘렸다.

```bash
multipass stop ids
multipass set local.ids.memory=4G
multipass set local.ids.cpus=2
multipass start ids
```

`free -h`로 총 메모리가 3.8GiB로 늘어난 것을 확인했고, 이후 `suricata-update`는 에러 없이 끝났다.

### 문제 2: 인터페이스 이름이 바뀌었다

VM 설정을 바꾸고 다시 시작하자 Suricata 로그에 아래 에러가 나왔다.

```
af-packet: enp0s1: failed to find interface: No such device
```

처음 `ip addr`에서 확인한 인터페이스 이름은 `enp0s1`이었는데, 다시 확인하니 `eth0`였다(IP 192.168.252.2/24는 그대로). 설정 파일의 `interface`를 `eth0`로 맞추고 재시작했다. 이름이 바뀐 이유는 확인하지 못했다. VM 설정을 바꾼 뒤에는 인터페이스 이름부터 다시 확인하는 게 좋겠다.

### 엔진 시작 확인

재시작 후 `suricata.log`는 이렇게 끝났다(발췌).

```
detect: 53604 rules successfully loaded, 0 rules failed, 0 rules skipped
runmodes: eth0: creating 2 threads
threads: Threads created -> W: 2 FM: 1 FR: 1   Engine started.
```

### 공식 테스트가 동작하지 않았다

퀵스타트는 `fast.log`를 `tail -f`로 띄워 놓고 아래 명령을 실행하면 SID 2100498 경보가 나온다고 안내한다.

```bash
curl http://testmynids.org/uid/index.html
```

그런데 이 주소는 "This domain may be for sale"이라는 도메인 판매 페이지를 돌려줬고, `fast.log`에는 아무것도 찍히지 않았다. 이 경보는 응답에 root의 `id` 명령 결과가 들어 있을 때 뜨는 것이라, 응답이 달라져서 경보가 나오지 않은 것으로 보인다. 제3자가 관리하는 주소이므로 테스트 대상으로 계속 쓰지 않기로 했다.

### 직접 룰 만들어 탐지하기

공식 테스트가 안 되어서 직접 룰을 만들었다. HTTP 요청의 URI에 `ids-test`가 들어 있으면 경보를 내는 룰이고, 공식 문서의 예시 룰과 같은 구조(액션, 헤더, 옵션)를 따랐다.

```
alert http $HOME_NET any -> $EXTERNAL_NET any (msg:"LAB HTTP URI contains ids-test"; flow:established,to_server; http.uri; content:"ids-test"; sid:1000001; rev:1;)
```

- `alert`: 매치되면 경보만 낸다.
- `http $HOME_NET any -> $EXTERNAL_NET any`: 내부(VM)에서 외부로 나가는 HTTP 트래픽을 본다.
- `http.uri; content:"ids-test"`: 요청 URI에 이 문자열이 있는지 확인한다.
- `sid`는 룰의 고유 번호이고, `rev`는 룰의 버전이다.

이 룰은 ET 룰셋 파일(`suricata.rules`)과 따로 `/var/lib/suricata/rules/local.rules`에 만들고, `suricata.yaml`의 `rule-files`에 추가했다.

```yaml
rule-files:
  - suricata.rules
  - local.rules
```

`suricata -T`로 설정을 검증하니 룰 파일이 2개, 룰이 53,605개로 늘었다(기존 53,604개).

터미널 하나에서 `fast.log`를 `tail -f`로 띄워 놓고, 다른 터미널에서 요청을 보냈다.

```bash
curl http://example.com/ids-test
```

`fast.log`에 경보가 바로 찍혔다.

![두 터미널 캡처: 왼쪽은 fast.log 감시, 오른쪽은 curl 요청]({{ '/assets/img/posts/ids-ips/suricata-ids-detection.png' | relative_url }})

*왼쪽: `fast.log`를 `tail -f`로 감시 중인 터미널, 오른쪽: `curl`로 요청을 보낸 터미널*

`Classification: (null)`은 룰에 `classtype`을 넣지 않아서 나온 값이다. 이 룰은 요청 URI를 보기 때문에, 응답 내용이 달라도 경보가 난다. 앞의 공식 테스트가 응답 내용에 의존했던 것과 다르다.

같은 경보를 `eve.json`에서도 확인했다. 아래 명령의 출력에서 핵심 필드만 발췌했다.

```bash
sudo tail -n 100 /var/log/suricata/eve.json | jq 'select(.event_type=="alert")'
```

```json
"alert": {
  "action": "allowed",
  "signature_id": 1000001,
  "rev": 1,
  "signature": "LAB HTTP URI contains ids-test",
  "severity": 3
},
"http": {
  "hostname": "example.com",
  "url": "/ids-test",
  "http_method": "GET",
  "status": 404
}
```

여기서 눈여겨볼 필드는 `alert.action`이 `allowed`라는 점이다. 경보는 났지만 트래픽은 막히지 않았다. `http.status`가 404인 것도 요청이 서버까지 갔고 응답이 돌아왔다는 뜻이다. IDS 모드는 탐지만 한다. 이 값은 다음 단계에서 IPS로 차단했을 때 어떻게 달라지는지 비교하는 기준이 된다.

### IPS 모드로 차단하기

같은 룰의 액션을 `alert`에서 `drop`으로 바꾸고 `rev`를 2로 올렸다. `drop`은 패킷을 버리고 경보를 생성하는 액션이다.

```
drop http $HOME_NET any -> $EXTERNAL_NET any (msg:"LAB HTTP URI contains ids-test"; flow:established,to_server; http.uri; content:"ids-test"; sid:1000001; rev:2;)
```

차단하려면 Suricata가 트래픽 경로에 인라인으로 놓여야 한다. 이번에는 Linux의 NFQUEUE 방식을 썼다. 먼저 IDS 서비스를 멈추고, `-q 0`으로 Suricata를 인라인 모드로 실행했다.

```bash
sudo systemctl stop suricata
sudo suricata -c /etc/suricata/suricata.yaml -q 0
```

그다음 포트 80 트래픽만 큐로 보내도록 iptables 규칙을 추가했다. 포트를 80으로 제한한 것은 SSH 접속에 영향을 주지 않기 위해서다. Suricata가 없는 상태에서 큐 규칙만 남으면 해당 트래픽이 막히기 때문에, Suricata를 먼저 켜고 규칙을 나중에 넣었다.

```bash
sudo iptables -I OUTPUT -p tcp --dport 80 -j NFQUEUE
sudo iptables -I INPUT -p tcp --sport 80 -j NFQUEUE
```

이 상태에서 같은 요청을 다시 보냈다.

```bash
curl -4 --max-time 10 http://example.com/ids-test
```

`fast.log`에는 `[Drop]`이 붙은 경보가 찍혔다.

```
11/25/2026-18:54:38.313967  [Drop] [**] [1:1000001:2] LAB HTTP URI contains ids-test [**] [Classification: (null)] [Priority: 3] {TCP} 192.168.252.2:41630 -> 104.20.23.154:80
```

`eve.json`의 같은 경보는 `alert.action`이 `blocked`로 바뀌었다.

```json
"alert": {
  "action": "blocked",
  "signature_id": 1000001,
  "rev": 2,
  "signature": "LAB HTTP URI contains ids-test",
  "severity": 3
},
"http": {
  "hostname": "example.com",
  "url": "/ids-test",
  "http_method": "GET",
  "length": 0
},
"flow": {
  "pkts_toserver": 3,
  "pkts_toclient": 1,
  "bytes_toserver": 247,
  "bytes_toclient": 60
}
```

IDS 모드에서 남은 경보와 비교하면 차이가 분명하다.

| 구분 | IDS 모드 (alert) | IPS 모드 (drop) |
| --- | --- | --- |
| 룰 액션 | alert | drop |
| fast.log | 경보만 기록 | `[Drop]` 표시 |
| alert.action | allowed | blocked |
| 서버 응답 | HTTP 404 응답을 받음 | 응답 없음 |
| 서버에서 온 패킷 | 4개, 1130바이트 | 1개, 60바이트 |

두 경보에서 목적지 IP가 다른데(172.66.147.243, 104.20.23.154), example.com이 가리키는 주소가 요청마다 다를 수 있어서 생긴 차이로 보인다.

### 8절 참고 자료

- [Suricata Docs, Rules Format](https://docs.suricata.io/en/latest/rules/intro.html)
- [Suricata Docs, Meta Keywords (sid, rev)](https://docs.suricata.io/en/latest/rules/meta.html)
- [Suricata Docs, Quickstart guide](https://docs.suricata.io/en/latest/quickstart.html)
- [Suricata Docs, Setting up IPS/inline for Linux](https://docs.suricata.io/en/latest/setting-up-ipsinline-for-linux.html)
- [Suricata Docs, Setting up IPS/inline for Windows](https://docs.suricata.io/en/latest/setting-up-ipsinline-for-windows.html)
- [Suricata Docs, Command Line Options](https://docs.suricata.io/en/latest/command-line-options.html)
- [Suricata Docs, Eve JSON Format](https://docs.suricata.io/en/latest/output/eve/eve-json-format.html)

위 문서는 latest 버전(9.0.0-dev 표기)이고, 실습한 Suricata는 8.0.8이다.  

## 9. 마치며

채용 공고에서 자주 보던 IDS/IPS를 정의부터 실습까지 정리해 봤다.

가장 크게 바뀐 생각은 IDS와 IPS가 완전히 다른 장비가 아니라는 점이다. 탐지하는 방식은 같고, 차이는 트래픽 경로 위에 놓여 실제로 막을 수 있느냐에 있다. 실습에서도 같은 룰의 액션만 `alert`에서 `drop`으로 바꾸고 Suricata를 인라인(NFQUEUE)으로 놓자, 경보의 `alert.action`이 `allowed`에서 `blocked`로 바뀌고 서버 응답이 사라졌다.

막는 능력에는 대가가 따른다는 것도 알게 됐다. 오탐이 정상 트래픽을 막을 수 있고, 인라인 장비는 지연을 만들고 단일 장애점이 될 수 있다. 실습 중에도 Suricata 없이 큐 규칙만 남으면 트래픽이 막히기 때문에, 켜고 끄는 순서에 신경 써야 했다.

문서대로만 되지 않는 부분도 많았다. 룰 5만여 개를 불러오다 메모리 부족으로 엔진이 죽었고, 공식 테스트 주소는 더 이상 기대한 응답을 주지 않았다. IDS/IPS를 운영한다는 것이 장비를 켜 두는 일이 아니라, 룰과 성능과 오탐을 계속 조정하는 일이라는 것을 조금은 체감했다.

다음에는 이번에 다루지 못한 부분을 이어서 보고 싶다. ET 룰셋의 실제 룰을 읽어 보고, Suricata가 멈춘 상태에서 트래픽이 어떻게 되는지(fail-closed) 직접 확인해 볼 생각이다.

## 참고 자료

- [NIST SP 800-94, Guide to Intrusion Detection and Prevention Systems (2007)](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-94.pdf)
- [NIST CSRC Glossary, defense-in-depth](https://csrc.nist.gov/glossary/term/defense_in_depth)
- [Check Point, IPS vs. Firewall: What's the Difference?](https://www.checkpoint.com/kr/cyber-hub/network-security/what-is-firewall/ips-vs-firewall-differences/)
- [Palo Alto Networks, IPS vs. IDS vs. Firewall](https://origin-www.paloaltonetworks.com/cyberpedia/firewall-vs-ids-vs-ips)
- [Ptacek & Newsham, Insertion, Evasion, and Denial of Service: Eluding Network Intrusion Detection (1998)](https://insecure.org/stf/secnet_ids/secnet_ids.html)
- [Decrypting SSL/TLS traffic for hidden threats detection (arXiv:1904.08383)](https://arxiv.org/pdf/1904.08383)
- [F5, Defending Against Encrypted Threats (벤더 자료)](https://www.f5.com/c/app-protect/defending-against-encrypted-threats.html)
- [Cisco, Snort Fail Open vs. Hardware Bypass](https://docs.manage.security.cisco.com/cdfmc/c_failsafe_and_snort_fail_open_vs_hardware_bypass.html)
- [PSU, SIEM, SOC, and Threat Detection (강의 자료)](https://web.cecs.pdx.edu/~dmcgrath/courses/Su26/introsec/siem-and-soc.html)
- [Corelight, NDR vs. IDS (벤더 자료)](https://corelight.com/resources/glossary/ndr-vs-ids)
- [Exabeam, SIEM vs. IDS: Key Differences and Using Them Together (벤더 자료)](https://www.exabeam.com/explainers/siem/siem-vs-ids-key-differences-and-using-them-together/)
- [Cisco, What is a next-generation firewall (NGFW)? (Gartner 정의 인용, 벤더 자료)](https://www.cisco.com/c/en_in/products/security/firewalls/what-is-a-next-generation-firewall.html)
- [Fortinet, Firewall and IPS vs. NGFW: Which is Best for You? (벤더 자료)](https://www.fortinet.com/blog/business-and-technology/two-options-for-evolving-your-ips-solution)
- [Cisco, Know When to Have a Dedicated NGIPS vs. NGFW (벤더 자료)](https://blogs.cisco.com/security/know-when-to-have-a-dedicated-ngips-vs-ngfw/)
- [Huawei, What Is an NGFW? (벤더 자료)](https://info.support.huawei.com/info-finder/encyclopedia/en/NGFW.html)
- [Palo Alto Networks, Why Application and User Context are Vital to Next-Generation IPS (벤더 자료)](https://www.paloaltonetworks.com/blog/2015/05/why-application-user-awareness-are-vital-to-next-generation-ips/)
- [A Survey of Learning-Based Intrusion Detection Systems for In-Vehicle Network (arXiv:2505.11551)](https://arxiv.org/html/2505.11551v1)
- [Vehicular Intrusion Detection System for Controller Area Network: A Comprehensive Survey and Evaluation (arXiv:2505.17274)](https://arxiv.org/pdf/2505.17274)
- [Applied Sciences 14(7):2805, CAN 데이터 이상 탐지 논문](https://www.mdpi.com/2076-3417/14/7/2805)
- [차량 CAN 네트워크 침입 탐지 서베이 (Springer, 2019)](https://link.springer.com/article/10.1186/s13638-019-1484-3)
- [A survey of learning-based intrusion detection systems for in-vehicle networks (Computer Networks)](https://orca.cardiff.ac.uk/id/eprint/184156/1/1-s2.0-S1389128626000435-main.pdf)
- [Blockchain integration for in-vehicle CAN bus intrusion detection systems with ISO/SAE 21434 compliant reporting (Scientific Reports, 2024)](https://www.nature.com/articles/s41598-024-58694-4)
- [IBM, What is SOAR (security orchestration, automation and response)? (벤더 자료)](https://www.ibm.com/topics/security-orchestration-automation-response)
- [Google Cloud, Cloud IDS overview](https://docs.cloud.google.com/intrusion-detection-system/docs/overview)
- [AWS, What is Amazon GuardDuty?](https://docs.aws.amazon.com/guardduty/latest/ug/what-is-guardduty.html)
- [The Falco Project, Falco Docs](https://falco.org/docs/)
