# 커뮤니케이션 패턴

바운디드 컨텍스트 간 커뮤니케이션을 용이하게 하고, 애그리게이트 설계 원칙에 의해 부과된 제한 사항을 해결하고, 여러 시스템 컴포넌트에 걸쳐 비즈니스 프로세스를 조율한다.

<br/>

## 모델 변환

바운디드 컨텍스트는 유비쿼터스 언어 모델의 경계다.

사용자 - 제공자 관계에서 권력은 업스트림 또는 다운스트림 바운디드 컨텍스트가 갖는다. 

모델의 변환 로직은 스테이트리스 또는 스테이트풀이 될 수 있다.

상태를 보존하지 않는 스테이트리스 변환은 수신 또는 발신 요청이 발행할 때 즉석에서 발생하는 반면, 스테이트풀 변환은 상태 보존을 위해 데이터베이스를 사용하여 좀 더 복잡한 변환 로직을 다룰 수 있다. 

<br/>

#### 스테이트리스 모델 변환

스테이트리스 모델 변환을 소유하는 바운디드 컨텍스트는 프락시 디자인 패턴을 구현하여 수신과 발신 요청을 삽입하고 소스 모델을 바운디드 컨텍스트의 목표 모델에 매핑한다.

##### 동기

동기식 통신에 사용하는 모델을 변환하는 일반적인 방법은 바운디드 컨텍스트의 코드베이스에 변환 로직을 포함하는 것이다.

##### 비동기

비동기 통신에 사용하는 모델을 변환하기 위해 메시지 프락시를 구현할 수 있다.

메시지 프락시는 소스 바운디드 컨텍스트에서 오는 메시지를 구독하는 중개 컴포넌트다.

<br/>

#### 스테이트풀 모델 변환

더 중요한 모델 변환의 경우 스테이트풀 변환이 필요할 수 있다.

예를 들어, 원천 데이터를 집계 하거나 여러 개의 요청에서 들어오는 데이터를 단일 모델로 통합해야 하는 변환 메커니즘의 경우다.

<br/>

#### 들어오는 데이터 집계하기

바운디드 컨텍스트가 들어오는 요청을 집계하고 성능 최적화를 위해 일괄 처리에 관심이 있다고 가정해보자. 이 경우 동기와 비동기 요청 모두에 대해 집계가 필요할 수 있다.

소스 데이터를 집계하는 또 다른 유스케이스는 여러 개의 세분화된 메시지를 단일 메시지로 결합하는 것이다.

유입되는 데이터를 집계하는 모델 변환은 API 게이트웨이를 활용할 수 없으므로 변환 로직 자체에 영구 저장소가 필요하다.

일부 유스케이스에는 상용 제품을 사용하여 스테이트풀 변환을 위한 맞춤 제작 솔루션을 구현하지 않는 경우도 있다.

<br/>

#### 여러 요청 통합

다른 바운디드 컨텍스트를 포함하여 여러 요청에서 집계된 데이터를 처리해야 할 수도 있다. 이에 대한 일반적인 예는 사용자가 인터페이스가 여러 서비스에서 발생하는 데이터를 결합해야 하는 프론트엔드를 위한 백엔드 패턴이다.

또 다른 예는 여러 다른 컨텍스트의 데이터를 처리하고 이를 위해 복잡한 비즈니스 로직을 구현해야 하는 바운디드 컨텍스트다.

<br/>

## 애그리게이트 연동

애그리게이트가 시스템의 나머지 부분과 통신하는 방법 중 하나는 도메인 이벤트를 발행하는 것이다. 외부 컨포넌트는 이러한 도메인 이벤트를 구독하고 해당 로직을 실행할 수 있다.

<br/>

#### 아웃박스

아웃박스 패턴은 다음 알고리즘을 사용하여 도메인 이벤트의 안정적인 발행을 보장한다.

-  업데이트된 애그리게이트의 상태와 새 도메인 이벤트는 모두 동일한 원자성 트랜잭션으로 커밋된다.
-  메시지 릴레이는 데이터베이스에서 새로 커밋된 도메인 이벤트를 가져온다.
-  릴레이는 도메인 이벤트를 메시지 버스에 발행한다.
-  성공적으로 발행되면 릴레이는 이벤트를 데이터베이스에 발행한 것으로 표시하거나 완전히 삭제한다.

