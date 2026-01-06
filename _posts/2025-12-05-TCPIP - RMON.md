---
layout: post
title: TCPIP - RMON
date: 2025-12-05 17:25:23 +0900
category: TCPIP
---
# SNMP 원격 네트워크 모니터링 (RMON)

## 1. RMON 개요 및 역사

### 1.1 RMON의 등장 배경

1990년대 초반, 네트워크 관리의 패러다임은 개별 장치 중심에서 네트워크 세그먼트 전체의 모니터링으로 전환되고 있었습니다. 기존 SNMP는 장비별 통계는 제공했지만, 네트워크 트래픽 패턴, 성능 추세, 문제 진단에 필요한 링크 수준 정보는 부족했습니다.

**기존 SNMP의 한계:**
1. **장치 중심**: 개별 장비 정보만 제공
2. **실시간 부족**: 폴링 기반으로 즉시 감지 어려움
3. **진단 정보 부족**: 문제 원인 분석에 필요한 데이터 미흡
4. **대역폭 소비**: 빈번한 폴링으로 네트워크 부하 증가

### 1.2 RMON의 혁신적 접근

RMON(Remote Network Monitoring)은 1991년 RFC 1271로 처음 정의되었으며, 네트워크 세그먼트 전체를 모니터링하는 새로운 패러다임을 제시했습니다.

```
RMON의 핵심 개념:
+---------------------+-----------------------------------------------+-------------------+
| 개념                | 설명                                           | 혁신성            |
+---------------------+-----------------------------------------------+-------------------+
| 세그먼트 모니터링   | 개별 장치가 아닌 네트워크 세그먼트 전체 모니터 | 패러다임 전환     |
| 로컬 처리           | 데이터 수집과 분석을 로컬에서 수행            | 대역폭 절감       |
| 임계값 기반 알림    | 사전 정의 조건에서 자동 알림                  | 사전 예방적 관리  |
| 프로토콜 분석       | 트래픽 패턴과 프로토콜 사용 분석              | 심층 진단 가능    |
| 역사적 데이터       | 시간에 따른 추세 분석 가능                    | 용량 계획 지원    |
+---------------------+-----------------------------------------------+-------------------+
```

### 1.3 RMON 발전 역사

```
RMON 표준 진화:
1991년: RMON1 표준화 (RFC 1271) - 데이터 링크 계층 중심
1997년: RMON1 개정 (RFC 1757) - 이더넷 특화
1997년: RMON2 표준화 (RFC 2021) - 네트워크 계층 이상 확장
2003년: RMON2 개정 (RFC 4502) - 기능 강화
2000년대: RMON MIB 확장 및 벤더별 구현
현재: sFlow, NetFlow와 통합된 현대적 모니터링으로 진화
```

## 2. RMON 아키텍처

### 2.1 RMON 구성 요소

RMON은 전통적인 SNMP 아키텍처를 확장하여 새로운 구성 요소를 도입했습니다.

```
RMON 계층적 모델
+---------------------+       +---------------------+       +---------------------+
|   관리 스테이션     |       |     RMON 프로브     |       |   네트워크 세그먼트  |
|   (NMS)             |       |   (모니터링 장치)   |       |                     |
+---------------------+       +---------------------+       +---------------------+
| • 데이터 분석       |       | • 패킷 캡처         |       | • 실제 트래픽       |
| • 보고 생성         |<------| • 통계 계산         |<------| • 네트워크 활동     |
| • 임계값 관리       | SNMP  | • 임계값 검사       | 스니핑|                     |
| • 장애 진단         |       | • 이벤트 생성       |       |                     |
+---------------------+       +---------------------+       +---------------------+

RMON 프로브 유형:
1. 독립형 프로브: 전용 모니터링 장비
2. 내장형 프로브: 라우터/스위치에 통합
3. 소프트웨어 프로브: 서버에 설치
4. 가상 프로브: 가상화 환경용
```

### 2.2 RMON MIB 구조

RMON은 SNMP MIB를 확장하여 새로운 객체 그룹을 정의합니다.

```
RMON MIB 계층 구조 (RFC 2819)
rmon(16) 아래 주요 그룹:
├── statistics(1)         -- 이더넷 통계
├── history(2)            -- 역사적 데이터
├── alarm(3)              -- 임계값 알림
├── hosts(4)              -- 호스트 테이블
├── hostTopN(5)           -- 상위 N개 호스트
├── matrix(6)             -- 호스트 간 통신 매트릭스
├── filter(7)             -- 패킷 필터링
├── capture(8)            -- 패킷 캡처
├── event(9)              -- 이벤트 로깅
└── tokenRing(10)         -- 토큰링 확장 (RFC 1513)
```

## 3. RMON1 그룹 상세 분석

### 3.1 통계 그룹 (statistics)

네트워크 세그먼트의 기본 통계 정보를 수집하고 분석합니다.

#### etherStatsTable 구조
```asn.1
etherStatsTable OBJECT-TYPE
    SYNTAX      SEQUENCE OF EtherStatsEntry
    MAX-ACCESS  not-accessible
    STATUS      current
    DESCRIPTION 
        "각 모니터링되는 이더넷 세그먼트에 대한 통계를 포함하는 테이블"
    ::= { statistics 1 }

etherStatsEntry ::= SEQUENCE {
    etherStatsIndex            INTEGER,
    etherStatsDataSource       OBJECT IDENTIFIER,
    etherStatsDropEvents       Counter32,
    etherStatsOctets           Counter32,
    etherStatsPkts             Counter32,
    etherStatsBroadcastPkts    Counter32,
    etherStatsMulticastPkts    Counter32,
    etherStatsCRCAlignErrors   Counter32,
    etherStatsUndersizePkts    Counter32,
    etherStatsOversizePkts     Counter32,
    etherStatsFragments        Counter32,
    etherStatsJabbers          Counter32,
    etherStatsCollisions       Counter32,
    etherStatsPkts64Octets     Counter32,
    etherStatsPkts65to127Octets Counter32,
    etherStatsPkts128to255Octets Counter32,
    etherStatsPkts256to511Octets Counter32,
    etherStatsPkts512to1023Octets Counter32,
    etherStatsPkts1024to1518Octets Counter32,
    etherStatsOwner            OwnerString,
    etherStatsStatus           EntryStatus
}
```

