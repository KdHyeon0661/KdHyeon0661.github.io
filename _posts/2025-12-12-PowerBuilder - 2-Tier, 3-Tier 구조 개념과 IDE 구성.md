---
layout: post
title: PowerBuilder - 2-Tier, 3-Tier 구조 개념과 IDE 구성
date: 2025-12-12 15:30:23 +0900
category: PowerBuilder
---
# 2-Tier / 3-Tier 구조 개념과 PowerBuilder IDE 구성

PowerBuilder로 애플리케이션을 개발할 때, 아키텍처에 대한 이해는 효율적인 설계와 유지보수에 필수적입니다. 이 글에서는 PowerBuilder가 지원하는 전통적인 2-Tier(클라이언트-서버) 구조와 현대적인 3-Tier(분산) 구조의 개념을 살펴보고, 실제 개발 작업이 이루어지는 IDE(통합 개발 환경)의 구성 요소인 Workspace, Target, Painter에 대해 상세히 알아보겠습니다.

---

## 2-Tier / 3-Tier 구조 개념

### 2-Tier 클라이언트-서버 구조

2-Tier 구조는 가장 전통적인 형태의 클라이언트-서버 아키텍처입니다. 사용자 인터페이스와 비즈니스 로직을 클라이언트(1-Tier)에 두고, 데이터베이스 서버(2-Tier)가 데이터를 관리하는 방식입니다.

```
[클라이언트 Tier]
   ┌─────────────────────┐
   │  UI (Window, Menu)  │
   │  비즈니스 로직       │
   │  DataWindow (SQL)   │
   └──────────┬──────────┘
              │ (네트워크)
              ▼
[데이터베이스 Tier]
   ┌─────────────────────┐
   │   DBMS (데이터 저장,  │
   │   SQL 처리, 인덱스)   │
   └─────────────────────┘
```

#### 특징

- 클라이언트가 데이터베이스에 직접 연결(SQLCA 트랜잭션 객체 사용)
- DataWindow가 SQL을 생성하고 결과를 직접 처리
- 주로 소규모 애플리케이션이나 LAN 환경에서 사용

#### 장점

- 개발이 단순하고 직관적
- 빠른 응답 시간(데이터베이스와 직접 통신)
- 구현 비용이 낮음

#### 단점
- 클라이언트에 비즈니스 로직이 집중되어 배포 및 유지보수가 어려움
- 데이터베이스 연결 수가 증가하면 서버 부하 가중
- 보안 취약(클라이언트가 DB 계정 정보를 가짐)
- 인터넷 환경에 부적합

PowerBuilder의 초기 버전은 이러한 2-Tier 구조에 최적화되어 있었으며, 지금도 많은 내부 시스템이 이 방식으로 운영됩니다.

---

### 3-Tier 분산 구조

3-Tier 구조는 클라이언트, 애플리케이션 서버(미들웨어), 데이터베이스 서버의 세 계층으로 분리한 아키텍처입니다. PowerBuilder는 초기에는 EAServer와 Distributed PowerBuilder를 통해 분산 환경을 지원했으나, 현재는 이러한 기술들이 공식적으로 단종(EOL)되었고, **Appeon PowerServer**가 그 역할을 대체하고 있습니다.

#### 최신 PowerBuilder 3-Tier: PowerServer

**PowerServer**는 PowerBuilder 애플리케이션을 그대로 웹/모바일/클라우드 환경으로 전환해주는 솔루션입니다. 기존에 작성한 PowerBuilder 소스 코드(DataWindow, 윈도우, 비즈니스 로직)를 **C# REST API**로 자동 변환하여, 별도의 재개발 없이 3-Tier 또는 클라우드 네이티브 아키텍처로 마이그레이션할 수 있습니다.

- **작동 방식**: PowerBuilder 클라이언트(런타임 패키지)가 PowerServer에 RESTful 요청을 보내면, PowerServer가 데이터베이스와 통신하고 결과를 JSON 형태로 반환합니다.
- **지원 환경**: 온프레미스, AWS, Azure, 도커 컨테이너 등
- **기존 코드 재사용**: 기존 PowerBuilder 코드와 DataWindow의 비즈니스 로직을 최대한 재사용하면서, 현대적인 분산 환경의 이점을 얻을 수 있습니다.

> **참고**: 과거 Sybase 시절에는 EAServer와 Distributed PowerBuilder가 사용되었으나, 현재는 공식적으로 지원이 중단되었습니다. 새로운 3-Tier 프로젝트는 반드시 PowerServer를 기반으로 설계해야 합니다.

#### PowerServer 기반 3-Tier 아키텍처의 장단점

