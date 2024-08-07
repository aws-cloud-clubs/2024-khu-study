# #8 URL 단축기 설계

- URL 단축
    - 주어진 긴 URL을 훨씬 짧게 줄임
- URL 리디렉션(redirection)
    - 축약된 URL로 HTTP 요청이 오면 원래 URL로 안내
- 높은 가용성과 규모 확장성, 그리고 장애 감내가 요구됨

### API 엔드포인트(endpoint)

클라이언트는 서버가 제공하는 API 엔드포인트를 통해 서버와 통신한다. 우리는 이 엔드포인트를 REST 스타일로 설계할 것이다. URL 단축기는 기본적으로 두 개의 엔드포인트를 필요로 한다.

1. URL 단축용 엔드포인트
    
    새 단축 URL을 생성하고자 하는 클라이언트는이 엔드포인트에 단축할 URL을 인자로 실어서 POST 요청을 보내야 한다.
    이 엔드포인트는 아래와 같은 형태를 가진다.
    
    **POST/api/v1/data/shorten**
    
    - 인자
        - {longUrl: longURLstring}
    - 반환
        - 단축 URL
2. URL 리디렉션용 엔드포인트
    
    단축 URL에 대해서 HTTP 요청이 오면 원래 URL로 보내주기 위한 용도의 엔드포인트이다.
    아래와 같은 형태를 가진다.
    
    **GET/api/v1/shortUrl**
    
    - 반환
        - HTTP 리디렉션 목적지가 될 원래 URL


### URL 리디렉션

상태 코드 301, 302는 둘 다 리디렉션 응답이지만 차이가 존재한다.

- 301 Permanently Moved
    - 해당 URL에 대한 HTTP 요청의 처리 책임이 영구적으로 Location 헤더에 반환된 URL로 이전되었다는 응답
    - 영구적으로 이전되었으므로, 브라우저는 이 응답을 캐시(cache)함
    - 추후 같은 단축 URL에 요청을 보낼 필요가 있을 때 브라우저는 캐시된 원래 URL로 요청을 보냄
- 302 Found
    - 주어진 URL로의 요청이 ‘일시적으로’ Location 헤더가 지정하는 URL에 의해 처리되어야 한다는 응답
    - 클라이언트의 요청은 언제나 단축 URL 서버에 먼저 보내진 후에 원래 URL로 리디렉션 되어야 함

**두 방법의 각기 다른 장단점**

- 서버 부하를 줄이는 것이 중요하다면 301 Permanent Moved를 사용
    - 첫 번째 요청만 단축 URL 서버로 전송되기 때문
- 트래픽 분석이 중요할 때는 302 Found를 쓰는 쪽이 클릭 발생률이나 발생 위치를 추적하는데 유리

URL 리디렉션을 구현하는 가장 직관적인 방법은 **해시 테이블을 사용하는 것**이다. 해시 테이블에 <단축 URL, 원래 URL>의 쌍으로 저장한다고 가정하면, URL 리디렉션은 다음과 같이 구현될 것이다.

- 원래 URL = hashTable.get(단축 URL)
- 301 또는 302 응답 Location 헤더에 원래 URL을 넣은 후 전송


### URL 단축 플로

단축 URI이 www.tinyurl.com/{hashValue} 같은 형태라고 가정한다. 중요한 것은 긴 URL을 이 해시 값으로 대응시킬 해시 함수 fx를 찾는 일이 될 것이다.

