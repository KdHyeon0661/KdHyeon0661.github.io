---
layout: post
title: 데이터 통신 2장 - Network Model
date: 2024-07-13 19:20:23 +0900
category: DataCommunication
---
### 2.1 Protocol Layering

#### 2.1.1 Scenarios

![DC Scenarios](/../assets/img/network/2024-07-13-DC_Scenarios.jpg)

- Layering을 하게 되면 한 층에서 Protocol이 수행해야하는 행동 범위가 많이 줄어들면서 각자 자신의 임무만 명확히 수행하게 될 수 있는 장점이 생깁니다.
 따라서 Protocol Layering이라는 것은 복잡한 업무를 작은 일의 단위로 나누어 간단한 업무를 부담하게 하는 것으로 볼 수 있습니다.
 비록 간단한 업무가 복잡해질 수는 있더라도 Protocol Layering을 했을 때 다음과 같은 기대 효과를 얻을 수 있습니다.
  - 프로토콜 간 기능이 독립적이 되어 각 프로토콜을 독립적으로 수정할 수 있습니다.
  - 각 프로토콜의 내부 동작을 "블랙박스"처럼 다룰 수 있어, 내부 작동 원리를 몰라도 입력에 따른 출력을 확인할 수 있습니다.

#### 2.1.2 Principles of Protocol Layering
- **Bidirectional (양방향성)**: 통신이 필요한 경우, 각 계층은 송신과 수신을 모두 수행할 수 있어야 합니다. (즉, 송신자와 수신자의 역할을 모두 처리할 수 있어야 합니다.)
- **일관된 계층 구조**: 통신하는 두 객체의 하위 계층들이 동일하게 쌓여 있어야 합니다.

#### 2.1.3 Logical Connections(논리적 연결)

![DC Logical Connections](/../assets/img/network/2024-07-13-DC_Logical_Connections.jpg)

- 위 원칙에 따라 논리적 연결은 각 계층 간 종단 간 통신을 가능하게 하여 계층 간 올바른 상호 작용을 보장합니다.

### 2.2 TCP/IP protocol suite

#### 2.2.1 Layered Architecture

![DC Layered Architecture](/../assets/img/network/2024-07-13-DC_Layered_Architecture.jpg)

Layer 1~5까지 전부 포함되어있는 시스템은 양 끝단에 있는 단말기(터미널)밖에 없습니다. 적어도 중간에서 그들 사이를 이어주는 단계에서는 Application 계층을 포함해서는 안됩니다. Application 계층에서는 Data가 Information으로 바뀌는데 중간 단계에서 이러한 작업이 일어나면 개인의 정보가 유츌되거나 도청될 우려가 있습니다.

#### 2.2.2 Layers in the TCP/IP Protocol Suite

![DC Layered Architecture](/../assets/img/network/2024-07-13-DC_Logical_Connections_between_TCPIP.jpg)

- 그림으로 알 수 있듯이 application, transport, network는 end-to-end(종단간) 연결이고, data link와 physical은 hop-to-hop(기기간) 연결이다. 여기서 hop은 라우터나 호스트를 의미합니다.

#### 2.2.3 Description of Each Layer

![DC Layer](/../assets/img/network/2024-07-13-DC_Layers.jpg)

##### Physical Layer (물리 계층)
- 매체를 통해 비트 데이터를 스트림 형태로 전송하고 받으며, 기계적·전기적 특성을 제공합니다.
- 데이터 링크 계층으로부터 받은 프레임을 비트 스트림으로 변환하여 전송합니다.
- 비트를 **전기 신호**, **빛 신호**, **무선 신호**로 처리하는데, 한 신호에 여러 비트를 포함시켜 처리 횟수를 줄입니다.

##### Data Link Layer (데이터 링크 계층)
- 비트 데이터나 패킷을 프레임화하여 기기 간 **Hop-to-Hop** 전달을 담당합니다.
- Hop-to-Hop 전달이란 **인접 노드**로 데이터를 전달하는 것으로, **MAC 주소**를 사용하여 기기를 식별합니다.
- 네트워크 계층의 인터페이스로 작동하며, 인접 기기로의 데이터 전송을 **무결성** 상태로 보장하여 상위 계층이 **Error-Free** 상태로 가정할 수 있게 합니다.
- **흐름 제어**, **에러 감지 및 수정**이 가능하나, 이는 전송 계층의 기능(인터넷 상에서 논리적 연결 상태에서의 계층 간 전달)과 다르게 네트워크 내 기기간에서 발생합니다.