#### 통계 분석 알고리즘
```python
class EthernetStatistics:
    def __init__(self, interface_oid):
        self.interface = interface_oid
        self.counters = {
            'total_octets': 0,
            'total_packets': 0,
            'broadcast': 0,
            'multicast': 0,
            'errors': {
                'crc': 0,
                'alignment': 0,
                'undersize': 0,
                'oversize': 0,
                'fragments': 0,
                'jabbers': 0
            },
            'collisions': 0,
            'size_distribution': [0] * 7  # 7개 크기 범위
        }
    
    def process_packet(self, packet):
        """패킷 처리 및 통계 업데이트"""
        # 패킷 크기 분석
        packet_size = len(packet)
        self.update_size_distribution(packet_size)
        
        # 기본 통계
        self.counters['total_octets'] += packet_size
        self.counters['total_packets'] += 1
        
        # 브로드캐스트/멀티캐스트 확인
        dest_mac = packet[0:6]
        if dest_mac == b'\xff\xff\xff\xff\xff\xff':
            self.counters['broadcast'] += 1
        elif dest_mac[0] & 0x01:  # 멀티캐스트 비트 확인
            self.counters['multicast'] += 1
        
        # 오류 검사 (의사 코드)
        if self.check_crc_error(packet):
            self.counters['errors']['crc'] += 1
        if packet_size < 64:
            self.counters['errors']['undersize'] += 1
        elif packet_size > 1518:
            self.counters['errors']['oversize'] += 1
    
    def update_size_distribution(self, size):
        """패킷 크기 분포 업데이트"""
        if size == 64:
            self.counters['size_distribution'][0] += 1
        elif 65 <= size <= 127:
            self.counters['size_distribution'][1] += 1
        elif 128 <= size <= 255:
            self.counters['size_distribution'][2] += 1
        elif 256 <= size <= 511:
            self.counters['size_distribution'][3] += 1
        elif 512 <= size <= 1023:
            self.counters['size_distribution'][4] += 1
        elif 1024 <= size <= 1518:
            self.counters['size_distribution'][5] += 1
        else:  # 1518 초과
            self.counters['size_distribution'][6] += 1
```

### 3.2 이력 그룹 (history)

시간에 따른 성능 데이터를 수집하여 추세 분석을 가능하게 합니다.

#### 이력 수집 메커니즘
```
이력 컨트롤 테이블 구조:
historyControlTable → historyControlEntry → etherHistoryTable
    ↓                    ↓                      ↓
구성 설정           개별 이력 항목        실제 샘플 데이터

샘플링 프로세스:
1. 관리자가 샘플링 간격과 버킷 수 설정
2. RMON 프로브가 주기적으로 샘플 수집
3. 순환 버퍼에 저장 (오래된 데이터 덮어쓰기)
4. 관리자가 필요 시 데이터 조회
```

#### historyControlTable 정의
```asn.1
historyControlTable OBJECT-TYPE
    SYNTAX      SEQUENCE OF HistoryControlEntry
    MAX-ACCESS  not-accessible
    STATUS      current
    DESCRIPTION 
        "이력 데이터 수집을 제어하는 테이블"
    ::= { history 1 }

historyControlEntry ::= SEQUENCE {
    historyControlIndex           INTEGER,
    historyControlDataSource      OBJECT IDENTIFIER,
    historyControlBucketsRequested INTEGER,
    historyControlBucketsGranted  INTEGER,
    historyControlInterval        INTEGER,
    historyControlOwner           OwnerString,
    historyControlStatus          EntryStatus
}

etherHistoryTable OBJECT-TYPE
    SYNTAX      SEQUENCE OF EtherHistoryEntry
    MAX-ACCESS  not-accessible
    STATUS      current
    DESCRIPTION 
        "이더넷 세그먼트에 대한 역사적 통계 테이블"
    ::= { history 2 }

etherHistoryEntry ::= SEQUENCE {
    etherHistoryIndex           INTEGER,
    etherHistorySampleIndex     INTEGER,
    etherHistoryIntervalStart   Integer32,
    etherHistoryDropEvents      Gauge32,
    etherHistoryOctets          Gauge32,
    etherHistoryPkts            Gauge32,
    etherHistoryBroadcastPkts   Gauge32,
    etherHistoryMulticastPkts   Gauge32,
    etherHistoryCRCAlignErrors  Gauge32,
    etherHistoryUndersizePkts   Gauge32,
    etherHistoryOversizePkts    Gauge32,
    etherHistoryFragments       Gauge32,
    etherHistoryJabbers         Gauge32,
    etherHistoryCollisions      Gauge32,
    etherHistoryUtilization     Integer32
}
```

#### 이력 데이터 관리 알고리즘
```python
class HistoryCollector:
    def __init__(self, control_entry):
        self.control_index = control_entry['index']
        self.data_source = control_entry['data_source']
        self.interval = control_entry['interval']  # 초 단위
        self.buckets_granted = control_entry['buckets_granted']
        
        # 순환 버퍼 초기화
        self.buffer = [None] * self.buckets_granted
        self.current_index = 0
        self.start_time = time.time()
        self.sample_count = 0
    
    def collect_sample(self, current_stats):
        """샘플 수집 및 버퍼 관리"""
        if self.sample_count % self.interval == 0:
            # 새 샘플 생성
            sample = {
                'sample_index': self.sample_count // self.interval,
                'interval_start': int(time.time() - self.start_time),
                'drop_events': current_stats['drop_events'],
                'octets': current_stats['total_octets'],
                'packets': current_stats['total_packets'],
                'broadcast': current_stats['broadcast'],
                'multicast': current_stats['multicast'],
                'crc_errors': current_stats['errors']['crc'],
                'undersize': current_stats['errors']['undersize'],
                'oversize': current_stats['errors']['oversize'],
                'collisions': current_stats['collisions'],
                'utilization': self.calculate_utilization(current_stats)
            }
            
            # 순환 버퍼에 저장
            self.buffer[self.current_index] = sample
            self.current_index = (self.current_index + 1) % self.buckets_granted
        
        self.sample_count += 1
    
    def calculate_utilization(self, stats):
        """링크 활용률 계산"""
        # 이론적 최대 처리량 (10Mbps 이더넷 기준)
        max_throughput = 10000000  # bps
        
        # 샘플 간격 동안의 비트 수
        bits_in_interval = stats['total_octets'] * 8
        
        # 활용률 계산 (퍼센트)
        interval_seconds = self.interval
        actual_throughput = bits_in_interval / interval_seconds
        utilization = (actual_throughput / max_throughput) * 100
        
        return int(utilization * 100)  * 100분율로 반환
```

### 3.3 알람 그룹 (alarm)

임계값 기반 모니터링으로 사전 예방적 관리를 가능하게 합니다.

