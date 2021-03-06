## 문제 이해 및 설계 범위 확정

- 키-값 쌍의 크기는 10KB 이하
- 큰 데이터를 저장할 수 있어야 함
- 높은 가용성을 제공해야 함. 따라서 시스템은 설사 장애가 있더라도 빨리 응답해야 함
- 높은 규모 확장성을 제공해야 함. 따라서 트래픽 양에 따라 자동적으로 서버 증설/삭제가 이루어져야 함
- 데이터 일관성 수준은 조정이 가능해야 함
- 응답 지연시간(latency)이 짧아야 함

### 단일 서버 키-값 저장소

- 가장 직관적인 방법은 키-값 쌍 전부를 메모리에 해시 테이블로 저장하는 것
- 빠른 속도를 보장하긴 하지만 모든 데이터를 메모리 안에 두는 것이 불가능 할 수도 있다는 약점

#### 개선책

- 데이터 압축(compression)
- 자주 쓰이는 데이터만 메모리에 두고 나머지는 디스크에 저장

## 분산 키-값 저장소

-  분산 해시 테이블
- 키-값 쌍을 여러 서버에 분산
- CAP 정리(Consistency, Availability, Partition Tolerance theorem)를 이해하고 있어야 함

### CAP 정리

- 데이터 일관성(consistency), 가용성(availability), 파티션 감내(partition tolerance)라는 세가지 요구사항을 동시에 만족하는 분산 시스템을 설계하는 것은 불가능하다는 정리
- 어떤 두 가지를 충족하려면 나머지 하나는 반드시 희생되어야 함

### 데이터 일관성(consistency)

- 분산 시스템에 접속하는 모든 클라이언트는 어떤 노드에 접속했느냐에 관계없이 언제나 같은 데이터를 보게 되어야 함

#### 가용성(availability)

- 분산 시스템에 접속하는 클라이언트는 일부 노드에 장애가 발생하더라도 항상 응답을 받을 수 있어야 함

### 파티션 감내(partition tolerance)

- 파티션은 두 노드 사이에 통신 장애가 발생하였음을 의미
- 파티션 감내는 네트워크에 파티션이 생기더라도 시스템은 계속 동작하여야 함을 의미
- 키-값 저장소는 위 세가지 요구사항 가운데 어느 두 가지를 만족하느냐에 따라 다음과 같이 분류할 수 있다.

### CP 시스템

- 일관성과 파티션 감내를 지원하는 키-값 저장소. 가용성을 희생

### AP 시스템

- 가용성과 파티션 감내를 지원하는 키-값 저장소. 데이터 일관성을 희생

### CA 시스템

- 일관성과 가용성을 지원하는 키-값 저장소. 파티션 감내는 지원하지 않음.
- 하지만 통상 네트워크 장애는 피할 수 없는 일로 여겨지므로, 분산 시스템은 반드시 파티션 문제를 감내할 수 있도록 설계되어야 함.
- 그러므로 실세계에 CA 시스템은 존재하지 않음

## 실세계의 분산 시스템

### CP 시스템 선택

- 가용성 대신 일관성 선택
- 데이터 불일치 문제를 피하기 위해 쓰기 연산 중단시킴. 가용성 깨짐
- ex) 온라인 뱅킹 시스템이 계좌 최신 정보를 출력하지 못한다면 큰 문제
- 네트워크 파티션 때문에 일관성이 깨질 수 있는 상황이 발생하면 이런 시스템은 해결될 때까지 오류를 반환해야 함

### AP 시스템 선택

- 일관성 대신 가용성 선택
- 낡은 데이터를 반환할 위험이 있더라도 계속 읽기 연산 허용
- n1, n2는 계속 쓰기 연산 허용하고, 파티션 문제가 해결되면 새 데이터를 n3로 전송

## 시스템 컴포넌트

### 데이터 파티션

- 다음 두 가지 문제를 중요하게 따져봐야 함
    - 데이터를 여러 서버에 고르게 분산할 수 있는가
    - 노드가 추가되거나 삭제될 때 데이터의 이동을 최소화할 수 있는가

