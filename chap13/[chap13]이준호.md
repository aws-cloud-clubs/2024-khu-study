# #13 검색어 자동완성 시스템

## 1단계 문제 이해 및 설계 범위 특정

- 사용자가 입력하는 단어는 자동완성될 검색의 첫 부분인가? 중간 부분인가?
- 몇 개의 자동완성 검색어가 표시되어야 하는가?
- 자동완성 검색어를 고르는 기준은 무엇인가?
- 맞춤법 검사 기능도 제공해야 하는가?
- 질의는 영어인가?
- 대문자나 특수 문자 처리도 해야 하는가?

**요구사항**

- 빠른 응답속도 : 100밀리초 이내
- 연관성: 사용자가 입력한 단어와 연관된 검색어 출력
- 정렬: 계산 결과는 인기도 등의 순위 모델에 의해 정렬
- 규모 확장성: 많은 트래픽을 감당할 수 있도록 확장 가능할 것
- 고 가용성: 시스템 일부에 장애, 문제가 생겨도 시스템은 계속 사용 가능 할 것.


**개략적 규모 추정**
- 일간 능동 사용자(DAU)는 천만 명으로 가정
- 한 사용자는 매일 평균 10건의 검색을 수행
- 질의할 때마다 평균 20바이트의 데이터를 입력.(1문자 = 1 바이트, 평균 4단어, 단어=5문자)
- 검색창에 글자를 입력할 때마다 클라이언트는 검색어 자동완성 백엔드에 요청을 보냄. 즉 평균 1회 검색당 20건의 요청이 전달됨.
- 대략 초당 24,000건의 질의(QPS)qkftod.
- 최대 QPS = QPS * 2 = 48,000
- 질의 중 20%는 신규 검색어. -> 0.4GB, 즉 매일 0.4GB의 신규 데이터가 시스템에 추가됨.

## 2단계 개략적 설계안 제시 및 동의 구하기

시스템은 크게 두 부분
- 데이터 수집 서비스 : 사용자가 입력한 질의를 실시간 수집. 데이터가 많은 애플리케이션이라면 실시간 시스템은 바람직하지 않음. but.. 설계안을 만드는 출발로는 괜찮다.
- 질의 서비스 : 주어진 질의에 다섯 개의 인기 검색어를 정렬해 내놓는 서비스

## 3단계 상세 설계

> 트라이 자료구조

관계형 DB로 상위 다섯 개의 질의문을 골라내는 것은 효율적이지 않음. -> 트라이를 사용해 해결.

*트라이 -> 문자열을 간략하게 저장할 수 있는 자료구조.
![](https://velog.velcdn.com/images/kimhalin/post/897cd4fb-8e20-4e55-9712-9ce24939b513/image.png)

질의어 수 : k , 접두어 길이 : p, 트라이 안 노드 수 : n, 주어진 노드의 자식 노드 수 : c

1. 접두어 노드를 찾는다
2. 해당 노드부터 하위 트리를 탐새갛여 모든 유효 노드를 찾는다.
3. 유효 노드를 정렬하여 k개만 골라낸다.

트리구조임을 감안하면 시간 복잡도는 
O(p) + O(c) + O(c lg(c))

*전체 트라이를 검색하는 일이 생기지 않도록
- 접두어 최대 길이 제한
- 각 노드에 인기 검색어 캐시

> 데이터 수집 서비스

실시간으로 데이터를 수정하는 것은 아래의 문제로 별로 좋지 않음
- 매일 수천만 건의 질의를 그때그때 트라이 갱신하면 매우매우 느려질 것.
- 트라이가 만들어지고 나면 인기 검색어는 자주 바뀌지 않을 것이므로 자주 갱신할 필요가 없다.

**Flow**
데이터 분석 로그 -> 로그 취합 서버 -> 취합된 데이터 -> 작업 서버 -매주갱신-> 트라이 DB
-매주 DB 상태를 스냅샷-> 트라이 캐시

- 데이터 분석 로그 : 질의에 관한 원본 데이터 보관
- 로그 취합 서버 : 대부분 일주일에 한 번 정도 취합.(실시간성이 얼마나 중요한지에 따라 다름)
- 취합된 데이터 : 로그 취합 서버에 의해 취합된 데이터들.
- 작업 서버 : 주기적으로 비동기 작업을 하는 서버 집합 ( 트라이 자료구조 생성 및 저장)
- 트라이 캐시 : 분산 캐시 시스템으로 읽기 연산 성능 높임.
- 트라이 DB: 지속성 저장소
1. 문서 저장소
2. 키-값 저장소 

> 질의 서비스

질의 -> 로드 밸런서 -> 해당 API서버 -> 트라이 캐시를 이용한 검새개어 제안 응답 -> 캐시에 없다면 DB에서 가져와 캐시에 채움.

질의 서비스는 매우 빨라야하므로...
- AJAX요청 : 새로고침을 하지 않아도 된다
- 브라우저 캐싱 : 제안 결과는 빠른 시간안에 바뀌지 않으므로 브라우저 캐시에 두어 후속 질의의 결과는 해당 캐시에서 바로 가져갈 수 있다.
- 데이터 샘플링 : N개 요청 중 1개만 로깅

> 트라이 연산

- 트라이 생성 : 작업 서버가 담당
- 트라이 갱신 :
1. 매주 한 번 갱신
2. 각 노드를 개별적으로 갱신
- 검색어 삭제 : 위험한 질의어 결과에서 제거 / 트라이 캐시 앞에 필터 계층을 두어 구현

> 저장소 규모 확장

샤드 관리자를 두어 각 서버당 검색어 양이 비슷하도록 하고 알파벳을 끊어 샤딩을 구현한다.
- 서버가 2대라면 a~m은 1번 서버 나머지는 2번 서버
알파벳이 총 26개이므로 최대 26개의 서버로 샤딩가능. 이후 aa~am , 그외 이런식으로 하위 계층도 샤딩을 할 수 있음.





## 4단계 마무리

- 다국어 지원이 가능하도록 시스템을 확장하려면? : 트라이에 유니코드 데이터를 저장한다.
- 국가별로 인기 검색어 순위가 다르다면? : 국가별로 다른 트라이를 사용한다. 트라이를 CDN에 저장하여 응답속도를 높일 수도 있다.
*CDN : Content Delivery Network , 지리적으로 분산된 서버들을 연결한 네트워크
- 실시간으로 변하는 검색어의 추이를 반영하려면? 
1. 샤딩을 통해 작업 대상 데이터 양을 줄인다
2. 순위 모델을 바꾸어 최근 검색어에 보다 높은 가중치를 준다
3. 데이터가 한 번에 오는 것이 아니므로 동시에 사용할 수 없을 수도 있다. 따라서 스프림 프로세싱을 위한 특별한 종류의 시스템이 필요하다. ex. 아파치 하둡 맵리듀스, 스파크 스트리밍, 스톰, 카프카 등,,,

 **추가로 다룰만한 주제**
 - 