#### 알람 메커니즘
```python
class RMONAlarmManager:
    def __init__(self):
        self.alarm_table = {}
        self.event_table = {}
        self.alarm_history = []
    
    def configure_alarm(self, alarm_index, monitored_object, 
                       interval, sample_type, 
                       rising_threshold, falling_threshold):
        """알람 구성"""
        alarm_entry = {
            'index': alarm_index,
            'monitored_object': monitored_object,
            'interval': interval,  # 샘플링 간격 (초)
            'sample_type': sample_type,  # 'absolute' 또는 'delta'
            'rising_threshold': rising_threshold,
            'falling_threshold': falling_threshold,
            'rising_event_index': 0,
            'falling_event_index': 0,
            'rising_alarm_triggered': False,
            'falling_alarm_triggered': False,
            'last_value': 0,
            'startup_alarm': 'risingOrFalling',  # 또는 'rising', 'falling'
            'owner': 'admin',
            'status': 'active'
        }
        
        self.alarm_table[alarm_index] = alarm_entry
        return alarm_entry
    
    def check_alarms(self, current_values):
        """모든 알람 검사"""
        triggered_alarms = []
        
        for alarm_index, alarm_entry in self.alarm_table.items():
            if alarm_entry['status'] != 'active':
                continue
            
            monitored_oid = alarm_entry['monitored_object']
            current_value = current_values.get(monitored_oid, 0)
            
            # 샘플 값 계산
            if alarm_entry['sample_type'] == 'absolute':
                sampled_value = current_value
            else:  # delta
                sampled_value = current_value - alarm_entry['last_value']
                alarm_entry['last_value'] = current_value
            
            # 임계값 검사
            alarm_result = self.check_thresholds(
                sampled_value, 
                alarm_entry
            )
            
            if alarm_result:
                triggered_alarms.append({
                    'alarm_index': alarm_index,
                    'type': alarm_result,
                    'value': sampled_value,
                    'timestamp': time.time()
                })
                
                # 이벤트 생성
                self.generate_event(alarm_index, alarm_result, sampled_value)
        
        return triggered_alarms
    
    def check_thresholds(self, sampled_value, alarm_entry):
        """임계값 검사"""
        rising_threshold = alarm_entry['rising_threshold']
        falling_threshold = alarm_entry['falling_threshold']
        
        # 상승 임계값 검사
        if sampled_value >= rising_threshold:
            if not alarm_entry['rising_alarm_triggered']:
                alarm_entry['rising_alarm_triggered'] = True
                alarm_entry['falling_alarm_triggered'] = False
                return 'risingAlarm'
        
        # 하강 임계값 검사
        elif sampled_value <= falling_threshold:
            if not alarm_entry['falling_alarm_triggered']:
                alarm_entry['falling_alarm_triggered'] = True
                alarm_entry['rising_alarm_triggered'] = False
                return 'fallingAlarm'
        
        # 임계값 사이 영역
        elif (sampled_value < rising_threshold and 
              sampled_value > falling_threshold):
            # 알람 상태 초기화
            alarm_entry['rising_alarm_triggered'] = False
            alarm_entry['falling_alarm_triggered'] = False
        
        return None
    
    def generate_event(self, alarm_index, alarm_type, value):
        """이벤트 생성"""
        event_index = len(self.event_table) + 1
        
        event_entry = {
            'event_index': event_index,
            'event_description': f"{alarm_type} triggered for alarm {alarm_index}",
            'event_type': 'alarm',  # 또는 'log', 'trap', 'both'
            'community': 'public',
            'last_time_sent': time.time(),
            'owner': 'system',
            'status': 'active'
        }
        
        self.event_table[event_index] = event_entry
        
        # 로그 기록
        self.log_event(event_entry)
        
        # SNMP 트랩 발송 (설정된 경우)
        if event_entry['event_type'] in ['trap', 'both']:
            self.send_snmp_trap(event_entry)
        
        return event_index
```

### 3.4 호스트 그룹 (hosts)

네트워크 내 개별 호스트를 발견하고 모니터링합니다.

#### 호스트 발견 알고리즘
```python
class HostDiscovery:
    def __init__(self):
        self.host_table = {}  # MAC 주소 → 호스트 정보
        self.host_control_table = {}
        self.last_cleanup = time.time()
    
    def process_packet(self, packet, timestamp):
        """패킷 처리 및 호스트 정보 업데이트"""
        # MAC 주소 추출
        src_mac = bytes_to_mac(packet[0:6])
        dst_mac = bytes_to_mac(packet[6:12])
        
        # 호스트 테이블 업데이트
        self.update_host(src_mac, timestamp, 'source', len(packet))
        self.update_host(dst_mac, timestamp, 'destination', len(packet))
        
        # 정기적 클리닝 (비활성 호스트 제거)
        if time.time() - self.last_cleanup > 3600:  # 1시간마다
            self.cleanup_inactive_hosts(1800)  # 30분 이상 비활성 호스트
            self.last_cleanup = time.time()
    
    def update_host(self, mac_address, timestamp, direction, packet_size):
        """호스트 정보 업데이트"""
        if mac_address not in self.host_table:
            # 새 호스트 발견
            host_entry = self.create_host_entry(mac_address, timestamp)
            self.host_table[mac_address] = host_entry
        
        host_entry = self.host_table[mac_address]
        
        # 통계 업데이트
        if direction == 'source':
            host_entry['out_packets'] += 1
            host_entry['out_octets'] += packet_size
        else:  # destination
            host_entry['in_packets'] += 1
            host_entry['in_octets'] += packet_size
        
        # 마지막 활동 시간 업데이트
        host_entry['last_seen'] = timestamp
        
        # 호스트 상태 확인
        self.update_host_status(host_entry)
    
    def create_host_entry(self, mac_address, discovery_time):
        """새 호스트 엔트리 생성"""
        return {
            'mac_address': mac_address,
            'host_index': len(self.host_table) + 1,
            'in_packets': 0,
            'out_packets': 0,
            'in_octets': 0,
            'out_octets': 0,
            'discovery_time': discovery_time,
            'last_seen': discovery_time,
            'status': 'active',
            'owner': 'system',
            # 추가 정보 (나중에 채워짐)
            'ip_address': None,
            'hostname': None,
            'vendor': self.get_vendor_from_mac(mac_address)
        }
    
    def get_vendor_from_mac(self, mac_address):
        """MAC 주소로 벤더 정보 추출 (OUI 기반)"""
        # OUI (Organizationally Unique Identifier) 조회
        oui = mac_address[:8].replace(':', '')[:6].upper()
        
        # 알려진 OUI 매핑 (예시)
        oui_database = {
            '00000C': 'Cisco',
            '001122': 'Dell',
            'AABBCC': 'VMware',
            '080020': 'Sun',
            '0050BA': 'IBM'
        }
        
        return oui_database.get(oui, 'Unknown')
    
    def cleanup_inactive_hosts(self, inactivity_threshold):
        """비활성 호스트 정리"""
        current_time = time.time()
        inactive_hosts = []
        
        for mac_address, host_entry in self.host_table.items():
            inactivity_time = current_time - host_entry['last_seen']
            
            if inactivity_time > inactivity_threshold:
                inactive_hosts.append(mac_address)
        
        # 비활성 호스트 제거
        for mac_address in inactive_hosts:
            del self.host_table[mac_address]
        
        return len(inactive_hosts)
```

### 3.5 호스트 TopN 분석

주기적으로 호스트 테이블을 분석하여 상위 N개 호스트를 식별합니다.

#### TopN 분석 알고리즘
```python
class TopNAnalyzer:
    def __init__(self, host_table):
        self.host_table = host_table
        self.topn_tables = {}  # 기준별 TopN 결과
    
    def analyze_topn(self, rate_base, time_interval, number_of_hosts, direction):
        """
        상위 N개 호스트 분석
        rate_base: 'packets' 또는 'octets'
        time_interval: 분석 간격 (초)
        number_of_hosts: 상위 N개
        direction: 'in', 'out', 또는 'both'
        """
        host_metrics = []
        
        for mac_address, host_entry in self.host_table.items():
            # 호스트 메트릭 계산
            metric = self.calculate_host_metric(
                host_entry, rate_base, time_interval, direction
            )
            
            if metric > 0:
                host_metrics.append({
                    'mac_address': mac_address,
                    'metric': metric,
                    'host_entry': host_entry
                })
        
        # 메트릭 기준 정렬 (내림차순)
        host_metrics.sort(key=lambda x: x['metric'], reverse=True)
        
        # 상위 N개 선택
        topn_results = host_metrics[:number_of_hosts]
        
        # 결과 테이블 생성
        result_table = []
        for rank, result in enumerate(topn_results, 1):
            result_table.append({
                'rank': rank,
                'mac_address': result['mac_address'],
                'metric_value': result['metric'],
                'start_time': time.time() - time_interval,
                'end_time': time.time(),
                'rate_base': rate_base,
                'direction': direction
            })
        
        # 결과 저장
        table_key = f"{rate_base}_{direction}_{time_interval}"
        self.topn_tables[table_key] = {
            'generated_at': time.time(),
            'results': result_table
        }
        
        return result_table
    
    def calculate_host_metric(self, host_entry, rate_base, interval, direction):
        """호스트 메트릭 계산"""
        if direction == 'in':
            if rate_base == 'packets':
                return host_entry['in_packets'] / interval
            else:  # octets
                return host_entry['in_octets'] / interval
        
        elif direction == 'out':
            if rate_base == 'packets':
                return host_entry['out_packets'] / interval
            else:  # octets
                return host_entry['out_octets'] / interval
        
        else:  # both
            if rate_base == 'packets':
                total_packets = host_entry['in_packets'] + host_entry['out_packets']
                return total_packets / interval
            else:  # octets
                total_octets = host_entry['in_octets'] + host_entry['out_octets']
                return total_octets / interval
```

