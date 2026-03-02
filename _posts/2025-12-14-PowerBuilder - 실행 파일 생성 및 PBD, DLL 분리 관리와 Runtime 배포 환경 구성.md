---
layout: post
title: PowerBuilder - 실행 파일 생성 및 PBD, DLL 분리 관리와 Runtime 배포 환경 구성
date: 2025-12-14 18:30:23 +0900
category: PowerBuilder
---
# 실행 파일(EXE) 생성 및 PBD/DLL 분리 관리와 Runtime 배포 환경 구성

PowerBuilder 애플리케이션을 개발한 후, 최종 사용자에게 배포하기 위해서는 실행 파일(EXE)을 생성하고 필요한 런타임 파일들과 함께 패키징해야 합니다. 또한 대규모 애플리케이션의 경우 유지보수와 로딩 성능을 고려하여 PBD 또는 DLL로 모듈을 분리하는 것이 일반적입니다. 이 글에서는 PowerBuilder에서 실행 파일을 생성하는 방법부터 PBD/DLL 분리 전략, 그리고 실제 운영 환경에 배포하기 위한 런타임 구성 요소까지 상세히 알아보겠습니다.

---

## 1. 실행 파일(EXE) 생성

PowerBuilder 프로젝트를 컴파일하여 실행 파일을 만드는 과정은 Target의 속성에서 시작됩니다.

### 1.1 Target 컴파일 방식 선택

PowerBuilder는 두 가지 컴파일 방식을 지원합니다.

| 방식 | 설명 | 장단점 |
|------|------|--------|
| **P-code (Pseudocode)** | 중간 코드로 컴파일, 인터프리터 방식 | 파일 크기 작음, 디버깅 용이, 속도 느림 |
| **Machine Code** | 네이티브 기계어로 컴파일 | 실행 속도 빠름, 파일 크기 큼, 보안성 높음 |

**선택 기준**:
- 개발 중 디버깅이 필요하면 P-code
- 최종 배포 시 성능이 중요하면 Machine Code
- Machine Code로 컴파일하려면 C++ 컴파일러가 설치되어 있어야 함 (Visual C++ 등)

### 1.2 Project 객체 생성

1. **File → New** → **Project** 탭
2. 원하는 프로젝트 유형 선택:
   - **Application**: 일반적인 EXE 생성 (P-code)
   - **Application (Machine Code)**: 기계어 EXE 생성
3. Project Painter에서 다음 설정:
   - **Executable File Name**: 생성될 EXE 파일 경로
   - **Resource File**: 리소스 파일(PBR) 지정 (선택)
   - **Library Search Path**: 참조할 PBL 경로
   - **Version 정보**: 회사명, 제품명, 버전, 저작권 등
   - **Icon**: EXE 아이콘 파일

### 1.3 컴파일 실행

- **Project Painter**에서 **Build** 버튼 클릭
- 또는 메뉴 **Run → Build Project**
- 컴파일 로그를 통해 오류 확인

### 1.4 Resource File(PBR)의 중요성

PBR(PowerBuilder Resource) 파일은 EXE에 포함할 추가 리소스(이미지, 아이콘, DataWindow 객체 등)를 지정합니다. DataWindow 객체를 코드에서 동적으로 참조하거나, Picture 컨트롤에 사용할 이미지 파일 등을 PBR에 등록하면 EXE에 포함되어 별도 파일로 배포하지 않아도 됩니다.

**PBR 파일 예시**:
```
# 이미지 파일
logo.bmp
button.bmp

# DataWindow 객체 (라이브러리 형태)
d_customer_list
d_orders
```

PBR 파일을 작성한 후 Project의 Resource File 항목에 지정합니다.

---

## 2. PBD/DLL 분리 관리

대규모 애플리케이션에서는 모든 객체를 하나의 EXE에 포함시키면 파일 크기가 커지고, 수정 시 전체를 재배포해야 하는 단점이 있습니다. 따라서 기능별로 PBL을 나누고, 각 PBL을 PBD 또는 DLL로 분리하여 배포하는 전략을 사용합니다.

### 2.1 PBD (PowerBuilder Dynamic Library)

PBD는 P-code로 컴파일된 동적 라이브러리입니다. EXE와 함께 배포되며, 실행 중에 필요할 때 로드됩니다.

**특징**:
- P-code 기반, 기계어 DLL보다 작음
- EXE와 동일한 디렉토리나 검색 경로에 위치
- 수정 시 해당 PBD만 교체하면 됨

### 2.2 DLL (Dynamic Link Library)

Machine Code로 컴파일된 DLL입니다. 순수 기계어이므로 실행 속도가 빠르고, C++ 등 다른 언어와의 연동이 가능합니다.

**특징**:
- Machine Code로 컴파일
- 성능 중요 모듈에 적합
- PBD보다 크기가 큼

