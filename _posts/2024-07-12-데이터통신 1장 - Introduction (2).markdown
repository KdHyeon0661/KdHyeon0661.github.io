---
layout: post
title: 데이터 통신 1장 - Introduction (2)
date: 2024-07-12 19:20:23 +0900
category: DataCommunication
---
# 1.3 Network Types

## 1.3.1 Local Area Network (LAN)

![DC LAN](/../public/postImg/2024-07-12-DC_LAN.JPG)

- **정의**: LAN은 비교적 좁은 지역 내에서 컴퓨터, IP 전화기 등의 장비를 연결하는 네트워크입니다. 학교, 회사, 가정 내에서 사용되며, 여러 장치 간의 1대1 직접 연결이 아닌 공유기나 스위치와 같은 네트워크 장비를 통해 연결됩니다.
- **특징**: LAN은 이더넷이라는 프로토콜을 사용하며, 낮은 유지보수 비용으로 운영될 수 있습니다. 이를 통해 고속 데이터 전송이 가능합니다.

## 1.3.2 Wide Area Network (WAN)

- **정의**: WAN은 LAN과 LAN 사이를 넓은 지역에 걸쳐 연결한 네트워크입니다. LAN은 지역 내 네트워크이지만, WAN은 이를 연결하여 더 큰 네트워크를 형성합니다.
  - **Point-to-Point WAN**: 두 통신 장치가 케이블 또는 공기를 통해 연결된 네트워크 상태를 말합니다.
  ![DC P2P WAN](/../public/postImg/2024-07-12-DC_P2P_WAN.JPG)
  - **Switched WAN**: 두 개 이상의 종단 장치가 다수의 스위치를 통해 연결된 네트워크입니다. 현재의 글로벌 연결 사회에서 여러 Point-to-Point WAN이 스위치를 통해 결합된 형태입니다.
  ![DC Switched WAN](/../public/postImg/2024-07-12-DC_Switched_WAN.JPG)
  - **Internetwork**: 두 개 이상의 네트워크를 서로 연결하여 통신이 가능하도록 한 방식입니다. 이를 통해 '네트워크들의 네트워크(A Network of Networks)'라고 불리며, 인터넷의 기본적인 구성 방식입니다.
  ![DC Internetwork](/../public/postImg/2024-07-12-DC_Internetwork.JPG)

## 1.3.3 Switching

스위칭은 네트워크 내에서 데이터를 다른 네트워크로 전송할 때 필요한 기술입니다.

- **Circuit-Switched Network (회선교환방식)**:

  ![DC Circuit-Switched Network](/../public/postImg/2024-07-12-DC_circuit-switched_Network.JPG)

  - 예시로, A가 B에게 전화를 걸면 B가 전화를 받을 때까지 회선이 독점되며, B가 다른 사람과 통화 중이라면 통화 중임을 알립니다. A와 B가 통화할 수 있는 안정적인 연결이 완료되면 통신이 지속적으로 이루어집니다.

  - **장점**: 
   -- 대용량 데이터 고속 전송
   -- 고정 대역폭(Bandwidth) 사용
   -- 접속에는 긴 시간이 소요되나 접속 이후에는 접속이 항상 유지되어 전송 지연이 없으며, 데이터 전송률이 일정함
   -- 안정적인 연결
   -- 연속적인 전송에 적합

  - **단점**: 
   -- 고비용
   -- 비효율적인 회선 이용률
   -- 연결된 두 장치는 반드시 같은 전송률과 같은 기종 사이에 송수신이 요구됨(다양한 속도를 지닌 개체 간 통신 제약)
   -- 실시간 전송보다 에러 없는 데이터 전송이 요구되는 구조에서는 부적합하다
   -- 속도나 코드의 변환이 불가능(교환망 내에서의 에러 제어 기능이 어렵다)

- **Packet-Switched Network (패킷교환방식)**:

  ![DC Packet-Switched Network](/../public/postImg/2024-07-12-DC_Packet-Switched_Network.JPG)

  - 데이터 전송 시 메시지를 일정한 크기의 패킷으로 분해하여 송신하고, 수신 측에서 이를 다시 원래의 메시지로 조립하는 방식입니다.
  - **장점**:
   -- 회선 이용률이 높고, 속도 변환, 프로토콜 변환이 가능하며, 음성 통신도 가능하다
   -- 고 신뢰성 : (경로선택, 전송 여부 판별 및 장애 유무 등) 상황에 따라 교환기 및 회선 등의 장애가 발생하더라도 패킷의 우회 전송이 가능하므로 전송의 신뢰성이 보장된다.
   -- 고품질 : 디지털 전송이므로 인접 교환기 간 또는 단말기와 교환기 간에 전송 오류검사를 실시 하여 오류 발생 시 재전송이 가능함
   -- 고효율 : 다중화를 사용하므로 사용 효율이 좋음
   -- 이 기종 단말장치 간 통신 : 전송 속도, 전송 제어 절차가 다르더라도 교환망이 변환 처리를 제공하므로 통신이 가능
  - **단점**:
   -- 경로에서의 각 교환기에서 다소의 지연이 발생한다
   -- 이러한 지연은 가변적이다. 즉, 전송량이 증가함에 따라 지연이 더욱 심할 수도 있다.


## 1.3.4 The Internet

- **internet (소문자)**: 2개 이상의 네트워크가 공용 프로토콜을 통해 통신하는 것을 의미합니다.
- **Internet (대문자)**: 수천 개의 상호 연결된 네트워크가 TCP/IP 프로토콜을 사용하여 전 세계적으로 호스트 간 통신을 가능하게 하는 거대한 네트워크입니다.
- **ISP (Internet Service Provider)**: 네트워크 및 백본(인터넷 트래픽을 처리하는 주요 네트워크 경로)을 제공하는 회사입니다.