### 3.6 매트릭스 그룹 (matrix)

호스트 간 통신 패턴을 분석하여 대화 매트릭스를 생성합니다.

#### 매트릭스 분석
```python
class MatrixAnalyzer:
    def __init__(self):
        self.conversation_table = {}  # (src_mac, dst_mac) → 통계
        self.matrix_control_table = {}
    
    def process_conversation(self, src_mac, dst_mac, packet_size):
        """호스트 간 대화 분석"""
        # 브로드캐스트/멀티캐스트 제외
        if dst_mac == 'ff:ff:ff:ff:ff:ff' or dst_mac.startswith('01:'):
            return
        
        conversation_key = (src_mac, dst_mac)
        
        if conversation_key not in self.conversation_table:
            # 새 대화 생성
            self.conversation_table[conversation_key] = {
                'source_address': src_mac,
                'destination_address': dst_mac,
                'packets': 0,
                'octets': 0,
                'creation_time': time.time(),
                'last_updated': time.time()
            }
        
        # 통계 업데이트
        conversation = self.conversation_table[conversation_key]
        conversation['packets'] += 1
        conversation['octets'] += packet_size
        conversation['last_updated'] = time.time()
    
    def generate_matrix_report(self, sort_by='octets', limit=50):
        """매트릭스 보고서 생성"""
        conversations = list(self.conversation_table.values())
        
        # 정렬 기준 선택
        if sort_by == 'octets':
            conversations.sort(key=lambda x: x['octets'], reverse=True)
        elif sort_by == 'packets':
            conversations.sort(key=lambda x: x['packets'], reverse=True)
        elif sort_by == 'recent':
            conversations.sort(key=lambda x: x['last_updated'], reverse=True)
        
        # 제한 적용
        if limit > 0:
            conversations = conversations[:limit]
        
        # 보고서 생성
        report = {
            'generated_at': time.time(),
            'total_conversations': len(self.conversation_table),
            'sorted_by': sort_by,
            'conversations': []
        }
        
        for conv in conversations:
            report['conversations'].append({
                'source': conv['source_address'],
                'destination': conv['destination_address'],
                'packets': conv['packets'],
                'octets': conv['octets'],
                'average_packet_size': conv['octets'] / conv['packets'] if conv['packets'] > 0 else 0,
                'last_active': conv['last_updated'],
                'duration': time.time() - conv['creation_time']
            })
        
        return report
    
    def cleanup_old_conversations(self, age_threshold=3600):
        """오래된 대화 정리"""
        current_time = time.time()
        removed_count = 0
        
        keys_to_remove = []
        for key, conversation in self.conversation_table.items():
            inactive_time = current_time - conversation['last_updated']
            if inactive_time > age_threshold:
                keys_to_remove.append(key)
        
        for key in keys_to_remove:
            del self.conversation_table[key]
            removed_count += 1
        
        return removed_count
```

## 4. RMON2: 프로토콜 계층 확장

### 4.1 RMON2의 혁신

RMON2는 RMON1을 확장하여 OSI 모델의 네트워크 계층 이상을 모니터링합니다.

```
RMON2의 계층적 모니터링
+---------------------+-----------------------------------------------+-------------------+
| OSI 계층            | 모니터링 정보                                 | RMON2 그룹        |
+---------------------+-----------------------------------------------+-------------------+
| 응용 계층 (7)       | HTTP, FTP, SMTP, DNS 트래픽                  | protocolDist      |
|                     |                                               | appLayerHost      |
|                     |                                               | appLayerMatrix    |
+---------------------+-----------------------------------------------+-------------------+
| 표현 계층 (6)       | 데이터 형식, 암호화                          | (간접적 모니터링) |
+---------------------+-----------------------------------------------+-------------------+
| 세션 계층 (5)       | 세션 설정, 관리, 종료                        | (간접적 모니터링) |
+---------------------+-----------------------------------------------+-------------------+
| 전송 계층 (4)       | TCP/UDP 포트, 연결 상태                      | nlHost            |
|                     |                                               | nlMatrix          |
+---------------------+-----------------------------------------------+-------------------+
| 네트워크 계층 (3)   | IP 주소, 라우팅, 서브넷                      | addressMap        |
|                     |                                               | nlHost            |
+---------------------+-----------------------------------------------+-------------------+
| 데이터 링크 계층 (2)| MAC 주소, VLAN                               | RMON1 groups      |
+---------------------+-----------------------------------------------+-------------------+
| 물리 계층 (1)       | 인터페이스 통계                              | RMON1 groups      |
+---------------------+-----------------------------------------------+-------------------+
```

### 4.2 프로토콜 디렉토리 (protocolDir)

RMON2의 핵심 구성 요소로, 모니터링 가능한 프로토콜들을 계층적으로 정의합니다.

#### 프로토콜 인코딩 체계
```
프로토콜 식별자 형식:
계층적 점 표기법 사용
예시: Ethernet.IP.TCP.HTTP → 0.0.1.0.0.8.0.0.6.0.80

공통 프로토콜 식별자:
• Ethernet: 0.0.1
• LLC: 0.0.2
• SNAP: 0.0.3
• IP: 0.0.1.0.0.8 (Ethernet.IP)
• IPv6: 0.0.1.0.0.8.0.0.41 (Ethernet.IPv6)
• TCP: 0.0.1.0.0.8.0.0.6 (Ethernet.IP.TCP)
• UDP: 0.0.1.0.0.8.0.0.17 (Ethernet.IP.UDP)
• HTTP: 0.0.1.0.0.8.0.0.6.0.80 (Ethernet.IP.TCP.HTTP)
• DNS: 0.0.1.0.0.8.0.0.17.0.53 (Ethernet.IP.UDP.DNS)
```