### 2.3 분리 전략

1. **라이브러리 분할**: PBL을 기능별로 분리 (예: `base.pbl`, `customer.pbl`, `order.pbl`, `report.pbl`)
2. **Project 설정**: 각 PBL을 개별 프로젝트로 컴파일하거나, 하나의 프로젝트에서 PBD 생성 옵션을 지정
3. **PBD 생성 방법**:
   - **Project Painter(프로젝트 객체 설정 창)를 열고 Libraries 탭으로 이동합니다.**
   - **목록에 나타난 각 PBL 파일 옆의 PBD/DLL 체크박스를 체크한 후 Build를 실행하면, 해당 PBL들은 EXE에 통합되지 않고 독립된 PBD(또는 DLL) 파일로 자동 생성됩니다.**
   - 또는 각 PBL을 별도의 Project로 컴파일하여 PBD를 생성할 수도 있습니다.

### 2.4 실행 파일과 PBD의 관계

EXE는 기본적으로 하나의 주 PBL(Application PBL)을 포함하고, 나머지 PBL은 PBD 형태로 배포됩니다. 실행 시 EXE는 시스템 경로나 애플리케이션 디렉토리에서 PBD를 찾습니다.

```
[애플리케이션 디렉토리]
   ├── MyApp.exe
   ├── base.pbd
   ├── customer.pbd
   ├── order.pbd
   ├── report.pbd
   └── (기타 런타임 DLL들)
```

### 2.5 분리 시 장점

- **모듈화**: 기능별로 분리되어 유지보수 용이
- **패치 용이성**: 일부 모듈만 교체하여 배포 가능
- **초기 로딩 시간 단축**: 필요한 PBD만 메모리에 로드
- **보안**: 일부 로직을 DLL로 분리하여 보안 강화

---

## 3. Runtime 배포 환경 구성

개발이 완료된 애플리케이션을 최종 사용자에게 배포하려면 PowerBuilder 런타임 파일이 함께 설치되어야 합니다. 런타임 파일은 PowerBuilder 버전에 따라 다르며, 필수 DLL과 선택적 구성 요소로 나뉩니다.

### 3.1 PowerBuilder 런타임 파일 개요

PowerBuilder로 개발된 애플리케이션을 실행하려면 다음 구성 요소가 필요합니다.

- **PowerBuilder Runtime DLL**: PowerBuilder 가상 머신, 데이터베이스 드라이버 등
- **Database Client Software**: 연결할 DBMS의 클라이언트 라이브러리
- **추가 DLL**: OLE, 웹 서비스 등 사용 시 필요한 라이브러리

### 3.2 버전별 런타임 파일 (PowerBuilder 2019/2022 예)

일반적으로 필요한 주요 DLL:

| 파일명 | 설명 |
|--------|------|
| `pbvm190.dll` (또는 `pbvm120.dll` 등) | PowerBuilder 가상 머신 (핵심) |
| `pbdwe190.dll` | DataWindow 엔진 |
| `pbrtc190.dll` | 런타임 컨트롤 |
| `libjcc.dll`, `libjutils.dll` | Java 관련 (필요시) |
| `pborg190.dll` (또는 `pbora190.dll`) | Oracle 드라이버 |
| `pbmss190.dll` | MS SQL Server 드라이버 |
| `pbodb190.dll` | ODBC 드라이버 |
| `pbjag190.dll` | Jaguar/CORBA 지원 |
| `pbsys190.dll` | 시스템 함수 |

파일명의 숫자는 버전을 의미합니다. 예: 190은 2019, 200은 2022 등.

### 3.3 배포 디렉토리 구성

애플리케이션을 설치할 디렉토리를 구성합니다.

```
C:\Program Files\MyApp\
   ├── MyApp.exe
   ├── *.pbd (또는 *.dll)
   ├── *.ini (설정 파일)
   ├── *.pbr (리소스 파일은 EXE에 포함 권장)
   └── Runtime\
       ├── pbvm190.dll
       ├── pbdwe190.dll
       └── ... (기타 런타임 DLL)
```

런타임 DLL은 시스템 PATH에 등록하거나 애플리케이션 디렉토리와 같은 위치에 두어도 됩니다. 단, 여러 PowerBuilder 애플리케이션이 共存하는 경우 충돌을 방지하기 위해 각 애플리케이션 디렉토리에 런타임을 배치하거나, 공통 위치(예: `C:\Windows\System32`)에 두는 방법이 있습니다.

### 3.4 배포 도구 활용

- **InstallShield**, **Advanced Installer** 등의 설치 제작 도구를 사용하여 런타임 파일과 애플리케이션 파일을 패키징할 수 있습니다.
- PowerBuilder 설치 미디어에 포함된 **Runtime Packager**를 사용할 수도 있습니다.

### 3.5 데이터베이스 연결 설정

