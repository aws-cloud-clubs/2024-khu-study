## 6장 키-값 저장소 설계

### - <span style="background-color:#fff5b1">_키-값 저장소_</span>
> - 키-값 데이터 베이스라고도 불리는 비 관계형 데이터베이스이다.
- 키-값  쌍은 유일해야하며, 값은 키를 통해서만 접근 가능하다.
- 키는 일반텍스트일수도 있고, 해시 값일 수도 있다.
- 키는 짧을 수록 좋다.
- 가장 널리 알려진 키 값 저장소는 아마존 다이나모, memcahed, 레디스가 있다.


### - <span style="background-color:#fff5b1">_문제 이해 및 설계 범위 확정 _</span>
> - 키-값 쌍의 크기는 10KB이하이다. 
- 큰 데이터를 저장할 수 있어야 한다.
- 높은 가용성을 제공해야 한다.
- 높은 규모 확장성을 제공해야 한다.
- 데이터 일관성 수준은 조정이 가능해야 한다.
- 응답 지연시간이 짧아야 한다.

### - <span style="background-color:#fff5b1">_단일 서버 키-값 저장소_</span>
> - 가장 쉬운 방법으로 키-값 쌍을 전부 메모리에 해시테이블로 저장하는 방법
- 빠른 속도를 보장한다.
- <span style="color:red"> BUT </span> 모든 메모리 안에 두는 것이 불가능할 수 있다.
-> 해결 방안으로 1. 데이터압축, 2. 메모리-디스크분산 저장
- <span style="color:red"> BUT </span> 결국 한 대 서버로 부족한 때가 찾아온다. 

### - <span style="background-color:#fff5b1">_분산 키-값 저장소_</span>
> 
- 여러 서버에 키-값 쌍을 분산시키는 방법
#### <span style='background-color:#f7ddbe'>**_CAP 정리_**</span>
>> - 일관성(consistency), 가용성(availability), 파디션 감내(partition Tolerance)라는 세가지 요구사항을 동시에 만족하기는 불가능하다.
- 각 서비스에 맞는 전략을 취해야한다.
>>> **일관성** : 클라이언트가 어떤 노드를 통해서 접속하던지 같은 데이터를 보게해야한다.<br>
>>> **가용성** : 일부 노드에 장애가 발생하더라도 항상 응답을 받을 수 있어야한다.<br>
>>> **파티션 감내** : 파티션은 두 노드 사이에 장애가 발생하였음을 의미하고, 파티션 감내는 네트워크에 파티션이 생기더라도 시스템은 계속 동작하여야한다는 것을 의미한다.