#### protocolDirTable 구조
```python
class ProtocolDirectory:
    def __init__(self):
        self.protocol_table = {}
        self.next_index = 1
        
        # 기본 프로토콜 등록
        self.register_standard_protocols()
    
    def register_standard_protocols(self):
        """표준 프로토콜 등록"""
        standard_protocols = [
            {
                'id': '0.0.1',
                'description': 'Ethernet',
                'address_map_config': True,
                'host_config': True,
                'matrix_config': True,
                'parameters': []
            },
            {
                'id': '0.0.1.0.0.8',
                'description': 'IP over Ethernet',
                'address_map_config': True,
                'host_config': True,
                'matrix_config': True,
                'parameters': [
                    {'type': 'ipv4_address', 'length': 4}
                ]
            },
            {
                'id': '0.0.1.0.0.8.0.0.6',
                'description': 'TCP over IP over Ethernet',
                'address_map_config': False,
                'host_config': True,
                'matrix_config': True,
                'parameters': [
                    {'type': 'ipv4_address', 'length': 4},
                    {'type': 'tcp_port', 'length': 2}
                ]
            },
            {
                'id': '0.0.1.0.0.8.0.0.17',
                'description': 'UDP over IP over Ethernet',
                'address_map_config': False,
                'host_config': True,
                'matrix_config': True,
                'parameters': [
                    {'type': 'ipv4_address', 'length': 4},
                    {'type': 'udp_port', 'length': 2}
                ]
            }
        ]
        
        for proto in standard_protocols:
            self.add_protocol(proto)
    
    def add_protocol(self, protocol_info):
        """새 프로토콜 등록"""
        protocol_id = protocol_info['id']
        
        protocol_entry = {
            'protocol_dir_id': protocol_id,
            'protocol_dir_parameters': protocol_info.get('parameters', []),
            'protocol_dir_local_index': self.next_index,
            'protocol_dir_descr': protocol_info['description'],
            'protocol_dir_type': 'protocolOnly',  # 또는 'addressOnly', 'protocolAndAddress'
            'protocol_dir_address_map_config': protocol_info.get('address_map_config', False),
            'protocol_dir_host_config': protocol_info.get('host_config', False),
            'protocol_dir_matrix_config': protocol_info.get('matrix_config', False),
            'protocol_dir_owner': 'system',
            'protocol_dir_status': 'active'
        }
        
        self.protocol_table[self.next_index] = protocol_entry
        self.next_index += 1
        
        return protocol_entry
    
    def find_protocol_by_id(self, protocol_id):
        """프로토콜 ID로 검색"""
        for entry in self.protocol_table.values():
            if entry['protocol_dir_id'] == protocol_id:
                return entry
        return None
    
    def get_protocol_stack(self, packet):
        """패킷에서 프로토콜 스택 추출"""
        protocols = []
        offset = 0
        
        # 이더넷 헤더
        ethertype = packet[12:14]
        if ethertype == b'\x08\x00':  # IPv4
            protocols.append('0.0.1.0.0.8')
            offset = 14
            
            # IP 헤더 분석
            protocol_num = packet[offset + 9]
            if protocol_num == 6:  # TCP
                protocols.append('0.0.1.0.0.8.0.0.6')
                offset += 20  # IP 헤더 길이
                
                # TCP 포트 분석
                src_port = int.from_bytes(packet[offset:offset+2], 'big')
                dst_port = int.from_bytes(packet[offset+2:offset+4], 'big')
                
                # 응용 프로토콜 식별
                app_protocol = self.identify_application(src_port, dst_port)
                if app_protocol:
                    protocols.append(app_protocol)
            
            elif protocol_num == 17:  # UDP
                protocols.append('0.0.1.0.0.8.0.0.17')
                offset += 20
                
                # UDP 포트 분석
                src_port = int.from_bytes(packet[offset:offset+2], 'big')
                dst_port = int.from_bytes(packet[offset+2:offset+4], 'big')
                
                # 응용 프로토콜 식별
                app_protocol = self.identify_application(src_port, dst_port)
                if app_protocol:
                    protocols.append(app_protocol)
        
        return protocols
    
    def identify_application(self, src_port, dst_port):
        """포트 번호로 응용 프로토콜 식별"""
        port_protocol_map = {
            80: '0.0.1.0.0.8.0.0.6.0.80',    # HTTP
            443: '0.0.1.0.0.8.0.0.6.0.443',   # HTTPS
            25: '0.0.1.0.0.8.0.0.6.0.25',     # SMTP
            110: '0.0.1.0.0.8.0.0.6.0.110',   # POP3
            53: '0.0.1.0.0.8.0.0.17.0.53',    # DNS
            161: '0.0.1.0.0.8.0.0.17.0.161',  # SNMP
            162: '0.0.1.0.0.8.0.0.17.0.162',  # SNMP Trap
        }
        
        # 일반적으로 목적지 포트로 프로토콜 식별
        return port_protocol_map.get(dst_port)
```

### 4.3 프로토콜 분포 분석 (protocolDist)

네트워크 트래픽의 프로토콜별 분포를 분석합니다.

```python
class ProtocolDistribution:
    def __init__(self, protocol_dir):
        self.protocol_dir = protocol_dir
        self.distribution_table = {}  # 프로토콜 ID → 통계
        self.iface_distributions = {}  # 인터페이스별 분포
    
    def process_packet(self, interface_id, packet):
        """패킷 처리 및 프로토콜 분포 업데이트"""
        # 프로토콜 스택 추출
        protocol_stack = self.protocol_dir.get_protocol_stack(packet)
        
        if not protocol_stack:
            return
        
        # 각 프로토콜 레벨별로 분포 업데이트
        for i, protocol_id in enumerate(protocol_stack):
            # 프로토콜 경로 생성 (계층적 표현)
            protocol_path = '.'.join(protocol_stack[:i+1])
            
            # 전역 분포 업데이트
            self.update_distribution(protocol_path, len(packet))
            
            # 인터페이스별 분포 업데이트
            if interface_id not in self.iface_distributions:
                self.iface_distributions[interface_id] = {}
            
            self.update_interface_distribution(
                interface_id, protocol_path, len(packet)
            )
    
    def update_distribution(self, protocol_path, packet_size):
        """프로토콜 분포 업데이트"""
        if protocol_path not in self.distribution_table:
            self.distribution_table[protocol_path] = {
                'packets': 0,
                'octets': 0,
                'last_updated': time.time()
            }
        
        stats = self.distribution_table[protocol_path]
        stats['packets'] += 1
        stats['octets'] += packet_size
        stats['last_updated'] = time.time()
    
    def get_distribution_report(self, sort_by='octets', limit=20):
        """프로토콜 분포 보고서 생성"""
        # 데이터 준비
        report_data = []
        
        for protocol_path, stats in self.distribution_table.items():
            report_data.append({
                'protocol': protocol_path,
                'packets': stats['packets'],
                'octets': stats['octets'],
                'avg_packet_size': stats['octets'] / stats['packets'] if stats['packets'] > 0 else 0,
                'last_active': stats['last_updated']
            })
        
        # 정렬
        if sort_by == 'octets':
            report_data.sort(key=lambda x: x['octets'], reverse=True)
        elif sort_by == 'packets':
            report_data.sort(key=lambda x: x['packets'], reverse=True)
        elif sort_by == 'recent':
            report_data.sort(key=lambda x: x['last_active'], reverse=True)
        
        # 제한 적용
        if limit > 0:
            report_data = report_data[:limit]
        
        # 총계 계산
        total_packets = sum(item['packets'] for item in self.distribution_table.values())
        total_octets = sum(item['octets'] for item in self.distribution_table.values())
        
        # 비율 계산
        for item in report_data:
            if total_packets > 0:
                item['packet_percentage'] = (item['packets'] / total_packets) * 100
            if total_octets > 0:
                item['octet_percentage'] = (item['octets'] / total_octets) * 100
        
        return {
            'generated_at': time.time(),
            'total_protocols': len(self.distribution_table),
            'total_packets': total_packets,
            'total_octets': total_octets,
            'sorted_by': sort_by,
            'distribution': report_data
        }
```

