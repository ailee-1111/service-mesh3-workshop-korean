# 마이크로서비스에 대한 간략한 소개 (A Brief Introduction to Microservices)
마이크로서비스 아키텍처(Microservice Architecture)라고도 불리는 마이크로서비스(Microservices)는 애플리케이션을 느슨하게 결합된 서비스들의 집합으로 구성하는 소프트웨어 개발 기법입니다. 마이크로서비스 아키텍처는 복잡한 애플리케이션의 지속적인 인도/배포/확장(continuous delivery/deployment/scaling)을 가능하게 합니다.

## 왜 마이크로서비스인가요?
민첩성(Agility)입니다. 애플리케이션 업데이트를 더 빠르게 제공하고, 버그를 더 쉽게 격리하고 수정할 수 있습니다. 마이크로서비스 아키텍처를 올바르게 구현하면 다음과 같은 소프트웨어의 중요한 여러 비기능적 요구사항을 충족하는 데 도움이 됩니다:

* 확장성 (scalability)
* 성능 (performance)
* 신뢰성 (reliability)
* 회복탄력성 (resiliency)
* 확장 가능성 (extensibility)
* 가용성 (availability)

## 서비스 메시(Service Mesh)란 무엇인가요?
Red Hat OpenShift Service Mesh는 서비스 메시에 대한 동작 인사이트와 운영 제어 기능을 제공하여, 마이크로서비스 애플리케이션을 연결하고 보안을 확보하며 모니터링할 수 있는 일관된 방법을 제공하는 플랫폼입니다.

서비스 메시라는 용어는 분산 마이크로서비스 아키텍처에서 애플리케이션을 구성하는 마이크로서비스의 네트워크와 해당 마이크로서비스 간의 상호 작용을 설명합니다. 서비스 메시의 규모와 복잡성이 증가할수록 이를 이해하고 관리하기가 더 어려워질 수 있습니다.

오픈소스 Istio 프로젝트를 기반으로 하는 Red Hat OpenShift Service Mesh는 서비스 코드의 변경 없이 기존 분산 애플리케이션에 투명한 레이어를 추가합니다. 환경 전체에 마이크로서비스 간의 모든 네트워크 통신을 가로채는 특별한 사이드카(sidecar) 프록시를 배포하여 서비스에 Red Hat OpenShift Service Mesh 지원을 추가합니다. 컨트롤 플레인(control plane) 기능을 사용하여 서비스 메시를 구성하고 관리합니다.

Red Hat OpenShift Service Mesh는 디스커버리(discovery), 부하 분산(load balancing), 서비스 간 인증, 장애 복구, 메트릭 및 모니터링을 제공하는 배포된 서비스 네트워크를 쉽게 구축할 수 있는 방법을 제공합니다. 서비스 메시 또한 A/B 테스트, 카나리(canary) 릴리스, 속도 제한(rate limiting), 액세스 제어 및 엔드투엔드(end-to-end) 인증을 포함하여 더 복잡한 운영 기능을 제공합니다.