>>- CAP 정리는 이들 가운데 어떤 두가지를 충족하려면 나머지 하나는 반드시 희생되어야 한다는 것을 의미한다.
![](https://velog.velcdn.com/images/kjeng7897/post/84d67c97-c29a-4d8e-812e-cd83bcef43cc/image.png)
>>> **CP 시스템** : 일관성과 파티션 감내를 지원하는 키-값 저장소, 가용성을 희생한다.
>>> **AP 시스템** : 가용성과 파티션 감내를 지원하는 키-값 저장소, 데이터 일관성을 희생한다.
>>> **CA 시스템** : 일관성과 가용성을 지원하는 키-값 저장소, 파티션 감내는 지원하지 않는다. 실세계에 존재 X

### - <span style="background-color:#fff5b1">_시스템 컴포넌트_</span>

> #### <span style='background-color:#f7ddbe'>**_키-값 저장소 구현에 사용되는 핵심 컴포넌트들 및 기술들_**</span> 
- 데이터 파티션
- 데이터 다중화
- 일관성
- 일관성 불일치 해소
- 장애 처리
- 시스템 아키텍쳐 다이어그램
- 쓰기 경로
- 읽기 경로

> #### <span style='background-color:#f7ddbe'>**_데이터 파티션_**</span>
- 데이터를 작은 파티션으로 분할한 다음 여러 서버에 저장하는 것이다.
- 살펴야 할 점들
    - 데이터를 여러 서버에 고르게 분산할 수 있는가?
    - 노드가 추가되거나 삭제될 때 데이터의 이동을 최소화할 수 있는가?
- 안정 해시는 이런 문제를 푸는데 적합하다.
![](https://velog.velcdn.com/images/kjeng7897/post/26dd42d9-b92e-4d06-a266-6402ecff2851/image.png)
-> 안정해서 : 서버를 해시 링에 배치하고, 키-값 의 해당 키를 링 위에 배치한 후, 가장 가까운 시계방향 서버에 저장하는 방식
<br>
-- 장단점 --
- 규모 확장 자동화(automatic scaling)
- 다양성(heterogeneity)
	-> 각 서버의 용량에 맞게 가상 노드의 수를 조정할 수 있다.
    


> #### <span style='background-color:#f7ddbe'>**_데이터 다중화_**</span>
> - 높은 가용성과 안정성을 확보하기 위해 데이터를 N개 서버에 비동기적으로 다중화할 필요가 있다.
- N : 보관할 원본과 사본의 총 갯수
-> 해시링을 순회하며 만나는 N개의 서버에 저장된다.
-> <span style="color:red"> BUT </span> 같은 물리서버를 중복 선택하지 않도록 조심해야 한다.


> #### <span style='background-color:#f7ddbe'>**_데이터 일관성_**</span>
- 여러 노드에 다중화된 데이터는 동기화가 되어야 한다.
- N : 사본개수
- W : 쓰기 연산에 대한 정족수(쓰기 연산이 성공한 것으로 간주되려면 적어도 W개의 서버로부터 쓰기 연산이 성공했다는 응답을 받아야한다.)
- R : 읽기 연산에 대한 정족수(읽기 연산이 성공한 것으로 간주되려면 적어도 R개의 서버로부터 읽기 연산이 성공했다는 응답을 받아야한다.)
>> R = 1, W = N : 빠른 읽기 연산에 최적화된 시스템
>> R = N, W = 1 : 빠른 쓰기 연산에 최적화된 시스템
>> R + W > N : 강한 일관성 보장됨(보통 N 3, W, R 2)
>> R + W <= N : 강한 일관성 보장되지 않음


> #### <span style='background-color:#f7ddbe'>**_일관성 모델_**</span>
>> **강한 일관성** : 모든 읽기 연산은 가장 최근에 갱신된 결과를 반환한다.
>> **약한 일관성** : 읽기 연산은 가장 최근에 갱신된 결과를 반환하지 못할 수 있다. 
>> **결과적 일관성** : 약한 일관성의 한 형태로, 갱신 결과가 결국에는 모든 사본에 반영되는 모델이다.

> #### <span style='background-color:#f7ddbe'>**_비일관성 해소 기법 : 데이터 버저닝_**</span>
>- 데이터를 다중화하면 가용성은 높아지짖만, 사본 간 일관성이 깨질 수 있다.
- **_버저닝_**과 **_벡터 시계_**를 통해 해소가능하다. 
![](https://velog.velcdn.com/images/kjeng7897/post/eb835cf2-7818-4283-9258-12bcdaa05e51/image.png)
- 기존에 John이라고 저장되어 있던 데이터를 JohnSanfransico와 JohnNewYork으로 동시에 쓰기연산을 통해 수정한다고 한다.
- 각각의 사본에 대해 수정을 수행하여 충돌하는 두 값을 갖게 되었다.
- 변경 전 값이 같았기 때문에 변경 이전 원래 값은 무시할 수 없다. 
- 이 문제를 해결하기 위해 벡터 시계를 적용한다.
![](https://velog.velcdn.com/images/kjeng7897/post/0fa1b7fc-48e4-4f2e-bd86-c5e972dd8e87/image.png)
1. 클라이언트가 데이터 D1을 서버 Sx에서 시스템에기록한다. 
2. 다른 클라이언트가 데이터 D1을 읽고 D2로 업데이트 한 다음 기록한다. 이때는 같은 서버 Sx에서 처리한다.
3. 다른 클라이언트가 서버 Sy에서 D2를 읽어 D3으로 갱신한 다음 기록한다. 
4. 다른 클라이언트가 서버 Sz에서 D2를 읽어 D4로 갱신한다.
5. 어떤 클라이언트가 D3, D4를 읽으며 데이터 간 충돌이 있다는 것을 알게 되고, 이 충돌을 클라이언트가 해소한 후에 Sx 서버에서 기록한다.

>-- 장단점 --
- 서로간 선후관계를 알 수 있다.
- <span style="color:red"> BUT </span> 충돌 감지 및 해소 로직이 클라이언트에 들어가야 하므로, 클라이언트 구현이 복잡해진다.
- <span style="color:red"> BUT </span> [서버 : 버전]의 순서쌍이 굉장히 빨리 늘어난다는 것이다.




                                   

> #### <span style='background-color:#f7ddbe'>**_장애 처리_**</span>
- 대규모 시스템에서 장애는 흔하게 벌어진다.
- 장애를 탐지한다고 해소하는것이 아니다.
- 탐지와 해소는 별개의 문제인 것이다. 

> #### <span style='background-color:#f7ddbe'>**_장애 감지_**</span>
- 보통 두대 이상의 서버가 똑같이 다른 서버의 장애를 보고해야 장애가 발생했다고 판단하다.
- 멀티캐스팅 채널 구축을 통한 장애판단
-> 서버가 많을 때는 비효율적
![](https://velog.velcdn.com/images/kjeng7897/post/a5b868fa-18a2-4111-9c5c-b58a85784e90/image.png)
- 가십 프로토콜(분산형 장애감지)를 통한 장애판단
-> 주기적으로 무작위 노드들과 통신을 통해 장애를 판단한다.
-> 장애로 판단되면, 다른 정상 노드에게 이를 전달
![](https://velog.velcdn.com/images/kjeng7897/post/93b779aa-a62e-44c7-a534-bf93108421f7/image.png)



> #### <span style='background-color:#f7ddbe'>**_일시적 장애 처리_**</span>
- 엄격한 정족수 접근법대로라면, 일관성을 위해 읽기와 쓰기 연산을 금지해야 한다.
- 느슨한 정족수 접근법을 이용해 가용성을 높힐 수 있다. 
-> 정족수 요구사항을 강제하는 대신, 쓰기 연산을 수행할 W개의 건강한 서버와 읽기 연산을 수행할 R개의 건강한 서버를 해시링에서 고른다. 이때 장애 상태인 서버는 무시한다.
-> 복구되면, 장애서버로 갱신된 데이터를 인계한다.

> #### <span style='background-color:#f7ddbe'>**_영구적 장애 처리_**</span>
- 반-엔트로피 프로토콜을 구현하여 사본들을 동기화할수 있다.
- 반-엔트로피 프로토콜은 사본들을비교하여 최신 버전으로 갱신하는 과정을 포함
- 사본 간의 일관성 장애 탐지와 전송 데이터 압축을 위해 머클 트리 사용


> #### <span style='background-color:#f7ddbe'>**_시스템 아키텍쳐 다이어그램_**</span>
- 클라이언트는 키-값 저장소가 제공하는 두가지 단순한 API, 즉 get, put와 통신한다.
- 중재자는 클라이언트에게 키-값 저장소에 대한 프록시 역할을 하는 노드다
- 노드는 안정해시의 해시 링 위에 분포한다.
![](https://velog.velcdn.com/images/kjeng7897/post/34bf0f31-5e3c-4f64-b894-d3bd5f9c60ea/image.png)
- 노드를 자동으로 추가 또는 삭제할 수 있도록, 시스템은 완전히 분산된다.
- 데이터는 여러 노드에 다중화된다.
- 모든 노드가 같은 책임을 지므로, SPOF 는 존재하지 않는다.
- 완전히 분산된 설계를 채택했으므로, 모든 노드는 그림에 제시된 기능을 전부 지원해야한다.
![](https://velog.velcdn.com/images/kjeng7897/post/3a53305f-1448-43f7-8519-40985d11bf59/image.png)



> #### <span style='background-color:#f7ddbe'>**_쓰기 경로_**</span>
> - 쓰기 요청이 특저 노드에 전달되면 무슨일이 벌어지는지
![](https://velog.velcdn.com/images/kjeng7897/post/ef204469-738b-4378-b1d0-b9f3478159b9/image.png)
1. 쓰기 요청이 커밋 로그 파일에 기록된다.
2. 데이터가 메모리 캐시에 기록된다.
3. 메모리 캐시가 가득차거나 사전에 정의된 어떤 임계치에 도달하면 데이터는 디스크에 있는 SSTable 에 기록된다.

> #### <span style='background-color:#f7ddbe'>**_읽기 경로_**</span>
- 읽기 요청을 받은 노드는 메모리캐시에 있는지부터 살핀다.
- 메모리 캐시에 있는 경우
![](https://velog.velcdn.com/images/kjeng7897/post/652af562-601d-4d91-84c3-53b4a37095f3/image.png)
- 메모리에 캐시에 없는 경우
![](https://velog.velcdn.com/images/kjeng7897/post/1c1cf527-a786-4e93-adbe-fca39b298902/image.png)
1. 데이터가 메모리에 있는지 검사한다. 
2. 데이터가 메모리에 없으므로 볼룸 필터를 검사한다.
3. 볼륨 필터를 통해 어떤 SSTable에 키가 보관되어 있는지 알아낸다.
4. SSTable 에서 데이터를 가져온다.
5. 해당 데이터를 클라이언트에게 반환한다. 