![URL 단축 플로](https://github.com/user-attachments/assets/872ebde8-5bba-475e-9cdb-c7a6ec46ced6)


이 해시 함수는 다음 요구사항을 만족해야 한다.

- 입력으로 주어지는 긴 URL이 다른 값이면 해시 값도 달라야 함
- 계산된 해시 값은 원래 입력으로 주어졌던 긴 URL로 복원될 수 있어야 함


### 데이터 모델

개략적 설계를 진행할 때는 모든 것을 해시 테이블에 두었다. 이 접근법은 초기 전략으로 괜찮지만 실제 시스템에 쓰기에는 메모리가 유한적이고 비싸기 때문에 곤란하다.

더 나은 방법은 <단축 URL, 원래 URL>의 순서쌍을 관계형 데이터베이스에 저장하는 것이다.

![테이블 설계](https://github.com/user-attachments/assets/dba32eba-ab1e-4cc5-a4a5-529049954f95)


### 해시 함수

해시 함수(hash function)는 원래 URL을 단축 URL로 변환하는데 쓰인다.
편의상 해시 함수가 계산하는 단축 URL 값을 hashValue라고 지칭한다.

**해시 값 길이**

hashValue는 [0-9, a-z, A-Z]의 문자들로 구성된다. 따라서 사용할 수 있는 문자의 개수는 10 + 26 + 26 = 62개다. hashValue의 길이를 정하기 위해서는 62^(n) ≥ 3650억(365billion)인 n의 최솟값을 찾아야 한다. 개략적으로 계산했던 추정치에 따르면 이 시스템은 3650억 개의 URL을 만들 수 있어야 한다.

해시 함수 구현에 쓰일 기술로는 ‘**해시 후 충돌 해소**’와 ‘**base-62 변환**’의  두 가지 방법이 있다.

1. **해시 후 충돌 해소**

긴 URL을 줄이려면, 원래 URL을 7글자 문자열로 줄이는 해시 함수가 필요하다. 손쉬운 방법은 CRC32, MD5, SHA-1같이 잘 알려진 해시 함수를 이용하는 것이다.

2. **base-62 변환**

진법 변환(base conversion)은 URL 단축기를 구현할 때 흔히 사용되는 접근법중 하나다. 이 기법은 수의 표현 방식이 다른 두 시스템이 같은 수를 공유하여야 하는 경우에 유용하다.  
62진법을 쓰는 이유는 hashValue에 사용할 수 있는 문자(character) 개수가 62개이기 때문이다. 10진수로 11157을 62진수(62진법을 따르는 수)로 변환한다.

- 62진법은 수를 표현하기 위해 총 62개의 문자를 사용하는 진법
    - 62진법에서 ‘a’는 10을 나타내고, ‘Z’는 61을 나타냄


### URL 단축기 상세 설계

URL 단축기는 시스템의 핵심 컴포넌트이므로, 그 처리 흐름이 논리적으로는 단순해야 하고 기능적으로는 언제나 동작하는 상태로 유지되어야 한다. 예제에서는 62진법 변환 기법을 사용해 설계할 것이다.

![처리 흐름](https://github.com/user-attachments/assets/2fe8f737-b1fd-4d4b-9669-1a03402c6b6e)

#### 처리 흐름

1. 입력으로 긴 URL을 받음
2. 데이터베이스에 해당 URL이 있는지 확인
3. 데이터베이스에 있다면 해당 URL에 대한 단축 URL을 만든 적이 있는 것이므로, 데이터베이스에서 해당 단축 URL을 가져와서 클라이언트에게 반환
4. 데이터베이스에 없는 경우 해당 URL은 새로 접수된 것이므로 유일한 ID를 생성하고, 이 ID는 데이터베이스의 기본키로 사용
5. 62진법 변환을 적용, ID를 단축 URL로 만듬
6. ID, 단축 URL, 원래 URL로 새 데이터베이스 레코드를 만든 후 단축 URL을 클라이언트에게 전달


### URL 리디렉션 상세 설계

쓰기보다 읽기를 더 자주 하는 시스템이라, <단축 URL, 원래 URL>의 쌍을 캐시에 저장하여 성능을 높인다.

![URL 리디렉션 상세 설계](https://github.com/user-attachments/assets/b53f9c8d-7130-42c7-8245-110f214efcca)

**로드밸런서의 동작 흐름은 다음과 같이 요약**할 수 있다.

1. 사용자가 단축 URL을 클릭
2. 로드밸런서가 해당 클릭으로 발생한 요청을 웹 서버에 전달
3. 단축 URL이 이미 캐시에 있는 경우에는 원래 URL을 바로 꺼내서 클라이언트에게 전달
4. 캐시에 해당 단축 URL이 없는 경우에는 데이터베이스에서 꺼냄, 데이터베이스에 없다면 아마 사용자가 잘못된 단축 URL을 입력한 경우로 추측
5. 데이터베이스에서 꺼낸 URL을 캐시에 넣은 후 사용자에게 반환