## 1.3.5 Accessing the Internet

인터넷에 접속하는 다양한 방식들입니다.

- **전화 네트워크**:
  - **Dial-up (전화접속 서비스)**: 전화선을 이용해 인터넷에 접속하는 방식.
  - **DSL (디지털 가입자 회선)**: 전화망을 통해 고속 데이터 전송을 가능하게 하는 기술.
- **케이블 네트워크 사용**: 케이블을 통한 인터넷 연결.
- **무선 네트워크 사용**: Wi-Fi와 같은 무선 기술을 통해 인터넷에 접속.
- **직접 인터넷 연결**: 조직 또는 집에서 전용 회선을 통해 인터넷에 직접 연결.

# 1.4 Internet History

## 1.4.1 Early History

- **패킷 스위칭 이론**: 패킷 스위칭 이론은 MIT의 Leonard Kleinrock이 1961년에 처음 제안한 개념입니다. 이는 데이터 전송을 패킷이라는 작은 단위로 나누어 효율적으로 전송하는 방식입니다.
- **ARPANET**: 1969년에 미국 국방부의 ARPA가 세계 최초의 패킷 스위칭 네트워크를 구축했습니다. ARPANET은 현재 인터넷의 원형이 되었으며, 상호 연결된 네트워크의 기초가 되었습니다.

## 1.4.2 Birth of the Internet

- **TCP/IP 프로토콜**: TCP(Transmission Control Protocol)와 IP(Internet Protocol)는 인터넷의 핵심 프로토콜입니다. TCP는 데이터의 신뢰성을 보장하고, IP는 데이터의 경로를 결정합니다.
- **MILNET**: 군사용 인터넷으로, ARPANET과의 분리를 통해 보안이 강화된 네트워크입니다.
- **CSNET**: 1981년에 미국에서 시작된 컴퓨터 네트워크로, ARPANET에 직접 연결할 수 없는 학술 및 연구 기관을 위한 네트워크였습니다.
- **NSFNET**: 1985년부터 1995년까지 미국 NSF(National Science Foundation)가 후원한 네트워크로, 연구와 교육 네트워킹을 촉진하기 위해 설계되었습니다.
- **ANSNET**: NSFNET 백본 서비스를 지원하기 위해 1990년에 설립된 네트워크 인프라입니다.

## 1.4.3 Internet Today

- **월드 와이드 웹(WWW)**: 1989년 팀 버너스-리가 제안한 개념으로, 인터넷을 통해 정보를 공유할 수 있는 전 세계적 정보 공간을 의미합니다.
- **멀티미디어**: 소리, 영상, TV 등 다양한 콘텐츠를 인터넷을 통해 전송하고 공유하는 방식입니다.
- **P2P 어플리케이션**: 사용자가 중앙 서버 없이 직접 서로의 컴퓨터를 연결하여 데이터를 주고받는 애플리케이션으로, 파일 공유와 같은 분야에서 사용됩니다.

# 1.5 Standards and Administration

## 1.5.1 Internet Standards

- **인터넷 표준 (Internet Standard, STD)**: 인터넷에서 사용되는 기술과 방법론을 표준화한 규격입니다.
- **RFC (Request for Comments)**: 인터넷 기술과 관련된 연구, 혁신 등을 담은 문서로, 인터넷 표준의 기반이 됩니다.

  ![DC RFC](/../public/postImg/2024-07-12-DC_RFC.JPG)

- **Maturity Levels (성숙도 단계)**:
  - **Proposed Standard**: 인터넷 커뮤니티에 관심을 끌기 위한 초기 단계의 표준.
  - **Draft Standard**: 독립적인 실험 결과를 바탕으로 한 다음 단계.
  - **Internet Standard**: 성공적인 조사 후 표준으로 인정받은 상태.
  - **Historic**: 더 이상 사용되지 않거나 대체된 표준.
  - **Experimental**: 연구 단계에 있는 실험적인 기술.
  - **Informational**: 인터넷 커뮤니티에 유용한 정보를 제공하는 문서.
- **Requirement Levels**:
  - **Required**: 모든 인터넷 시스템에서 필수적으로 구현해야 하는 기술(IP, DHCP 등).
  - **Recommended**: 필수는 아니지만 유용한 기술(FTP, TELNET 등).
  - **Elective**: 필요하지 않지만 사용하면 이득이 되는 기술.
  - **Limited Use**: 특정 상황에서만 사용하는 기술.
  - **Not Recommended**: 일반적으로 부적절하여 추천되지 않는 기술.

## 1.5.2 Internet Administration

![DC Internet Administration](/../public/postImg/2024-07-12-DC_Internet_Administration.JPG)

- **Internet Society (ISOC)**: 인터넷의 최상위 관리 기관으로, 인터넷과 관련된 모든 활동을 총괄합니다.
- **Internet Architecture Board (IAB)**: ISOC의 기술적 조언자로서, 인터넷의 기술적 관리와 표준을 담당합니다.
- **Internet Engineering Task Force (IETF)**: 인터넷 프로토콜 및 기술 연구를 수행하는 조직으로, 미래의 인터넷 기술을 개발합니다.
- **Internet Research Task Force (IRTF)**: 인터넷 기술의 최신 동향을 연구하고, 현재 사용 중인 인터넷 기술을 발전시키기 위한 연구를 진행합니다.
- **IANA (Internet Assigned Numbers Authority)**: 전 세계적으로 IP 주소, 도메인 이름, 포트 번호와 같은 인터넷 자원을 관리하는 기관입니다.