- 안정해시가 이런 문제를 푸는 데 적합한 기술
    - 서버를 해시 링에 배치
    - 어떤 키-값 쌍을 어떤 서버에 저장할지 결정하려면 해당 키를 같은 링 위에 배치
    - 그 지점으로부터 링을 시계 방향으로 순회하다 만나는 첫번째 서버가 바로 해당 키-값 쌍을 저장할 서버

### 안정 해시 사용해 파티션시 장점

- 규모 확장 자동화 : 시스템 부하에 따라 서버가 자동으로 추가되거나 삭제되도록 만들 수 있음
- 다양성 : 각 서버의 용량에 맞게 가상 노드의 수를 조정할 수 있음. 고성능 서버는 더 많은 가상노드를 갖도록 설정 가능

### 데이터 다중화

- 높은 가용성과 안정성을 확조하기 위해서 데이터를 N개 서버에 비동기적으로 다중화할 필요 있음
- N개의 서버를 선정하는 방법
    - 어떤 키를 해시 링 위에 배치한 후, 그 지점으로 링을 순회하면서 만나는 첫 N개 서버에 데이터 사본을 보관
    - 그런데 가상 노드를 사용하면 선택한 N개의 노드가 대응될 실제 물리 서버의 개수가 N보다 작아질 수 있음

- 이 문제를 피하려면 노드를 선택할 때 같은 물리 서버를 중복 선택하지 않도록 해야 함
- 같은 데이터 센터에 속한 노드는 정전, 네트워크 이슈, 자연재해 등의 문제를 동시에 겪을 가능성 있음
- 따라서 안정성을 담보하기 위해 데이터의 사본은 다른 센터의 서버에 보관하고, 센터들은 고속 네트워크로 연결

### 데이터의 일관성

- 여러 노드에 다중화된 데이터는 적절히 동기화 되어야 함
- 정족수 합의(Quorum Consensus) 프로토콜을 사용하면 읽기/쓰기 연산 모두에 일관성 보장 가능

## 일관성 모델

### 강한 일관성(strong consistency)

- 모든 읽기 연산은 가장 최근에 갱신된 결과를 반환 
- 클라이언트는 정대로 낡은 데이터를 보지 못함
- 가장 일반적인 방법은 모든 사본에 쓰기 연산 결과가 반영될 때까지 해당 데이터에 대한 읽기/쓰기를 금지
- 새로운 요청의 처리가 중단되기 때문에 고가용성 시스템에 부적합

### 약한 일관성(weak consistency)

- 읽기 연산은 가장 최근에 갱신된 결과를 반환하지 못할 수 있음

### 최종 일관성(eventual consistency)

- 약한 일관성의 한 형태로, 갱신 결과가 결국에는 모든 사본에 반영(동기화) 되는 모델
- 쓰기 연산이 병렬적으로 발생하면 시스템에 저장된 값의 일관성이 깨질 수 있는데, 이 문제는 클라이언트가 해결해야 함
- 클라이언트 측에서 데이터 버전 정보를 활용해 일관성이 깨진 데이터를 읽지 않도록 할수도 있음

## 비 일관성 해소 기법 : 데이터 버저닝

### 버저닝

- 데이터를 변경할 때마다 해당 데이터의 새로운 버전을 만듦
- 각 버전의 데이터는 불변 (immutable)

### 벡터 시계(vector clock)

- [서버, 버전]의 순서쌍을 데이터에 메단 것
- 어떤 버전이 선행, 후행인지, 충돌이 있는지 판별하는데 사용됨
- D([S1, v1], [S2, v2], ..., [Sn, Vn])와 같이 표현한다고 가정
    - D는 데이터

- 데이터 D를 서버 Si에 기록하면 아래 작업 가운데 하나를 수행해야 함
    - [Si, Vi]가 있으면 Vi를 증가시킴
    - 그렇지 않으면 새항목 [Si, 1]를 만듦
    - 어떤 버전 X와 Y 사이에 충돌이 있는지 보려면 Y의 벡터 시계 구성요소 가운데 X의 벡터 시계 동일 서버 구성요소보다 작은 값을 갖는 것이 있는지 보면 됨
    D([s0, 1], [s1, 2])와 D([s0, 2], [s1, 1])는 서로 충돌

### 단점
    