애플리케이션 배포 시 데이터베이스 연결 정보는 코드에 하드코딩하기보다 INI 파일이나 레지스트리, 환경 변수 등을 통해 외부에서 설정할 수 있도록 하는 것이 좋습니다.

```ini
[Database]
DBMS=ODBC
DBParm=ConnectString='DSN=MyDB;UID=sa;PWD=1234'
```

### 3.6 배포 시 주의사항

1. **런타임 버전 일치**: 개발에 사용된 PowerBuilder 버전과 정확히 일치하는 런타임을 배포해야 합니다.
2. **재배포 가능 파일 라이선스**: 런타임 파일은 **Appeon(아피온)**에서 재배포를 허용하지만, 라이선스 조건을 확인해야 합니다.
3. **Windows 비트 수**: 32비트로 개발된 애플리케이션은 64비트 Windows에서도 실행 가능하지만, 32비트 런타임을 사용해야 합니다. 반대로 64비트로 개발된 경우 64비트 런타임이 필요합니다.
4. **관리자 권한**: 일부 DLL을 시스템 디렉토리에 복사할 때는 관리자 권한이 필요할 수 있습니다.

### 3.7 Runtime Packager 사용 예

PowerBuilder 설치 디렉토리 아래 `Runtime Packager` 유틸리티를 실행하여 필요한 런타임 파일을 선택적으로 추출할 수 있습니다. 추출된 파일을 설치 프로그램에 포함시킵니다.

---

## 4. 실전 배포 예제

### 시나리오
고객 관리 시스템(CMS)을 개발했습니다. 개발 환경: PowerBuilder 2019, 데이터베이스: MS SQL Server. 기능 모듈은 다음과 같이 PBL을 분리했습니다.
- `cms_app.pbl`: 애플리케이션 객체, 메인 윈도우
- `cms_cust.pbl`: 고객 관리 관련 객체
- `cms_order.pbl`: 주문 관리 관련 객체
- `cms_util.pbl`: 공통 함수, 사용자 객체

### 배포 단계

1. **Project 생성**:
   - New → Project → Application (Machine Code) 선택
   - Executable: `C:\Build\cms.exe`
   - Resource File: `cms.pbr` (이미지, DataWindow 리스트)
   - Library Search Path: 각 PBL 경로 순서대로 지정
   - Version 정보 입력

2. **PBD 생성** (선택):
   - Project Painter의 **Libraries 탭**으로 이동
   - 목록에 나타난 각 PBL 파일 옆의 PBD 체크박스를 체크
   - Build 실행 → 체크된 PBL들은 EXE에 통합되지 않고 별도의 PBD 파일로 생성됨
   - 결과: `cms_app.pbd`, `cms_cust.pbd`, `cms_order.pbd`, `cms_util.pbd`

3. **런타임 파일 수집**:
   - `pbvm190.dll`, `pbdwe190.dll`, `pbmss190.dll` (SQL Server 드라이버), `pbodb190.dll` (ODBC) 등 복사

4. **설치 디렉토리 구성**:
   ```
   C:\Program Files\CMS\
      ├── cms.exe
      ├── cms_app.pbd
      ├── cms_cust.pbd
      ├── cms_order.pbd
      ├── cms_util.pbd
      ├── cms.ini
      └── runtime\
          ├── pbvm190.dll
          ├── pbdwe190.dll
          └── ...
   ```

5. **설치 프로그램 작성**:
   - InstallShield 등을 이용해 위 파일들을 대상 디렉토리에 복사
   - 필요시 레지스트리에 경로 등록
   - 바탕화면 바로가기 생성

6. **배포 후 테스트**:
   - 대상 PC에 설치 후 실행 확인
   - 데이터베이스 연결 테스트 (ODBC DSN 설정 또는 연결 문자열 확인)

---

## 5. 결론

PowerBuilder 애플리케이션의 배포는 단순히 EXE 파일만 복사하는 것이 아니라, 런타임 환경과의 정확한 조화가 필요합니다. PBD/DLL 분리 전략을 통해 유지보수성을 높이고, 런타임 파일을 적절히 구성하여 사용자에게 안정적인 애플리케이션을 제공할 수 있습니다.

- **실행 파일 생성**: 프로젝트 설정을 통해 EXE 생성 (P-code 또는 Machine Code)
- **PBD/DLL 분리**: 기능별 PBL을 분리하여 Project의 Libraries 탭에서 PBD 생성 옵션 지정
- **런타임 배포**: PowerBuilder 버전에 맞는 런타임 DLL을 함께 배포하고, 데이터베이스 연결 설정을 외부화

이러한 과정을 체계적으로 수행하면, 개발부터 배포까지 전체 라이프사이클을 효율적으로 관리할 수 있습니다. 다음 글에서는 PowerBuilder 애플리케이션의 성능 최적화 기법에 대해 알아보겠습니다.