---
layout: post
title: PowerBuilder - 설치 및 기본 가이드
date: 2025-12-12 14:30:23 +0900
category: 영상처리
---
# PowerBuilder 설치 및 기본 설정 가이드

이전 글에서는 PowerBuilder의 개념과 특징, 개발 환경 구성 요소를 살펴보았습니다. 이번에는 실제로 PowerBuilder를 설치하고, 개발을 시작하기 위한 기본 설정을 단계별로 알아보겠습니다. PowerBuilder 2022 버전을 기준으로 설명하며, 설치부터 간단한 애플리케이션 실행까지의 과정을 다룹니다.

---

## PowerBuilder 다운로드 및 설치

### 1. 설치 파일 다운로드

PowerBuilder는 현재 **Appeon(아피온)** 사에서 독점적으로 개발 및 배포하고 있습니다.  
공식 웹사이트([https://www.appeon.com/products/powerbuilder](https://www.appeon.com/products/powerbuilder))에서 평가판 또는 라이선스 버전을 다운로드할 수 있습니다.  
Appeon 계정이 필요하며, 기업 사용자는 Appeon 고객 포털을 통해서도 다운로드 가능합니다.

> **주의**: PowerBuilder 2017 이후 버전(2022 포함)은 더 이상 SAP 사이트에서 제공되지 않습니다. 반드시 Appeon 공식 사이트를 이용하세요.

### 2. 설치 프로그램 실행

다운로드한 실행 파일(예: `PowerBuilder2022_xxxx.exe`)을 관리자 권한으로 실행합니다. 설치 언어는 기본적으로 영어를 선택하며, 이후 설치 마법사가 진행됩니다.

### 3. 설치 구성 요소 선택

설치 마법사에서 다음과 같은 구성 요소를 선택할 수 있습니다.

- **PowerBuilder IDE**: 통합 개발 환경 (필수)
- **Database Drivers**: 연결할 DBMS에 맞는 드라이버 (Oracle, MS SQL Server, ODBC 등)
- **SQL Anywhere**: 로컬 개발용 임베디드 데이터베이스 (예제나 테스트에 유용)
- **.NET Assembly Wizards**: .NET 통합 기능
- **Help Files**: 도움말 문서

필요한 항목을 체크하고 다음으로 진행합니다.

### 4. 설치 경로 설정

기본 설치 경로는 `C:\Program Files\Appeon\PowerBuilder2022`입니다. 필요에 따라 변경할 수 있습니다.

### 5. 라이선스 활성화

설치 완료 후 처음 실행하면 라이선스 키를 입력하거나 평가판을 선택합니다. 평가판은 30일간 사용 가능하며, 이후 구매한 라이선스 키를 등록할 수 있습니다. 라이선스 관리도 모두 Appeon 라이선스 서버를 통해 이루어집니다.

---

## 데이터베이스 연결 설정

PowerBuilder의 핵심은 데이터베이스 연동이므로, 개발 환경에서 데이터베이스 연결을 먼저 구성해야 합니다.

### 1. ODBC 데이터 원본 설정 (선택 사항)

ODBC 드라이버를 사용할 경우, Windows의 **ODBC 데이터 원본 관리자**를 열고 시스템 DSN 또는 사용자 DSN을 추가합니다.
```
제어판 → 관리 도구 → ODBC 데이터 원본(64비트/32비트)
   └─ [추가] → SQL Server Native Client 또는 Oracle 드라이버 선택
        → 데이터 원본 이름(DSN) 입력, 서버 지정, 로그인 정보 설정
```

### 2. PowerBuilder 내 Database Painter 실행

PowerBuilder를 실행한 후, 도구 모음에서 **Database** 아이콘을 클릭하거나 메뉴에서 **Tools → Database Painter**를 엽니다.

### 3. 연결 프로필 생성

Database Painter 창에서 연결하려는 DBMS 인터페이스를 오른쪽 클릭하고 **New Profile**을 선택합니다.  
- **ODBC**를 사용할 경우: **ODB ODBC** 선택  
- **네이티브 드라이버**를 사용할 경우: 예를 들어 MS SQL Server는 **MSS Microsoft SQL Server**, Oracle은 **OR8 Oracle8/8i** 이상을 선택합니다. (실무에서는 성능과 기능상 네이티브 드라이버를 권장합니다.)

```text
Profile Name: MyLocalDB (식별 가능한 이름)
Connect Information:
   - Server: (서버명 또는 IP)
   - Database: 데이터베이스 이름
   - User ID: 사용자 계정
   - Password: 비밀번호
```

입력 후 **OK**를 클릭하면 프로필이 생성됩니다.

> **추천 스크린샷 위치**: 이 부분에서 '새 프로필 생성' 대화상자(서버, 데이터베이스, 사용자 ID 입력 창)의 스크린샷을 첨부하면 독자가 실수 없이 따라 할 수 있습니다.

### 4. 연결 테스트

생성된 프로필을 더블클릭하거나 오른쪽 클릭 후 **Connect**를 선택합니다. 연결에 성공하면 Database Painter에 테이블 목록 등이 표시됩니다.

---

## 작업 공간(Workspace)과 대상(Target) 생성

PowerBuilder에서 프로젝트를 관리하는 기본 단위는 **Workspace**, **Target**, **Library(PBL)** 입니다.  
이들의 관계는 다음과 같습니다.

- **Workspace (.pbw)**: 가장 큰 작업 공간입니다. Visual Studio의 Solution과 유사하며, 여러 Target을 포함할 수 있습니다.
- **Target (.pbt)**: 개별 애플리케이션 프로젝트입니다. 컴파일 설정과 라이브러리 목록을 관리합니다.
- **Library (.pbl)**: PowerBuilder의 핵심 단위로, 윈도우, 데이터윈도우, 메뉴 등 모든 객체의 **바이너리 파일**이 저장됩니다. 단순 폴더가 아니라 객체가 컴파일되어 압축된 파일이므로, 버전 관리 시 주의해야 합니다.

> **추천 스크린샷 위치**: 세 가지 구조를 시각적으로 보여주는 시스템 트리 화면이나, 새 Workspace/Target 생성 대화상자의 이미지를 첨부하면 이해도가 크게 향상됩니다.

### 1. 새 Workspace 만들기

- 메뉴: **File → New**
- 대화상자에서 **Workspace** 탭을 선택하고 **Workspace** 아이콘을 더블클릭합니다.
- 저장 위치와 이름(예: `MyFirstWorkspace.pbw`)을 지정합니다.

### 2. 새 Target(애플리케이션) 생성

Workspace가 열린 상태에서 다시 **File → New**를 선택합니다.
- **Target** 탭에서 **Application** 아이콘을 더블클릭합니다.
- 애플리케이션 이름과 저장할 PBL 파일 경로를 지정합니다.
- 마법사가 추가 설정을 요구할 수 있으나, 기본값으로 진행합니다.

완료되면 Workspace에 애플리케이션 대상이 추가되고, 해당 PBL 파일이 생성됩니다.

---

## 애플리케이션 객체(Application Object) 구성

각 Target에는 하나의 **Application Object**가 존재하며, 애플리케이션의 시작점과 전역 설정을 담당합니다.

### 1. Application Object 열기

시스템 트리(System Tree)에서 Target 아래의 **Application**을 더블클릭하면 Application Painter가 열립니다.

### 2. 기본 속성 설정

- **AppLibrary**: 애플리케이션이 사용할 라이브러리 목록(기본 생성된 PBL 외에 추가 가능)
- **Product Name**, **Company Name** 등 정보 입력
- **Default Fonts**: 윈도우, 데이터윈도우, 대화상자 등에 사용할 기본 폰트 지정

### 3. 전역 변수와 이벤트 핸들러

Application Object에는 `open`, `close`, `idle`, `systemerror` 등의 이벤트가 있습니다. 가장 중요한 `open` 이벤트에 애플리케이션 시작 시 실행할 코드(예: 메인 윈도우 열기)를 작성합니다.

> **참고**: PowerBuilder는 애플리케이션 실행 시 데이터베이스 통신을 담당하는 기본 전역 객체인 **SQLCA(SQL Communications Area)** 를 자동으로 메모리에 생성합니다. 우리는 이 객체에 접속 정보를 세팅하기만 하면 됩니다.

```
// Application의 open 이벤트 예시 (ODBC)
SQLCA.DBMS = "ODBC"
SQLCA.DBParm = "ConnectString='DSN=MyLocalDB;UID=sa;PWD=1234'"
CONNECT USING SQLCA;
IF SQLCA.SQLCode <> 0 THEN
   MessageBox("오류", "데이터베이스 연결 실패")
   RETURN
END IF

Open(w_main)   // 메인 윈도우 열기
```

**실무 팁**: 네이티브 드라이버를 사용한다면 다음과 같이 연결 문자열이 달라집니다.

```
// MS SQL Server 네이티브 드라이버 예시
// SQLCA.DBMS = "SNC"
// SQLCA.DBParm = "Database='MyLocalDB',ServerName='192.168.0.10'"
```

> 주의 사항: 위 예제에서 `Open(w_main)` 코드는 아직 생성하지 않은 윈도우를 참조하고 있습니다.  
> **팁**: 컴파일 에러(`Undefined variable: w_main`)를 방지하려면, 다음 단계에서 설명할 빈 윈도우 `w_main`를 먼저 생성하여 저장한 뒤, Application Object의 `open` 이벤트에 `Open(w_main)` 코드를 작성하는 것이 좋습니다.

---

## 첫 번째 윈도우 생성 및 실행

### 1. 새 윈도우 생성

- **File → New** → **PB Object** 탭에서 **Window** 선택
- Window Painter가 열리면 빈 윈도우에 컨트롤(버튼, 텍스트 등)을 배치할 수 있습니다.
- 간단히 버튼 하나를 추가하고, 버튼의 `clicked` 이벤트에 코드를 작성합니다.
```

// 버튼 clicked 이벤트
MessageBox("환영", "PowerBuilder 첫 실행!")
```

### 2. 윈도우 저장
윈도우 이름을 `w_main`으로 지정하고 저장합니다.

### 3. 애플리케이션 실행
도구 모음의 **Run** 버튼(또는 F5)을 클릭합니다. 컴파일 과정을 거친 후, 앞서 Application Object의 `open` 이벤트에서 연결한 데이터베이스 연결과 함께 `w_main` 윈도우가 열립니다.

---

## 기본 개발 환경 설정 팁

### 1. 편집기 설정
- **Tools → Options**에서 **Editor** 탭을 선택하여 탭 크기, 글꼴, 색상 등을 조정할 수 있습니다.
- **Auto Indent**, **Syntax Coloring** 활성화를 권장합니다.

### 2. 시스템 트리 옵션
시스템 트리 상단의 필터 아이콘을 사용하여 특정 유형의 오브젝트만 표시할 수 있습니다. 검색 기능도 유용합니다.

### 3. 라이브러리 검색 경로
Application Object 속성에서 **Library Search Path**에 추가 PBL 파일을 세미콜론(;)으로 구분하여 등록하면, 여러 라이브러리에 분산된 오브젝트를 사용할 수 있습니다.

### 4. 디버깅 준비
**Debug** 메뉴에서 중단점 설정, 변수 감시 등을 사용하려면, 컴파일 시 디버그 정보를 포함하도록 설정해야 합니다. Target 속성(마우스 오른쪽 클릭 → Properties)에서 **Debug** 탭의 **Generate Debug Information**을 체크합니다.

---

## 마치며

이상으로 PowerBuilder 설치부터 간단한 애플리케이션 실행까지의 기본 설정을 알아보았습니다. 처음에는 낯선 용어와 인터페이스일 수 있지만, DataWindow의 강력함과 직관적인 Painter 도구에 익숙해지면 빠른 개발이 가능합니다. 다음 글에서는 본격적인 DataWindow 생성과 데이터 처리 방법을 다루어 보겠습니다.

---

이 글에서 설명한 설치 및 설정 과정은 PowerBuilder 2022 기준이며, 버전에 따라 일부 화면이나 옵션이 다를 수 있습니다.  
**Appeon 공식 문서**(https://docs.appeon.com)나 **PowerBuilder 커뮤니티 포럼**을 참고하면 더 자세한 정보를 얻을 수 있습니다.