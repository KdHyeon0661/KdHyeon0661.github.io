---
layout: post
title: PowerBuilder - 파워빌더 소스 버전 관리 및 유지보수
date: 2025-12-14 19:30:23 +0900
category: PowerBuilder
---
# SVN/Git을 활용한 파워빌더 소스 버전 관리 및 유지보수/차세대 전환 전략

PowerBuilder로 개발된 시스템을 운영하다 보면, 여러 개발자가 동시에 작업해야 하는 상황이 발생합니다. 특히 PBL(PowerBuilder Library)은 바이너리 파일이기 때문에 일반적인 텍스트 기반 버전 관리 시스템으로는 효과적인 형상 관리가 어렵습니다. 이 글에서는 SVN과 Git을 이용하여 PowerBuilder 프로젝트의 소스 버전을 관리하는 방법과, 객체 단위 Check-in/Check-out 전략을 통한 충돌 방지 기법, 그리고 장기적인 유지보수와 차세대 시스템 전환 시 고려해야 할 사항을 상세히 소개합니다.

---

## 1. PowerBuilder PBL의 버전 관리 특성과 도전 과제

PowerBuilder의 객체(윈도우, DataWindow, 메뉴, 함수 등)는 모두 PBL이라는 바이너리 파일에 저장됩니다. 이로 인해 다음과 같은 문제가 발생합니다.

- **바이너리 파일의 비호환성**: SVN이나 Git은 기본적으로 텍스트 파일의 변경 내역을 비교(merge)할 수 있지만, 바이너리 파일은 변경 내용을 자동으로 병합할 수 없습니다.
- **동시 편집 충돌**: 여러 개발자가 같은 PBL 내의 서로 다른 객체를 수정하더라도, PBL 자체가 변경되었기 때문에 충돌로 간주되어 버전 관리 시스템에서 거부됩니다.
- **히스토리 추적의 어려움**: 객체 단위로 변경 이력을 추적하기 어렵고, 누가 언제 어떤 객체를 수정했는지 파악하기 까다롭습니다.

이러한 문제를 해결하기 위해 PowerBuilder는 자체적인 **소스 제어 연동 기능(Object Cycle)**을 제공하며, SVN이나 Git과 같은 외부 형상 관리 도구와 연동할 수 있는 인터페이스(MSSCCI, PBNative, Git 통합)를 지원합니다.

---

## 2. SVN/Git과 PowerBuilder의 연동 방식

### 2.1 소스 제어 인터페이스 종류

PowerBuilder에서 사용할 수 있는 소스 제어 인터페이스는 다음과 같습니다.

| 인터페이스 | 설명 | 지원 버전 |
|-----------|------|----------|
| **MSSCCI (Microsoft Source Code Control Interface)** | Microsoft 공급업체 표준 인터페이스. Visual SourceSafe, SVN(win32svn) 등에서 사용 가능 | PowerBuilder 6.0 이상 |
| **PBNative** | PowerBuilder 자체 네이티브 인터페이스. SVN, Git 등과 직접 연동 | PowerBuilder 2017 R3 이상 (Git 통합) |
| **Git** | PowerBuilder 2017 R3부터 Git을 기본 지원. 별도 인터페이스 없이 Git 저장소 직접 연결 | PowerBuilder 2017 R3 이상 |

### 2.2 SVN과 PowerBuilder 연동 설정

SVN을 사용할 경우, 일반적으로 **TortoiseSVN**과 같은 클라이언트와 MSSCCI 드라이버(예: PushOK의 SVN SCC)를 함께 사용합니다. 또는 PowerBuilder 외부에서 파일 기반으로 관리하는 방법도 있습니다.

**MSSCCI 방식 설정 단계**:
1. SVN 클라이언트와 SVN SCC 드라이버 설치
2. PowerBuilder에서 메뉴 **File → Connect to Source Control** 선택
3. MSSCCI 공급자로 SVN 선택 후 저장소 URL 지정
4. 각 PBL을 소스 제어에 추가

### 2.3 Git과 PowerBuilder 연동 (Native Git 통합)

PowerBuilder 2017 R3 이상에서는 **File → Source Control → Git** 메뉴를 통해 Git 저장소를 직접 연결할 수 있습니다. 초기화, 커밋, 푸시, 풀 등의 기본 Git 명령을 IDE 내에서 수행할 수 있습니다.

