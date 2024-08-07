# Ch7

# 분산 시스템을 위한 유일 ID 생성기 설계

### 1단계 - 문제 이해 및 설계 범위 확정

- ID는 유일해야 함
- ID는 숫자로만 구성되어야 함
- ID는 64비트로 표현될 수 있는 값이어야 함
- ID는 발급 날짜에 따라 정렬 가능해야 함
- 초당 10,000개의 ID를 만들 수 있어야 함

### 2단계 - 개략적 설계안 제시 및 동의 구하기

: 다중 마스터 복제, UUID, 티켓 서버, 트위터 스노플레이크 접근법 중 선택하여 유일성이 보장되는 ID를 만들 수 있음

- **다중 마스터 복제 (multi-master replication)**
    
    : 데이터베이스의 auto_increment 기능을 활용하는 것. 단, 다음 ID의 값을 구할 때 1을 증가시켜 얻는 것이 아니라, k만큼 증가시킴
    
    (여기서 k는 현재 사용중인 데이터베이스 서버의 수)
    
    **단점**
    
    - 여러 데이터 센터에 걸쳐 규모를 늘리기 어려움
    - ID의 유일성은 보장되겠지만, 그 값이 시간 흐름에 맞추어 커지도록 보장할 수 없음
    - 서버를 추가/삭제할 때 잘 동작하도록 만들기 어려움

- **UUID (Universally Unique Identifier)**
    
    : 컴퓨터 시스템에 저장되는 정보를 유일하게 식별하기 위한 128비트 수. 충돌 가능성이 지극히 낮음
    
    **장점**
    
    - UUID를 만드는 것은 단순함. 서버 사이 조율이 필요 없으므로 동기화 이슈도 없음
    - 각 서버가 자기가 쓸 ID를 알아서 만드는 구조이므로 규모 확장이 쉬움
    
    **단점**
    
    - ID가 128비트로 긺. 이번 장의 요구사항은 64비트.
    - ID를 시간 순으로 정렬할 수 없음
    - ID에 숫자가 아닌 값이 포함될 수 있음

- **티켓 서버(Ticket Server)**
    
    : 플리커(Flicker)가 분산 기본 키를 만들어 내기 위해 사용한 기술.
    
    auto_increment 기능을 갖춘 데이터베이스 서버(티켓 서버)를 중앙 집중형으로 하나만 사용하는 것
    
    **장점**
    
    - 유일성이 보장되는 오직 숫자로만 구성된 ID를 쉽게 만들 수 있음
    - 구현이 쉽고, 중소 규모 애플리케이션에 적합함
    
    **단점**
    
    - 티켓 서버가 SPOF(Single-Point-of-Faliure)가 됨
    이 서버에 장애 발생 시, 해당 서버를 이용하는 모든 시스템이 영향을 받음

- **트위터 스노플레이크(Twitter Snowflake) 접근법**
    
    : 트위터에서 사용하는 독창적인 ID 생성 기법
    
    divide and conquer를 적용하여 ID의 구조를 여러 절(section)으로 분할한다
    
    사인비트(1) - 타임스탬프(41) - 데이터센터ID(5) - 서버ID(5) - 일련번호(12)
    

### 3단계 - 상세 설계

데이터센터ID, 서버ID는 시스템이 시작될 때 결정되고, 일반적으로 시스템 운영 중에는  바뀌지 않음 - 잘못 변경 시 충돌 발생 가능

- **타임스탬프**
    
    UTC 시각을 이진 표현 형태로 추출하여 타임스탬프 값으로 사용
    
- **일련번호**
    
     어떤 서버가 같은 밀리초 동안 1개 이상의 ID를 만들어낸 경우에만 0보다 큰 값을 갖게 됨
    

### 4단계 - 마무리

추가로 논의할 수 있는 사항

- **시계 동기화 (clock synchronization)**
    
    : 서버가 물리적으로 독립된 여러 장비에서 실행되는 경우, 
    하나의 서버가 여러 코어에서 실행되는 경우
    → ID생성 서버들이 모두 같은 시계를 사용한다는 가정이 유효하지 않을 수 있음
    
    NTP(Network Time Protocol)로 해결 가능
    

- **각 절(section)의 길이 최적화**
    
    : 동시성이 낮고, 수명이 긴 애플리케이션의 경우, 일련번호 절의 길이를 줄이고, 타임스탬프 절의 길이를 늘리는 것이 효과적일 수 있음
    

- **고가용성(high availability)**
    
    : ID 생성기는 필수 불가결 컴포넌트이므로, 아주 높은 가용성을 제공해야 함