관계형 데이터베이스를 사용할 때 두 개의 테이블에 원자적으로 커밋하고 메시지를 저장하기 위한 전용 테이블을 사용하는 데이터베이스의 기능을 활용하는 것이 좋다.

<br/>

#### 발행되지 않은 이벤트 가져오기

발행 릴레이는 풀 기반 또는 푸시 기반 방식으로 새 도메인 이벤트를 가져올 수 있다.

##### 풀: 발행자 폴링

릴레이는 발행되지 않은 이벤트에 대해 데이터베이스를 지속해서 질의할 수 있다. 지속적인 폴링으로 인한 데이터베이스 부하를 최소화 하려면 적절한 인덱스가 있어야 한다.

##### 푸시: 트랜잭션 로그 추적

데이터베이스의 기능을 활용하여 새 이벤트가 추가될 때마다 발행 릴레이를 호출할 수 있다.

아웃 박스 패턴은 적어도 한 번은 메시지 배달을 보장한다는 점에 유의해야 한다.

<br/>

#### 사가

핵심 애그리게이트 설계 원칙 중 하나는 각 트랜잭션을 애그리게이트의 단일 인스턴스로 제한하는 것이다.

여러 애그리게이트에 걸쳐 있는 비즈니스 프로세스를 구현해야 하는 경우가 있다. 이렇게 하면 애그리게이트의 경계를 신중하게 고려하고 응집된 비즈니스 기능 집합을 캡슐화할 수 있다.

사가는 오래 지속되는 비즈니스 프로세스디. 사가가 몇 초에 몇 년까지 계속될 수 있지만, 반드시 시간 측면이 아니라 트랜잭션 측면에서 보는 것이다.

도메인 이벤트 발행의 경우 사가 상태의 전환을 커맨드 실행과 분리하면 프로세스가 어느 단계에서 실패하더라도 커맨드가 안정적으로 실행되 수 있다.

<br/>

#### 일관성

사가 패턴이 다중 컴포넌트의 트랜잭션을 조율하기는 하지만 관련된 컴포넌트의 상태는 궁극적으로 일관성을 갖는다. 그리고 사가가 결국 관련 커맨드를 실행한다고 해도 두 개의 트랜잭션은 원자적으로 간주되지 않으므로 모두 성공하거나 실패할 수 있다.

##### *애그리게이트 경계 내의 데이터만 강한 일관성을 가진다. 외부의 모든 것은 궁극적으로 일관성을 갖는다.*

<br/>

#### 프로세스 관리자

사가 패턴은 단순하고 선형적인 흐름을 관리한다.

사가 구현을 시연하는 데 사용한 예제에서도 실제로 이벤트와 커맨드를 간단하게 일치시켰다.

- CampaignActivated 이벤트와 PublishingService.SubmitAdvertisement 커맨드
- PublishingConfirmed 이벤트와 Campaign.TrackConfirmation 커맨드
- PublishingRejected 이벤트와 Campaign.TrackRejection 커맨드

프로세스 관리자 패턴은 비즈니스 로직 기반 프로세스를 구현하기 위한 것이다.

프로세스 관리자는 시퀀스의 상태를 유지하고 다음 처리 단계를 결정하는 중앙 처리 장치로 정의한다.

프로세스 관리자는 명시적으로 인스턴스화해야 한다.

<br/>

## 결론

시스템 컴포넌트를 연동하기 위한 다양한 패턴을 배웠다.

충돌 방지 계층 또는 오픈 호스트 서비스를 구현하는 데 사용할 수 있는 모델 변환 패턴을 살펴보았다.

아웃박스 패턴은 애그리게이트의 도메인 이벤트를 발행하는 안정적인 방법이다. 다른 프로세스 실패에 직면해도 도메인 이벤트를 항상 발행한다.

사가 패턴은 간단란 교차 컴포넌트 비즈니스 프로세스를 구현하는 데 사용할 수 있다.

프로세스 관리자 패턴을 사용하여 좀 더 복잡한 비즈니스 프로세스를 구현할 수 있다. 

두 패턴 모두 도메인 이벤트에 대한 비동기식 반응과 커맨드 실행에 의존한다.

<br/>

# 느낀점

시스템 컴포넌트를 연동하기 위한 다영한 패턴을 알게 되었다.