---
title: _chems test
tags:
  - chemistry, test0
---

 AWS 클라우드 환경에 서버를 생성하고 싶을 때, 아무 설정 없이 인스턴스만 툭 하고 생성할 수도 있다. 이 경우 AWS 에서 Default로 제공하는 VPC 네트워크 환경에 인스턴스가 올라가게 된다.
 뭐 사용하는데 전혀 상관은 없지만, 원하는 대역의 사설 IP와 Subnet등을 사용하려면 VPC를 새로 생성해야 한다.
 이런 경우 기본적으로는 AWS VPC, Routing Table, Internet Gateway가 필요하다!

## AWS VPC
* VPC는 Virtual Private Cloud (가상 사설 클라우드) 의 약자이다.
* 클라우드 환경 위에 **내부망**을 생성할 수 있고 이를 사내망처럼 사용하게 된다. 이를 통해 내부 IP 사용으로 외부에서의 침입을 차단하고 (public ip를 설정하면 외부와의 연결도 가능하다.) 좀더 안전한 환경 구성에 도움이 된다.
* Private Network을 생성할 때의 IP 대역은 [rfc1918](https://tools.ietf.org/html/rfc1918){:target="_blank"}을 지키며 생성해야 한다. 간단한 설명과 CIDR은 아래와 같다.