> **💡 핵심 특징**  
> IDE 내에서 Git 커밋(Commit)을 수행하면, PowerBuilder가 알아서 무거운 바이너리 파일(.pbl)을 해체하여 **텍스트 기반의 원시 소스 파일(.srw, .srd 등)**로 변환한 뒤 Git에 올립니다. 반대로 풀(Pull)을 받으면 텍스트 소스를 가져와서 다시 .pbl로 자동 조립해 줍니다.  
> 덕분에 개발자들은 수동으로 Export/Import를 할 필요 없이, 일반 Java나 C# 코드처럼 파일 단위의 이력 추적과 코드 비교(Diff)를 완벽하게 수행할 수 있습니다.

---

## 3. 객체 단위 Check-in/Check-out 충돌 방지 전략

### 3.1 PowerBuilder의 소스 제어 동작 방식

PowerBuilder에서 소스 제어를 활성화하면, PBL 내의 각 객체는 **개별 파일**로 저장소에 저장됩니다(예: `w_main.srw`, `d_customer.srd`). 개발자가 객체를 수정하려면 **Check Out**하여 자신의 작업 공간에 복사본을 가져오고, 수정 완료 후 **Check In**하여 변경 내용을 저장소에 반영합니다.

이 방식의 장점은 객체 단위로 변경 이력을 추적할 수 있고, 여러 개발자가 다른 객체를 동시에 수정해도 충돌이 발생하지 않는다는 점입니다.

### 3.2 효과적인 Check-in/Check-out 정책

#### 3.2.1 사전 예약(Exclusive Lock) 모드 사용
- **Exclusive Checkout**: 한 개발자가 객체를 Check Out하면 다른 개발자는 해당 객체를 Check Out할 수 없습니다. 충돌을 원천적으로 방지합니다.
- 설정 방법: **File → Source Control → Source Control Properties**에서 체크아웃 모드를 Exclusive로 지정.

#### 3.2.2 공유(Shared) 모드와 병합 정책
- **Shared Checkout**: 여러 개발자가 동시에 같은 객체를 Check Out할 수 있습니다. 이 경우, 먼저 Check In한 개발자의 변경 사항이 우선되며, 나중에 Check In하는 개발자는 충돌을 수동으로 해결해야 합니다.
- PowerBuilder는 객체의 텍스트 표현(.srw)을 비교하여 병합을 시도할 수 있지만, 복잡한 객체는 수동 병합이 필요합니다.

#### 3.2.3 권장 사항
- **팀 규모가 작거나 객체 중요도가 높은 경우**: Exclusive Lock 사용
- **팀 규모가 크고 객체 수정 빈도가 높은 경우**: Shared Lock + 수시 업데이트 정책

### 3.3 충돌 방지를 위한 실무 팁

1. **작업 전 항상 Get Latest Version** (Update): 저장소의 최신 버전을 받아서 작업 시작
2. **작업 단위는 작게 유지**: 하나의 객체에 대해 오랜 기간 Check Out 상태를 유지하지 말고, 완료되면 즉시 Check In
3. **Check In 전에 다시 Update**: 다른 사람이 Check In한 내용이 있는지 확인하고, 충돌이 있으면 로컬에서 해결 후 Check In
4. **PBL을 기능별로 분할**: 하나의 큰 PBL보다 여러 개의 작은 PBL로 나누어 개발자 간 작업 영역을 분리
5. **주석(Comment) 작성 의무화**: Check In 시 변경 사항에 대한 명확한 설명을 남겨 히스토리 추적 용이

### 3.4 과거의 수동 Export/Import 방식과 최신 자동화

**과거에는** 소스 제어를 위해 PowerBuilder 객체를 일일이 텍스트 파일(.srw, .srd 등)로 내보내기(Export)하고, 변경 후 다시 가져오기(Import)하는 번거로운 과정이 필요했습니다. 또한 객체 내에 포함된 그림 등 바이너리 정보는 별도로 관리해야 했습니다.

**하지만 최신 PowerBuilder(2017 R3 이상)의 Native Git 통합을 사용하면 이러한 과정이 완전히 자동화됩니다.** 개발자는 단지 Git 명령(커밋, 풀, 푸시)만 수행하면, PowerBuilder가 내부적으로 PBL을 해체하여 텍스트 소스로 변환하고, 다시 조립하는 작업을 대신 처리해 줍니다. 따라서 개발자는 형상 관리 도구의 장점을 최대한 활용하면서도 복잡한 Export/Import 작업에서 해방될 수 있습니다.

---

## 4. 유지보수 단계에서의 버전 관리 전략

