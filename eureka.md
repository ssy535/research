# Eureka

### 들어가기

Netflix Eureka에 대해서 설명합니다.

Eureka의 구성요소에는 두가지가 존재합니다.

1. Eureka Server
2. Eureka Client

**Eureka Server** 는 로드밸런싱<sup>load balancing</sup>과 장애조치<sup>fail over</sup>의 목적을 둔 AWS 클라우드 환경에서 주로 사용되는 REST<sup>Representational State Transfer</sup> 기반의 미드티어 서버<sup>middle-tier servers</sup>이다.

**Eureka Client**는 Server와 상호작용을 좀더 쉽게 할 수 있는 자바 기반의 서비스이다.

### 언제 Eureka를 사용해야 될까?

거대한 Legacy 폭탄에서 벗어나기 위해 MSA를 적용하기로 한다!

하여, 하나의 어플리케이션 서버를 여러대의 API 서버로 분리했는데... 수많은 API 서버의 IP 주소와 포트 등을 관리했으면 하는 요구사항이 나온다.



A service registry is a phone book for your microservices.