### 4.4 네트워크 계층 호스트 모니터링 (nlHost)

네트워크 계층(IP)에서의 호스트 모니터링을 제공합니다.

```python
class NetworkLayerHostMonitor:
    def __init__(self):
        self.nl_host_table = {}  # (protocol_id, address) → 통계
        self.address_cache = {}  # MAC → IP 매핑
        self.host_control_table = {}
    
    def process_packet(self, protocol_id, network_address, packet_size, direction):
        """네트워크 계층 호스트 통계 업데이트"""
        host_key = (protocol_id, network_address)
        
        if host_key not in self.nl_host_table:
            # 새 호스트 엔트리 생성
            self.nl_host_table[host_key] = {
                'protocol_id': protocol_id,
                'address': network_address,
                'in_packets': 0,
                'out_packets': 0,
                'in_octets': 0,
                'out_octets': 0,
                'first_seen': time.time(),
                'last_seen': time.time(),
                'status': 'active'
            }
        
        host_entry = self.nl_host_table[host_key]
        
        # 통계 업데이트
        if direction == 'in':
            host_entry['in_packets'] += 1
            host_entry['in_octets'] += packet_size
        else:  # out
            host_entry['out_packets'] += 1
            host_entry['out_octets'] += packet_size
        
        host_entry['last_seen'] = time.time()
        
        # 추가 정보 수집
        self.collect_host_info(protocol_id, network_address)
    
    def collect_host_info(self, protocol_id, address):
        """호스트 추가 정보 수집"""
        if protocol_id == '0.0.1.0.0.8':  # IPv4
            # DNS 역방향 조회 시도
            if address not in self.address_cache:
                try:
                    hostname = socket.gethostbyaddr(address)[0]
                    self.address_cache[address] = {
                        'hostname': hostname,
                        'resolved_at': time.time()
                    }
                except (socket.herror, socket.gaierror):
                    self.address_cache[address] = {
                        'hostname': None,
                        'resolved_at': time.time()
                    }
    
    def get_host_report(self, protocol_filter=None, sort_by='octets'):
        """호스트 보고서 생성"""
        filtered_hosts = []
        
        for host_key, host_entry in self.nl_host_table.items():
            protocol_id, address = host_key
            
            # 프로토콜 필터 적용
            if protocol_filter and protocol_id != protocol_filter:
                continue
            
            # 총 통계 계산
            total_packets = host_entry['in_packets'] + host_entry['out_packets']
            total_octets = host_entry['in_octets'] + host_entry['out_octets']
            
            # 활동 기간
            uptime = time.time() - host_entry['first_seen']
            
            # 호스트 정보 조회
            host_info = self.address_cache.get(address, {})
            
            filtered_hosts.append({
                'protocol': protocol_id,
                'address': address,
                'hostname': host_info.get('hostname'),
                'in_packets': host_entry['in_packets'],
                'out_packets': host_entry['out_packets'],
                'in_octets': host_entry['in_octets'],
                'out_octets': host_entry['out_octets'],
                'total_packets': total_packets,
                'total_octets': total_octets,
                'first_seen': host_entry['first_seen'],
                'last_seen': host_entry['last_seen'],
                'uptime': uptime,
                'avg_traffic_rate': total_octets / uptime if uptime > 0 else 0
            })
        
        # 정렬
        if sort_by == 'octets':
            filtered_hosts.sort(key=lambda x: x['total_octets'], reverse=True)
        elif sort_by == 'packets':
            filtered_hosts.sort(key=lambda x: x['total_packets'], reverse=True)
        elif sort_by == 'recent':
            filtered_hosts.sort(key=lambda x: x['last_seen'], reverse=True)
        elif sort_by == 'rate':
            filtered_hosts.sort(key=lambda x: x['avg_traffic_rate'], reverse=True)
        
        return filtered_hosts
```

### 4.5 주소 매핑 (addressMap)

네트워크 계층 주소(IP)와 데이터 링크 계층 주소(MAC) 간의 매핑을 관리합니다.

```python
class AddressMapping:
    def __init__(self):
        self.address_map_table = {}  # (network_address, mac_address) → 매핑 정보
        self.arp_cache = {}  # ARP 캐시
        self.discovery_methods = ['arp', 'traffic_analysis', 'manual']
    
    def discover_mapping(self, packet, interface_id):
        """패킷으로부터 주소 매핑 발견"""
        # 이더넷 프레임 분석
        src_mac = bytes_to_mac(packet[0:6])
        dst_mac = bytes_to_mac(packet[6:12])
        ethertype = packet[12:14]
        
        # IP 패킷인 경우
        if ethertype == b'\x08\x00':  # IPv4
            # IP 헤더 시작 오프셋
            ip_header_start = 14
            
            # 출발지 IP 주소 추출
            src_ip = self.extract_ip_address(packet, ip_header_start + 12)
            
            # 매핑 발견
            self.add_mapping(src_ip, src_mac, 'traffic_analysis', interface_id)
            
            # ARP 패킷인 경우 추가 처리
            if packet[14:16] == b'\x08\x06':  # ARP
                self.process_arp_packet(packet, interface_id)
        
        # IPv6 처리
        elif ethertype == b'\x86\xdd':  # IPv6
            # IPv6 주소 추출 (의사 코드)
            pass
    
    def process_arp_packet(self, arp_packet, interface_id):
        """ARP 패킷 처리"""
        # ARP 패킷 구조 분석
        # offset 14: hardware type (2)
        # offset 16: protocol type (2)
        # offset 18: hardware size (1)
        # offset 19: protocol size (1)
        # offset 20: opcode (2)
        # offset 22: sender MAC (6)
        # offset 28: sender IP (4)
        # offset 32: target MAC (6)
        # offset 38: target IP (4)
        
        opcode = arp_packet[20:22]
        sender_mac = bytes_to_mac(arp_packet[22:28])
        sender_ip = self.bytes_to_ip(arp_packet[28:32])
        target_mac = bytes_to_mac(arp_packet[32:38])
        target_ip = self.bytes_to_ip(arp_packet[38:42])
        
        if opcode == b'\x00\x01':  # ARP Request
            self.add_mapping(sender_ip, sender_mac, 'arp', interface_id)
        
        elif opcode == b'\x00\x02':  # ARP Reply
            self.add_mapping(sender_ip, sender_mac, 'arp', interface_id)
            self.add_mapping(target_ip, target_mac, 'arp', interface_id)
    
    def add_mapping(self, ip_address, mac_address, discovery_method, interface_id):
        """주소 매핑 추가"""
        mapping_key = (ip_address, mac_address)
        
        if mapping_key not in self.address_map_table:
            # 새 매핑 생성
            self.address_map_table[mapping_key] = {
                'network_address': ip_address,
                'mac_address': mac_address,
                'interface_id': interface_id,
                'discovery_method': discovery_method,
                'first_discovered': time.time(),
                'last_verified': time.time(),
                'verification_count': 1,
                'status': 'active'
            }
        else:
            # 기존 매핑 업데이트
            mapping = self.address_map_table[mapping_key]
            mapping['last_verified'] = time.time()
            mapping['verification_count'] += 1
        
        # ARP 캐시 업데이트
        self.arp_cache[ip_address] = {
            'mac': mac_address,
            'updated': time.time(),
            'interface': interface_id
        }
    
    def get_mapping_report(self, ip_address=None, mac_address=None, 
                          interface_id=None, method=None):
        """매핑 보고서 생성"""
        filtered_mappings = []
        
        for mapping_key, mapping in self.address_map_table.items():
            ip_addr, mac_addr = mapping_key
            
            # 필터 적용
            if ip_address and ip_addr != ip_address:
                continue
            if mac_address and mac_addr != mac_address:
                continue
            if interface_id and mapping['interface_id'] != interface_id:
                continue
            if method and mapping['discovery_method'] != method:
                continue
            
            # 최근 확인 시간 계산
            last_seen_ago = time.time() - mapping['last_verified']
            
            filtered_mappings.append({
                'ip_address': ip_addr,
                'mac_address': mac_addr,
                'interface': mapping['interface_id'],
                'discovery_method': mapping['discovery_method'],
                'first_discovered': mapping['first_discovered'],
                'last_verified': mapping['last_verified'],
                'last_seen_ago': last_seen_ago,
                'verification_count': mapping['verification_count'],
                'status': mapping['status']
            })
        
        # 정렬 (기본: 마지막 확인 시간 역순)
        filtered_mappings.sort(key=lambda x: x['last_verified'], reverse=True)
        
        return {
            'total_mappings': len(self.address_map_table),
            'filtered_count': len(filtered_mappings),
            'mappings': filtered_mappings
        }
    
    def cleanup_stale_mappings(self, age_threshold=7200):
        """오래된 매핑 정리 (2시간 기본)"""
        current_time = time.time()
        stale_count = 0
        
        keys_to_remove = []
        for mapping_key, mapping in self.address_map_table.items():
            inactive_time = current_time - mapping['last_verified']
            
            if inactive_time > age_threshold:
                keys_to_remove.append(mapping_key)
        
        for key in keys_to_remove:
            ip_addr, mac_addr = key
            del self.address_map_table[key]
            
            # ARP 캐시에서도 제거
            if ip_addr in self.arp_cache:
                del self.arp_cache[ip_addr]
            
            stale_count += 1
        
        return stale_count
```

