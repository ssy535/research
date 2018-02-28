# Hystrix
![Image](https://github.com/Netflix/Hystrix/wiki/images/hystrix-logo-tagline-640.png)

Netflix에서 개발해 사용중인 오픈소스로써, 분산 서비스 모델의 장애 대응 및 모니터링 관련 도구.

다음은 Hystrix에 대한 간략한 설명입니다.
>In a distributed environment, inevitably some of the many service dependencies will fail. Hystrix is a library that helps you control the interactions between these distributed services by adding latency tolerance and fault tolerance logic. Hystrix does this by isolating points of access between the services, stopping cascading failures across them, and providing fallback options, all of which improve your system’s overall resiliency.

장애내성<sup>fault tolerance</sup>과 지연대기 내성<sup>latency tolerance</sup>을 통해 MSA 환경에서 분산된 서비스들간의 통신을 돕는 라이브러리고 요약해볼수 있습니다.

**일명 회로 차단기!<sup>circuit Breaker</sup>**

## What is Hystrix For?
Hystrix은 다음과 같은 일들을 한다고 합니다.

* 장애와 지연에 대한 제어와 depenency로 부터 보호
* 복잡한 분산 시스템에서 계단식 오류를 멈춤
* 빨리 실패하고, 빨리 회복하자 ( 오류에 대해 )
* 실시간 모니터링 및 Alerting, 그리고 작동 제어가 가능
* 기능을 우아하게 저하시킨다<sup>gracefully degrade</sup>

## Hystrix가 해결하는 문제점
Hystrix은 MSA 환경과 같이 수십개의 디펜던시를 가진 복잡한 분산된 구조를 가진 어플리케이션에서 필연적으로 이들 디펜던시들 중에서 오류를 발생할 수 있습니다.

만일 어플리케이션에서 이런 외부 오류로 부터 격리되지 않는 다면, 큰 장애를 유발할 수 있습니다.

Hystrix은 이러한 문제점을 해결하는데 도움을 줍니다!

* * *
모든 요청에 대한 리퀘스트가 정상적으로 동작한다고 가정하면 다음과 같다.

![Image](https://github.com/Netflix/Hystrix/wiki/images/soa-1-640.png)

* * *
만일 요청에 대한 리퀘스트 중 특정 부분에 대한 서비스에 문제가 발생했을 경우, 전체 리스퀘트는 Block 될 수 있다.

여기서 빨간색으로 표현된 부분은 특정 서비스의 호출이 지연되고 있다는 뜻입니다.

![Image](https://github.com/Netflix/Hystrix/wiki/images/soa-2-640.png)

* * *

해당 상황에서 대용량 트래픽이 발생하면 모든 서버에서 모든 리소스가 몇 초 내에 포화 상태가 될 수 있습니다.

실패보다 더 심각한 것은 이런 서비스들 사이의 대기열이 증가하여 계단식 장애의 원인이 될 수 있습니다.

![Image](https://github.com/Netflix/Hystrix/wiki/images/soa-3-640.png)

 * * *

 Hystrix을 사용하여 기존 Dependency를 Wrapping하면 다음과 같은 다이어그램과 같은 아키텍처가 변형됩니다.

 Dependency에서 어떤 유형의 오류가 발생했을 때 어떤 대응을 할 것이닞 결제하는 대체 논리로 볼 수 있습니다.

![Image](https://github.com/Netflix/Hystrix/wiki/images/soa-4-isolation-640.png)

## 격리 (Isolation)

Hystrix은 벌크 헤드 패턴을 사용하여 서로의 종속성을 격리하고 이들 중 하나에 대한 동시 액세스를 제한합니다.

격리의 방법으로는 2가지가 있습니다.

1. Thread & Thread Pools
2. Semaphore ( or Counter )

### Thread & Thread Pools

Tomcat(Jetty, etc)과 같은 Thread Pool과는 별개로 별도의 Thread Pool의 thread로 HystrixCommand를 수행합니다.

Semaphore에 비해 상대적으로 연산 오버헤드<sup>computational overhead</sup>가 큼 ( 별도의 Thread Pool를 사용하기 때문에 )

하지만, 성능에 크게 영향을 미치지 않을 정도로 사소한 정도?

### Semaphore

Thread & Thread Pool 방식과는 틀리게 별도의 스레드를 만들지 않고, 각 서비스에 대한 동시 호출 수를 제한합니다.

Tomcat과 같은 Container의 Thread로 HystrixCommand를 수행


### 그렇다면 THread & Thread Pools과 Semaphore 둘중에 어느걸 사용해야 할까?

> The default, and the recommended setting, is to run HystrixCommands using thread isolation (THREAD) and HystrixObservableCommands using semaphore isolation (SEMAPHORE). Commands executed in threads have an extra layer of protection against latencies beyond what network timeouts can offer. Generally the only time you should use semaphore isolation for HystrixCommands is when the call is so high volume (hundreds per second, per instance) that the overhead of separate threads is too high; this typically only applies to non-network calls.

Hystrix은 격리 권장사항으로 Semaphore보단 Thread를 사용하라고 하고 있습니다.

Thread에서 커맨드 실행이 네트워크 타임아웃이 제공하는것을 넘어서 지연에 대한 보호를 할 수 있는 별도의 계층이 존재하기 때문이라고 합니다.

Semaphore를 고려해볼 상황으로는 일초에 몇백개의 요청이 오는 대용량<sup>so high volume</sup> 트래픽 환경이라고 합니다.
이러한 환경에서는 thread overhead의 비용이 너무 크게 때문이라고 하네요.

# 이제 구현!

스프링에서는 Netflix Hystrix을 사용할 수 있도록 이미 [가이드](https://spring.io/guides/gs/circuit-breaker/)를 제공해주고 있습니다.

Circuit Breaker<sup>회로 차단기</sup>라는 이름으로 되어있는데요, 가이드 대로 시작해보도록 할게요.

#### 준비물
* About 15 minutes
* A favorite text editor or IDE
* JDK 1.8 or later
* Gradle 2.3+ or Maven 3.0+
* You can also import the code straight into your IDE:
    * Spring Tool Suite (STS)
    * IntelliJ IDEA

