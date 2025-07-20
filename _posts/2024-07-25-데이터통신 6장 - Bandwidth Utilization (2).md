---
layout: post
title: 데이터 통신 6장 - Bandwidth Utilization (2)
date: 2024-07-25 19:20:23 +0900
category: DataCommunication
---
## 6.2 Spread Spectrum (SS, 확산 대역 방식)
- 무선통신 환경에서 군사목적으로 악의적인 접근에 방해를 주기 위해 만들어졌다.
- 큰 대역폭(LAN, WAN)에 맞추기 위해 조합되며, 목적은 도청 방지이다.
- 이를 위해 확산 대역 기술에 여분의 정보를 추가한다.
- 신호 $$B$$를 $$B_{\text{ss}}$$로 확장시킨다.

### 6.2.1 Frequency Hopping Spread Spectrum (FHSS)
- 발신지 $$M$$개의 신호를 서로 다른 반송파를 사용하여 변조한다.
- **Pseudorandom code generator**는 모든 hopping period $$T_h$$에 $$k$$-bit 패턴을 만든다.
- **Frequency table**은 이 hopping 기간 동안 사용되는 주파수를 찾기 위해 패턴을 쓴다.

- 홉핑 주파수의 수가 $$M$$이면, 우리는 같은 $$B_{\text{ss}}$$ 대역폭에서 $$M$$개의 채널로 다중화한다. 각 기지국은 1개의 홉핑 주파수 주기를 가진다. 총 \(M\)개의 기지국이 서로 다른 홉핑 주파수 주기를 가진다. 다중 **FSK**를 사용해 $$M$$개의 다른 기지국은 같은 $$B_{\text{ss}}$$를 쓴다. **FHSS**는 **FDM**과 비슷하다.

### 6.2.2 Direct Sequence Spread Spectrum (DSSS)
- 원래 신호의 **bandwidth**를 확장한다.
- 각 데이터 비트를 확산코드를 사용하여 $$n$$개의 비트(**chips**라 불림)로 대체한다.
- **b/w**는 원래 신호의 **b/w**보다 $$n$$배 커진다.

- 예시로 무선 LAN에 사용하는 **Barker sequence**(바커 코드)를 보자. 우리는 **폴라 NRZ** 인코딩을 쓴 칩 전환기의 오리지널 신호와 칩이 있다. $$10110111000$$을 사용한다. 1이면 뒤집고, 0이면 그대로 사용한다.