## 5. 현대적 RMON 구현 및 통합

### 5.1 sFlow와의 통합

```python
class RMONsFlowIntegration:
    def __init__(self, rmon_system):
        self.rmon = rmon_system
        self.sflow_agents = {}
        self.integration_mode = 'hybrid'  # 'rmon_only', 'sflow_only', 'hybrid'
    
    def integrate_sflow_sample(self, sflow_sample):
        """sFlow 샘플을 RMON 데이터로 통합"""
        if self.integration_mode in ['sflow_only', 'hybrid']:
            # sFlow 샘플 처리
            self.process_sflow_counters(sflow_sample.get('counters', []))
            self.process_sflow_flows(sflow_sample.get('flows', []))
        
        if self.integration_mode in ['rmon_only', 'hybrid']:
            # RMON 데이터와 상호보완
            self.correlate_data(sflow_sample)
    
    def process_sflow_counters(self, counters):
        """sFlow 카운터 처리"""
        for counter in counters:
            if counter['type'] == 'interface':
                # 인터페이스 통계 업데이트
                self.update_interface_stats(
                    counter['interface_index'],
                    counter['in_octets'],
                    counter['out_octets'],
                    counter['in_packets'],
                    counter['out_packets']
                )
    
    def process_sflow_flows(self, flows):
        """sFlow 흐름 샘플 처리"""
        for flow in flows:
            # 호스트 통계 업데이트
            self.update_host_stats(
                src_ip=flow.get('src_ip'),
                dst_ip=flow.get('dst_ip'),
                src_port=flow.get('src_port'),
                dst_port=flow.get('dst_port'),
                protocol=flow.get('protocol'),
                octets=flow.get('octets'),
                packets=flow.get('packets')
            )
            
            # 프로토콜 분포 업데이트
            self.update_protocol_distribution(
                protocol=flow.get('protocol'),
                octets=flow.get('octets'),
                packets=flow.get('packets')
            )
    
    def correlate_data(self, sflow_sample):
        """RMON과 sFlow 데이터 상관 분석"""
        # 시간 동기화
        sample_time = sflow_sample.get('timestamp')
        
        # 데이터 일관성 검사
        self.validate_data_consistency(sflow_sample)
        
        # 보완적 데이터 통합
        self.integrate_complementary_data(sflow_sample)
    
    def get_integrated_report(self):
        """통합 보고서 생성"""
        report = {
            'timestamp': time.time(),
            'integration_mode': self.integration_mode,
            'rmon_stats': self.rmon.get_summary_statistics(),
            'sflow_stats': self.get_sflow_statistics(),
            'correlated_insights': self.generate_correlated_insights()
        }
        
        return report
    
    def generate_correlated_insights(self):
        """상관된 통찰 생성"""
        insights = []
        
        # 예시: 대역폭 사용 패턴
        bandwidth_pattern = self.analyze_bandwidth_patterns()
        if bandwidth_pattern:
            insights.append({
                'type': 'bandwidth_pattern',
                'description': bandwidth_pattern['description'],
                'recommendation': bandwidth_pattern.get('recommendation')
            })
        
        # 예시: 이상 탐지
        anomalies = self.detect_anomalies()
        insights.extend(anomalies)
        
        return insights
```

### 5.2 NetFlow와의 비교

```
RMON, sFlow, NetFlow 비교
+---------------------+---------------------+---------------------+-------------------+
| 특성                | RMON                | sFlow               | NetFlow           |
+---------------------+---------------------+---------------------+-------------------+
| 데이터 수집 방식    | 스니핑/카운터      | 샘플링 기반         | 흐름 기반         |
| 프로토콜            | SNMP 확장          | sFlow 프로토콜      | NetFlow 프로토콜  |
| 실시간성            | 높음                | 매우 높음           | 높음              |
| 확장성              | 중간                | 매우 높음           | 높음              |
| 장비 지원           | 제한적              | 광범위              | 매우 광범위       |
| 분석 깊이           | 매우 깊음           | 중간                | 기본              |
| 대역폭 사용         | 중간-높음          | 매우 낮음           | 낮음              |
| CPU 사용            | 높음                | 낮음                | 중간              |
| 저장소 요구         | 높음                | 낮음                | 중간              |
| 주요 용도           | 상세 진단          | 실시간 모니터링    | 회계/보안         |
| 계층 지원           | L2-L7 (RMON2)      | L2-L4               | L3-L4             |
+---------------------+---------------------+---------------------+-------------------+

통합 전략:
• RMON: 상세 진단, 문제 해결, 역사적 분석
• sFlow: 실시간 트래픽 분석, 대규모 네트워크 모니터링
• NetFlow: 트래픽 회계, 보안 분석, QoS 모니터링
• SNMP: 기본 상태 모니터링, 구성 관리
```

### 5.3 클라우드 환경에서의 RMON