PowerServer를 도입한 3-Tier 구조는 다음과 같은 특징과 장단점을 가집니다.

| 구분 | 내용 |
|------|------|
| **클라이언트 역할** | UI와 간단한 입력 검증만 담당 (Thin Client) |
| **애플리케이션 서버 역할** | 비즈니스 로직 실행, 세션 관리, 보안, REST API 노출 |
| **데이터베이스 역할** | 순수 데이터 저장 및 처리 |
| **통신 프로토콜** | HTTP/HTTPS, REST, JSON |

**장점**
- **보안 강화**: 데이터베이스 접속 정보가 서버에만 존재하므로 클라이언트 유출 위험이 없습니다.
- **확장성**: 애플리케이션 서버를 여러 대로 확장하여 사용자 증가에 대응할 수 있습니다.
- **유지보수 용이**: 비즈니스 로직 변경 시 서버만 업데이트하면 되므로 배포가 간편합니다.
- **멀티플랫폼 지원**: 동일한 비즈니스 로직을 기반으로 웹, 모바일, 데스크톱 클라이언트를 개발할 수 있습니다.
- **기존 투자 보호**: 기존 PowerBuilder 코드와 DataWindow를 재사용하므로 마이그레이션 비용이 절감됩니다.

**단점**
- **초기 설계 복잡성**: 아키텍처 설계와 PowerServer 구성에 대한 학습이 필요합니다.
- **네트워크 레이턴시**: 클라이언트-서버 간 통신이 추가되어 응답 시간이 다소 증가할 수 있습니다.
- **인프라 비용**: 애플리케이션 서버와 라이선스 비용이 추가로 발생할 수 있습니다.

---

### PowerBuilder에서의 아키텍처 선택

애플리케이션의 규모, 사용자 수, 보안 요구사항, 배포 환경에 따라 2-Tier 또는 3-Tier를 선택합니다.

- **2-Tier**: 부서 단위 소규모 시스템, 내부 인트라넷, 빠른 개발이 필요한 경우 적합
- **3-Tier (PowerServer)**: 전사적 시스템, 인터넷 노출, 높은 보안과 확장성, 멀티플랫폼(웹/모바일)이 필요한 경우 적합

PowerBuilder는 두 아키텍처를 모두 지원하며, 동일한 DataWindow 기술을 그대로 사용할 수 있어 마이그레이션도 비교적 용이합니다.

---

## IDE 구성: Workspace, Target, Painter

PowerBuilder 개발 환경은 효율적인 프로젝트 관리를 위해 계층적으로 구성되어 있습니다.

### Workspace (작업 공간)

Workspace는 관련된 여러 Target(프로젝트)을 그룹화하는 최상위 컨테이너입니다. 하나의 Workspace에는 여러 Target이 포함될 수 있으며, 동시에 열어서 작업할 수 있습니다.

- **파일 확장자**: `.pbw`
- **역할**: 개발자가 동시에 작업할 여러 애플리케이션이나 컴포넌트를 논리적으로 묶음
- 예: 클라이언트 애플리케이션 Target과 공통 라이브러리 Target을 하나의 Workspace에 포함

Workspace를 사용하면 여러 Target 간의 의존성 관리, 함께 빌드, 통합 디버깅이 가능합니다.

```
[Workspace]
    │
    ├─ Target A (주문 처리 애플리케이션)
    │      ├─ Application Object
    │      ├─ Window 객체들
    │      ├─ DataWindow 객체들
    │      └─ ...
    │
    └─ Target B (공통 유틸리티 라이브러리)
           ├─ User Object
           ├─ Function 라이브러리
           └─ ...
```

### Target (대상)

Target은 빌드 단위로, 실행 파일이나 컴포넌트를 생성하는 데 필요한 모든 객체와 설정을 포함합니다. PowerBuilder에서는 여러 유형의 Target을 생성할 수 있습니다.

- **Application Target**: 전통적인 PowerBuilder 실행 파일(.exe) 또는 DLL을 생성하는 대상
- **.NET Target**: .NET 어셈블리 또는 윈도우 폼 애플리케이션을 생성하는 대상
- **PowerServer Target**: PowerServer용 클라우드 배포 패키지를 생성하는 대상
- **Template Target**: 미리 정의된 패턴으로 신속한 개발을 지원

Target은 하나 이상의 라이브러리(PBL 파일)로 구성되며, 라이브러리 검색 경로를 통해 다른 Target의 객체를 참조할 수 있습니다.

**Target 속성**에서는 다음을 설정합니다.
- 컴파일러 옵션
- 디버그 정보 생성 여부
- 실행 파일 이름 및 아이콘
- 버전 정보