##### Network Layer (네트워크 계층)
- 패킷 형태로 송신지에서 목적지까지 데이터를 운송하며 **라우팅**과 관련이 있습니다.
- **IP 주소**라는 일종의 논리적 주소를 사용하여 송신-수신지를 구별하고, 네트워크 간 패킷 송수신을 담당합니다.
- IP는 **비연결 지향적**이며 **Best Effort Delivery**로 신뢰성 보장이 없으나 최선을 다해 전달합니다.
- 공용 IP와 같이 사용되는 IP 주소는 사용자에 대해서 독특하고 보편적으로 구별 가능하게 해줍니다.

##### Transport Layer (전송 계층)
- 신뢰성 보장을 위해 정해진 순서대로 메세지를 구성하여 넘겨주어, **에러 감지 및 수정** 기능을 수행합니다.
- 기본 기능은 **Unreliable Data Delivery**이며 신뢰성을 보장해주지 않은채로 전달하는 것이 기본이며, 대표적으로 **UDP** 프로토콜이 있습니다.
- **TCP**와 같은 연결 지향 프로토콜의 주요 기능은 다음과 같습니다:
  - **연결 관리** (예: 3-way hankshake)
  - **연결 다중화** (포트 사용)
  - **세그먼트화** (SCTP의 Chuck)
  - **신뢰성 있는 전송**
  - **흐름 제어** (수신 측), **혼잡 제어** (송신 측)

##### Application Layer (응용 계층)
- 세션 생성, 관리, 종료를 담당하는 계층으로, 두 호스트 간의 응용 프로세스들 사이에 세션을 열고 닫고 관리하는 메커니즘을 수행합니다.
- **VoIP**에서의 **SIP**와 같이 세션 계층의 역할을 합니다.
- **대화 제어** 및 **동기화** 기능을 통해 메시지 송수신의 순서를 조정하고 스트림 데이터의 동기화 지점을 기록할 수 있습니다.
- 이 계층에는 **WWW, HTTP, SMTP, FTP, TELNET, SSH, SNMP, DNS, IGMP** 등이 포함됩니다.

#### 2.2.4 Encapsulation and Decapsulation(캡슐화와 역캡슐화)

[DC capsulation](/../assets/img/network/2024-07-13-DC_Capsule.jpg)

- 처음 application에서 메시지가 생성되면 하위 계층으로 내려가면서 캡슐화로 감쌉니다. 중간에 라우터에서는 캡슐화를 풀고 다시 캡슐화를 한니다. 도착지에 오면 캡슐화를 풀면서 상위 계층에 올라갑니다. 도착지의 application에 메세지가 들어옵니다.

#### 2.2.5 Addressing
 
![DC Addressing](/../assets/img/network/2024-07-13-DC_Addressing.jpg)

- link-layer address는 **MAC**이라 부르는데 네트워크(LAN or WAN) 안의 호스트나 라우터를 구별짓는 주소입니다. 
- network layer의 주소는 인터넷 범위에서 유일한 장치임을 구별하기 위해서있습니다.
- transport layer의 포트 주소는 동시에 돌아가는 여러 프로그램을 구별하기 위해 있는 주소입니다.
- application layer의 주소는 **프로그램의 이름**입니다.

#### 2.2.6 Multiplexing and Demultiplexing
 
![DC Multiplexing](/../assets/img/network/2024-07-13-DC_Multiplexing.jpg)

- multiplexing : 출발지에서 이뤄지며, 상위 계층에서 패킷을 캡슐화하는 것입니다.
- demultiplexing : 도착지에서 이뤄지며, 상위 계층 프로토콜에서 패킷을 캡슐화를 해제하는 것입니다.

### 2.3 The OSI Model

![DC OSI 7](/../assets/img/network/2024-07-13-DC_OSI7.jpg)

- ISO에서 1947년에 제정한 것이다. Open Systems Interconnection(OSI)라 불립니다.

#### 2.3.1 OSI vs TCP/IP

![DC OSI vs TCP/IP](/../assets/img/network/2024-07-13-DC_OSI7vsTCPIP.jpg)

  - 차이점
    - tcp/ip의 transport layer에 프로토콜이 많은데 일부 프로토콜이 OSI에서 세션 계층으로 넘어갔기 때문입니다.
    - application layer 계층이 OSI에서는 세분화가 되었습니다.

#### 2.3.2 Lack of OSI Model’s Success

- 문제점
    - OSI 계층을 만드는데 돈과 시간이 많이 듭니다.
    - 일부 계층은 OSI가 제대로 증명하지 못했습니다.
    - 협회가 다른 기준으로 적용해 구현되어서 고성능을 보여주지 못합니다.