```python
class CloudRMONAdapter:
    def __init__(self, cloud_platform):
        self.platform = cloud_platform  # 'aws', 'azure', 'gcp', 'vmware'
        self.virtual_probes = {}
        self.metrics_collectors = {}
        
        # 클라우드별 초기화
        self.initialize_for_platform()
    
    def initialize_for_platform(self):
        """클라우드 플랫폼별 초기화"""
        if self.platform == 'aws':
            self.initialize_aws_monitoring()
        elif self.platform == 'azure':
            self.initialize_azure_monitoring()
        elif self.platform == 'gcp':
            self.initialize_gcp_monitoring()
        elif self.platform == 'vmware':
            self.initialize_vmware_monitoring()
    
    def initialize_aws_monitoring(self):
        """AWS 모니터링 초기화"""
        # CloudWatch 메트릭 수집기
        self.metrics_collectors['cloudwatch'] = CloudWatchCollector()
        
        # VPC Flow Logs 수집기
        self.metrics_collectors['flow_logs'] = VPCFlowLogsCollector()
        
        # 가상 프로브 배포 (Lambda 함수)
        self.deploy_virtual_probes_aws()
    
    def deploy_virtual_probes_aws(self):
        """AWS 가상 프로브 배포"""
        # 각 VPC/서브넷에 가상 프로브 배포
        vpcs = self.discover_vpcs()
        
        for vpc in vpcs:
            probe_id = f"rmon-probe-{vpc['id']}"
            
            self.virtual_probes[probe_id] = {
                'type': 'lambda',
                'vpc_id': vpc['id'],
                'subnets': vpc['subnets'],
                'status': 'active',
                'deployed_at': time.time(),
                'metrics_collected': []
            }
            
            # Lambda 함수 배포 (의사 코드)
            # self.deploy_lambda_function(probe_id, vpc)
    
    def collect_cloud_metrics(self):
        """클라우드 메트릭 수집"""
        all_metrics = {}
        
        for collector_name, collector in self.metrics_collectors.items():
            try:
                metrics = collector.collect()
                all_metrics[collector_name] = metrics
                
                # RMON 형식으로 변환
                rmon_metrics = self.convert_to_rmon_format(metrics, collector_name)
                self.update_rmon_tables(rmon_metrics)
                
            except Exception as e:
                self.log_error(f"Collector {collector_name} failed: {e}")
        
        return all_metrics
    
    def convert_to_rmon_format(self, cloud_metrics, source_type):
        """클라우드 메트릭을 RMON 형식으로 변환"""
        rmon_data = {
            'statistics': {},
            'hosts': [],
            'matrix': [],
            'protocol_dist': {}
        }
        
        if source_type == 'cloudwatch':
            # CloudWatch 메트릭 변환
            for metric in cloud_metrics.get('NetworkMetrics', []):
                interface_id = metric.get('InterfaceId')
                
                rmon_data['statistics'][interface_id] = {
                    'octets': metric.get('Bytes', 0),
                    'packets': metric.get('Packets', 0),
                    'errors': metric.get('Errors', 0),
                    'drops': metric.get('PacketDropCount', 0)
                }
        
        elif source_type == 'flow_logs':
            # VPC Flow Logs 변환
            for flow in cloud_metrics.get('Flows', []):
                # 호스트 정보
                rmon_data['hosts'].append({
                    'src_addr': flow.get('srcaddr'),
                    'dst_addr': flow.get('dstaddr'),
                    'packets': flow.get('packets', 0),
                    'bytes': flow.get('bytes', 0),
                    'protocol': flow.get('protocol')
                })
                
                # 매트릭스 정보
                rmon_data['matrix'].append({
                    'src': flow.get('srcaddr'),
                    'dst': flow.get('dstaddr'),
                    'packets': flow.get('packets', 0),
                    'octets': flow.get('bytes', 0)
                })
                
                # 프로토콜 분포
                protocol = flow.get('protocol', 'unknown')
                if protocol not in rmon_data['protocol_dist']:
                    rmon_data['protocol_dist'][protocol] = {
                        'packets': 0,
                        'octets': 0
                    }
                
                rmon_data['protocol_dist'][protocol]['packets'] += flow.get('packets', 0)
                rmon_data['protocol_dist'][protocol]['octets'] += flow.get('bytes', 0)
        
        return rmon_data
```

## 6. 결론

RMON은 네트워크 모니터링의 패러다임을 근본적으로 변화시킨 혁신적인 기술입니다. 개별 장치 관리에서 네트워크 세그먼트 전체의 통합적 모니터링으로의 전환은 현대 네트워크 운영의 기초를 마련했습니다.

RMON1의 데이터 링크 계층 모니터링은 네트워크의 물리적 상태와 기본 통계를 이해하는 강력한 도구를 제공했습니다. 통계 그룹의 상세한 오류 분석, 이력 그룹의 시간 기반 추세 모니터링, 알람 그룹의 사전 예방적 관리, 호스트와 매트릭스 그룹의 트래픽 패턴 분석은 모두 네트워크 문제 해결과 성능 최적화에 필수적인 기능입니다.

RMON2의 프로토콜 계층 확장은 모니터링의 범위를 근본적으로 확장했습니다. 프로토콜 디렉토리를 통한 계층적 프로토콜 식별, 프로토콜 분포 분석, 네트워크 계층 호스트 모니터링, 주소 매핑 관리 등은 응용 프로그램 수준의 가시성을 제공하며, 현대 네트워크의 복잡성을 효과적으로 관리할 수 있는 기반을 마련했습니다.

RMON의 가장 중요한 설계 원칙은 **로컬 처리**와 **지능적 필터링**입니다. 네트워크 프로브에서 데이터를 수집하고 처리함으로써 관리 스테이션으로의 대역폭 소비를 최소화하면서도 풍부한 분석 정보를 제공합니다. 이는 대규모 네트워크 환경에서의 확장성을 보장하는 핵심 메커니즘입니다.

현대 네트워크 환경에서 RMON은 sFlow, NetFlow, 클라우드 모니터링 시스템과 통합되어 새로운 생명력을 얻고 있습니다. 이러한 통합은 각 기술의 강점을 결합하여 종합적인 네트워크 가시성을 제공합니다. RMON의 상세한 진단 기능, sFlow의 실시간 샘플링, NetFlow의 흐름 기반 분석, 클라우드 메트릭의 확장성은 상호 보완적으로 작동합니다.

네트워크 성능 관리(NPM)와 보안 정보 및 이벤트 관리(SIEM) 시스템에서 RMON의 개념과 원리는 여전히 핵심적인 역할을 합니다. 프로토콜 분석, 호스트 발견, 트래픽 패턴 인식, 이상 탐지 등은 모두 RMON이 처음으로 체계화한 개념들입니다.

앞으로 RMON은 소프트웨어 정의 네트워킹(SDN), 네트워크 기능 가상화(NFV), 5G 네트워크, IoT 환경에서도 그 가치를 발휘할 것입니다. 가상화된 네트워크 기능에서의 모니터링, 분산 에지 컴퓨팅 환경의 가시성, 대규모 IoT 디바이스 관리 등 새로운 도전 과제에 RMON의 원칙과 아키텍처는 여전히 유효한 솔루션을 제공할 수 있습니다.

RMON은 단순한 모니터링 프로토콜이 아닌, 네트워크 인텔리전스를 구현하는 철학적 프레임워크입니다. 네트워크가 수동적으로 관리되는 객체들의 집합에서 능동적으로 자신을 모니터링하고 보고하는 지능적 시스템으로 진화하는 길을 제시했습니다. 이는 오늘날의 자동화된 네트워크 운영, AI 기반 네트워크 관리, 예측적 유지보수의 선구자적 역할을 했습니다.