#### PBL(Library)의 역할
PowerBuilder의 모든 객체(윈도우, DataWindow, 메뉴 등)는 반드시 **PBL 파일(.pbl)** 안에 저장됩니다. PBL은 단순한 폴더가 아니라 객체들이 컴파일되어 압축된 바이너리 파일입니다. 따라서 System Tree에서는 Target 아래에 PBL 계층이 존재하며, 그 아래에 개별 객체가 위치합니다.

다음은 실제 System Tree 구조를 올바르게 표현한 예시입니다.

```
System Tree 예시 (PBL 계층 포함)
MyWorkspace (Workspace)
├─ MyApp (Target)
│  ├─ myapp_main.pbl (Library)
│  │  ├─ Application (MyApp)
│  │  └─ w_main (Window)
│  └─ myapp_dw.pbl (Library)
│     ├─ d_customer (DataWindow)
│     └─ d_orders (DataWindow)
└─ CommonLib (Target)
   └─ common.pbl (Library)
      ├─ u_utility (User Object)
      └─ f_global (Function)
```

### Painter (화가)

Painter는 PowerBuilder IDE에서 특정 유형의 객체를 시각적으로 설계하고 편집하는 도구입니다. 각 Painter는 고유한 편집기와 도구 모음을 제공합니다.

> **왜 'Painter'일까?**  
> PowerBuilder는 코드를 '타이핑'해서 화면을 그리는 방식이 아니라, 마우스로 캔버스에 그림을 그리듯(Paint) 객체를 배치하고 속성을 지정합니다. 이러한 시각적 개발 철학 때문에 모든 편집기 이름이 Painter로 끝납니다.

#### 주요 Painter 종류

| Painter | 용도 | 주요 기능 |
|---------|------|----------|
| **Application Painter** | 애플리케이션 객체 편집 | 라이브러리 경로, 기본 폰트, 이벤트 핸들러 작성 |
| **Window Painter** | 윈도우(폼) 디자인 | 컨트롤 배치, 속성 설정, 이벤트 스크립트 작성 |
| **DataWindow Painter** | DataWindow 객체 생성 | 데이터 소스 정의, 표시 스타일 선택, 컬럼 속성 설정 |
| **Menu Painter** | 메뉴 디자인 | 메뉴 항목, 도구 모음, 이벤트 연결 |
| **User Object Painter** | 재사용 가능한 커스텀 객체 | 시각적/비시각적 사용자 객체 생성 |
| **Function Painter** | 전역 함수 정의 | 반환형, 인자, 코드 작성 |
| **Structure Painter** | 데이터 구조 정의 | 여러 필드를 묶는 구조체 생성 |
| **Query Painter** | SQL 쿼리 시각적 작성 | 그래픽 방식으로 SELECT 문 생성 |
| **Pipeline Painter** | 데이터 파이프라인 정의 | 데이터베이스 간 데이터 복사/이관 |

각 Painter는 다음과 같은 공통 요소를 가집니다.

- **레이아웃 영역**: 객체를 시각적으로 배치
- **속성 시트(Properties)**: 선택한 객체의 속성 편집
- **이벤트 목록(Events)**: 객체에 연결된 이벤트와 스크립트 편집
- **함수 목록(Functions)**: 객체 레벨 함수 정의

### System Tree와의 통합

Painter는 주로 메인 편집 영역에 열리지만, 모든 객체는 **System Tree**를 통해 탐색하고 열 수 있습니다. System Tree는 Workspace 내의 모든 Target, Library(PBL), 그리고 그 안의 객체들을 트리 구조로 보여주며, 드래그 앤 드롭, 오른쪽 클릭 메뉴 등으로 Painter를 실행합니다.

---

## 결론

PowerBuilder의 아키텍처 이해(2-Tier와 3-Tier)는 올바른 시스템 설계의 기초이며, IDE의 구성 요소(Workspace, Target, Painter)는 개발 생산성과 프로젝트 관리에 직접적인 영향을 줍니다.

- **2-Tier**는 단순하고 빠른 개발이 필요한 환경에, **3-Tier(PowerServer)**는 확장성과 보안, 멀티플랫폼 지원이 중요한 환경에 적합합니다.
- **Workspace**는 여러 프로젝트를 통합 관리하고, **Target**은 빌드 단위로서 애플리케이션의 구성을 정의합니다. 모든 객체는 **PBL**이라는 바이너리 라이브러리 안에 저장됩니다.
- **Painter**들은 객체 지향적인 시각적 개발을 가능하게 하여, 코드 작성보다는 설계에 집중할 수 있도록 돕습니다. 그 이름처럼 마우스로 그림을 그리듯 애플리케이션을 완성해 나가는 것이 PowerBuilder의 핵심 철학입니다.