### 4.1 브랜치 전략

PowerBuilder 프로젝트에서도 Git Flow와 같은 브랜치 전략을 적용할 수 있습니다.

- **master(main)**: 항상 배포 가능한 안정 버전 유지
- **develop**: 다음 릴리스를 위한 통합 브랜치
- **feature/xxx**: 새로운 기능 개발 브랜치 (develop에서 분기)
- **hotfix/xxx**: 긴급 버그 수정 브랜치 (master에서 분기)
- **release/xxx**: 릴리스 준비 브랜치

**예시 텍스트 다이어그램**:
```
master   ●─────────●─────────●─────────●
         ↑         ↑         ↑         ↑
         hotfix    hotfix    release   master(배포)
develop  ●─────●─────●─────●─────●─────●
               ↑     ↑           ↑
            feature feature    feature
```

### 4.2 릴리스 태깅

각 배포 버전에는 반드시 태그를 부여합니다. Git에서는 `git tag -a v1.0.0 -m "릴리스 v1.0.0"` 형태로 태그를 생성합니다. 이를 통해 특정 시점의 소스 상태를 쉽게 복원할 수 있습니다.

### 4.3 유지보수 시나리오별 대응

| 상황 | 대응 전략 |
|------|----------|
| 운영 중 버그 발생 | master에서 hotfix 브랜치 생성 → 수정 → master에 병합 → 태그 생성 → 배포 |
| 신규 기능 개발 | develop에서 feature 브랜치 생성 → 개발 완료 후 develop에 병합 |
| 여러 버전 동시 지원 | 각 버전별로 브랜치를 유지하고, 중요한 수정 사항은 cherry-pick |

---

## 5. 차세대 전환 전략과 버전 관리

### 5.1 차세대 시스템 전환 시나리오

레거시 PowerBuilder 시스템을 현대화하는 방법은 다양합니다.

1. **PowerBuilder 버전 업그레이드**: 최신 버전으로 마이그레이션 (예: 12.5 → 2022)
2. **점진적 재개발**: 특정 모듈을 .NET, Java, Python 등으로 재개발하고, 기존 시스템과 웹 서비스로 연동
3. **완전 재개발**: 전체 시스템을 새로운 플랫폼으로 재구축
4. **UI 현대화**: PowerBuilder를 백엔드로 두고, 프론트엔드를 웹으로 전환 (예: PowerBuilder REST API + React)

### 5.2 전환 과정에서의 버전 관리

- **별도 저장소 운영**: 신규 시스템은 기존 레거시와 별도의 저장소에서 관리
- **공통 모듈 분리**: 두 시스템에서 공통으로 사용하는 비즈니스 로직이 있다면, 별도 라이브러리로 분리하여 버전 관리
- **데이터베이스 스키마 변경 관리**: Flyway, Liquibase 등 마이그레이션 도구를 활용하여 스키마 변경 이력을 버전 관리

### 5.3 레거시 코드 보존 전략

차세대 시스템으로 전환되더라도, 기존 레거시 시스템의 소스 코드는 일정 기간 보존해야 합니다. 다음 사항을 고려하세요.

- 저장소 아카이브(예: 태그 또는 브랜치로 `legacy-final` 생성)
- 관련 문서(설계서, 배포 매뉴얼) 함께 보관
- 법적 요구사항에 따른 보존 기간 확인

---

## 6. 결론

PowerBuilder 프로젝트에서 SVN이나 Git을 활용한 소스 버전 관리는 단순히 파일 백업이 아니라, 팀 협업의 효율성과 시스템 안정성을 결정짓는 핵심 요소입니다.

- **객체 단위 Check-in/Check-out**을 통해 충돌을 방지하고, 명확한 이력을 남기세요.
- **브랜치 전략과 태깅**을 통해 배포 관리와 유지보수를 체계화하세요.
- **최신 PowerBuilder의 Native Git 통합**을 이용하면 수동 Export/Import 없이도 텍스트 기반의 완전한 소스 관리가 가능합니다.
- 차세대 전환 시에도 버전 관리 체계를 일관되게 유지하여 위험을 최소화하세요.

도입 초기에는 다소 번거로울 수 있지만, 장기적으로 보면 유지보수 비용 절감과 시스템 신뢰도 향상에 큰 도움이 됩니다. 지금부터라도 체계적인 버전 관리 문화를 정착시켜, 더 안전하고 효율적인 PowerBuilder 개발 환경을 만들어 나가시길 바랍니다.