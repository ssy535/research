# Hystrix
![Image](https://github.com/Netflix/Hystrix/wiki/images/hystrix-logo-tagline-640.png)

Netflix에서 개발해 사용중인 오픈소스로써, 분산 서비스 모델의 장애 대응 및 모니터링 관련 도구.

다음은 Hystrix에 대한 간략한 설명입니다.
>In a distributed environment, inevitably some of the many service dependencies will fail. Hystrix is a library that helps you control the interactions between these distributed services by adding latency tolerance and fault tolerance logic. Hystrix does this by isolating points of access between the services, stopping cascading failures across them, and providing fallback options, all of which improve your system’s overall resiliency.

장애내성<sup>fault tolerance</sup>과 지연대기 내성<sup>latency tolerance</sup>을 통해 MSA 환경에서 분산된 서비스들간의 통신을 돕는 라이브러리.

 