- 충돌 감지 및 해소 로직이 클라이언트에 들어가야 하므로, 클라이언트 구현이 복잡해짐
- [서버, 버전]의 순서쌍 개수가 굉장히 빨리 늘어남 
    - 이 문제를 해결하려면 길이에 임계치를 설정하고, 임계치 이상으로 길이가 길어지면 오래된 순서쌍을 벡터 시계에서 제거 
    - 하지만 이렇게 하면 버전 간 선후 관계가 정확하게 결정될 수 없기에 충돌 해소 과정의 효율성이 낮아짐
    - 하지만 실제로 문제가 벌어진 적은 없음

### 장애 감지

- 서버 A가 죽어도 바로 A 장애를 처리하지 않고, 보통 두 대 이상의 서버가 똑같이 서버 A의 장애를 보고해야 해당 서버에 실제로 장애가 발생했다고 간주
모든 노드 사이에 멀티캐스팅 채널을 구축하는 것이 서버 장애 감지하는 가장 손쉬운 방법
- 하지만 서버가 많을 때는 비효율적


## 시스템 아키텍처 다이어그램

- 클라이언트는 키-값 저장소가 제공하는 두 가지 get(key), put(key, value)와 통신
- 중재자(coordinator)는 클라이언트에세 키-값 저장소에 대한 프락시 역할을 하는 노드
- 노드는 안정해시의 해시 링 위에 분포
- 노드를 자동으로 추가, 삭제할 수 있도록 시스템은 완전히 분산
- 데이터는 여러 노드에 다중화
- 모든 노드가 같은 책임을 지므로 SPOF(Single Point of Failure)는 존재하지 않음
- 완전 분산 설계이므로, 모든 노드는 아래와 같은 기능 전부 지원해야 함
    - 클라이언트 API
    - 장애 감지
    - 데이터 충돌 해소
    - 장애 복구 매커니즘
    - 다중화
    - 저장소 엔진
    - 쓰기 경로
쓰기 요청이 커밋 로그 파일에 기록됨
데이터가 메모리 캐시에 기록됨
메모리 캐시가 가득차거나, 임계치에 도달하면 데이터는 디스크에 있는 SSTable에 기록됨.
(Sorted-Srting Table의 약어, <키, 값>의 순서쌍을 정렬된 리스트 형태로 관리하는 테이블)
읽기 경로
데이터가 메모리 캐시에 있는지부터  살핌
메모리에 없는 경우는 디스크에서 가져옴 
어느 SSTable에 찾는 키가 있는지 알아낼 효율적인 방법이 필요함
블룸 필터 사용됨
블룸 필터란?
블룸 필터(Bloom filter)는 원소가 집합에 속하는지 여부를 검사하는데 사용되는 확률적 자료 구조이다. 1970년 Burton Howard Bloom에 의해 고안되었다. 블룸 필터에 의해 어떤 원소가 집합에 속한다고 판단된 경우 실제로는 원소가 집합에 속하지 않는 긍정 오류가 발생하는 것이 가능하지만, 반대로 원소가 집합에 속하지 않는 것으로 판단되었는데 실제로는 원소가 집합에 속하는 부정 오류는 절대로 발생하지 않는다는 특성이 있다. 집합에 원소를 추가하는 것은 가능하나, 집합에서 원소를 삭제하는 것은 불가능하다. 집합 내 원소의 숫자가 증가할수록 긍정 오류 발생 확률도 증가한다.  -위키백과-
요약
목표/문제	기술
대규모 데이터 저장	안정 해시를 사용해 서버들에 부하 분산
읽기 연산에 대한 높은 가용성 보장	데이터를 여러 데이터센터에 다중화
쓰기 연산에 대한 높은 가용성 보장	버저닝 및 벡터 시계를 사용한 충돌 해소
데이터 파티션	안정 해시
점진적 규모 확장성	안정 해시
다양성(heterogeneity)	안정 해시
조정 가능한 데이터 일관성	정족수 합의 
일시적 장애 처리	느슨한 정족수 프로토콜과 단서 후 임시 위탁
영구적 장애 처리	머클 트리
데이터 센터 장애 대응 	여러 데이터 센터에 걸친 데이터 다중화