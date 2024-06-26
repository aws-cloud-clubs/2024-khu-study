# #2 개략적인 규모 추정

보편적으로 통용되는 성능 수치상에서 사고 실험을 통해 어떤 설계가 요구사항에 부합하는지 보는 것.

## 2의 제곱수

데이터 볼륨 단위를 2의 제곱수로 나타내자.
최소 단위: 1바이트(8비트로 구성) //아스키 문자 하나는 1바이트

| 2의 x제곱 | 축약형 |
| --------- | ------ |
| 10        | 1KB    |
| 20        | 1MB    |
| 30        | 1GB    |
| 40        | 1TB    |
| 50        | 1PB    |

## 모든 프로그래머가 알아야 하는 응답 지연 값

컴퓨터에서 구현된 연산들의 응답지연 값.
처리 속도가 어느 정도인지 짐작할 수 있도록 해줌.

2020년 기준으로 시각화한 값을 보면 다음의 결론을 내릴 수 있음.
![image](https://velog.velcdn.com/images/dev_dc_hyeon/post/a7c2fc0c-f6e0-454f-9722-14dd69eb8d55/image.png)

- 메모리는 빠르지만 디스크는 느림.
- 디스크 탐색은 피하기
- 단순 압축 알고리즘은 빠름.
- 데이터를 인터넷에 보내기 전에 압축하기
- 데이터 센터는 여러 지역에 있고 센터간 데이터를 주고받는 데 시간이 걸림.

## 가용성에 관계된 수치

고가용성: 오랜 시간 동안 지속적으로 중단 없이 운영될 수 있는 능력. %로 표현.
SLA(Service Level Agreement): 서비스 사업자와 고객 사이에 맺어진 합의. 가용시간이 기술되어 있다.

가용성은 보통 99%~100%사이의 값을 갖으며 소수점까지 9가 많을수록 높은 가용성.
99%는 하루당 장애시간이 14.40분. 99.9999%는 86.40밀리초로 확연한 차이를 보임.

## 트위터 QPS와 저장소 요구량 추정

> 가정

- 월간 능동 사용자(monthly active user)는 3억 명
- 50%의 사용자가 트위터를 매일 사용
- 평균적으로 각 사용자는 매일 2건의 트윗을 올림
- 미디어를 포함하는 트윗은 약 10%
- 데이터는 5년간 보관

> 추정

QPS(Query Per Second) 추정치

- 일간 능동 사용자(Daily Active User, DAU) = 3억 X 50% = 1.5억
- QPS = 1.5억 X 2트윗/24시간/3600초 = 약 3500
- 최대 QPS(Peek QPS) = 2 X QPS = 약 7000 \*일반적으로 2배로 잡음. 시스템 기록을 통해 기간내 최고치로 잡을수도 있음.

QPS: 초당 서버가 응답할 수 있는 쿼리의 수
TPS: 초당 처리할 수 있는 트랜잭션의 수 (클라이언트 요청 → 서버 내부 처리 → 응답 // 1 TPS) \*페이지 방문시 1TPS but,, QPS는 1이상(실제 동작으로 게시글 조회, 댓글 조회 등등)

> 미디어 저장을 위한 저장소 요구량

- 평균 트윗 크기
  - tweet_id에 64바이트
  - 텍스트에 140바이트
  - 미디어에 1MB
- 미디어 저장소 요구량: 1.5억 X 2 X 10% X 1MB = 30TB/일
- 5년간 미디어를 보관하기 위한 저장소 요구량: 30TB X 365 X 5 = 약 55PB

> 팁

결과를 내는 것보다 '**올바른 절차를 밟았는지**' 가 중요

- 근사치를 활용한 계산 : 9888/9.234은 10000/10으로 보자
- 가정들은 나중에 볼 수 있게 적어두자
- 데이터 단위를 붙이자. 5를 보고 5KB, 5MB인지 알 수 없다
- 많이 출제되는 건 미리 계산해보자.
