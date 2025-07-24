---
layout: post
title: Avalonia - 환경 기반 설정 (dev, prod 분리)
date: 2025-03-04 20:20:23 +0900
category: Avalonia
---
# ⚙️ 6. 환경 기반 설정 (dev/prod 분리)  
## appsettings.dev.json, appsettings.prod.json 로딩 방법  
Avalonia에서도 환경에 따라 설정을 분리하는 것은 유지보수성과 배포 안정성 측면에서 매우 중요합니다.  
.NET Core의 설정 시스템을 활용하면 `appsettings.json`을 기반으로 환경별 설정 파일을 손쉽게 분리할 수 있습니다.

예를 들어 다음과 같은 구조로 설정 파일을 관리할 수 있습니다:

```
📁 Project Root
├── appsettings.json
├── appsettings.dev.json
└── appsettings.prod.json
```

### 🔧 기본 설정 로딩 코드 예제
`Program.cs` 또는 DI를 구성하는 초기 코드에 다음과 같이 설정을 적용합니다:

```csharp
using Microsoft.Extensions.Configuration;

var environment = Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT") ?? "prod";

var configuration = new ConfigurationBuilder()
    .SetBasePath(AppContext.BaseDirectory)
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddJsonFile($"appsettings.{environment}.json", optional: true, reloadOnChange: true)
    .Build();
```

이렇게 하면 `DOTNET_ENVIRONMENT` 값에 따라 다음이 자동으로 로딩됩니다:
- `dev` → `appsettings.dev.json`
- `prod` → `appsettings.prod.json`

### 🌐 API 주소 분기 예제

```json
// appsettings.dev.json
{
  "Api": {
    "BaseUrl": "https://api-dev.example.com"
  }
}

// appsettings.prod.json
{
  "Api": {
    "BaseUrl": "https://api.example.com"
  }
}
```

C#에서의 사용:

```csharp
string baseUrl = configuration["Api:BaseUrl"];
```

### 📜 로깅 레벨 분리 예제

```json
// appsettings.dev.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug"
    }
  }
}

// appsettings.prod.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  }
}
```

개발 시에는 디버깅에 유리하게 `Debug` 수준으로 로그를 많이 출력하고,  
배포 환경에서는 `Warning` 이상만 출력하여 성능 저하와 로그 오염을 방지합니다.

---

## 🎨 [UX 향상] 사용자 경험 개선 포인트

환경 분리는 단순히 설정 관리를 넘어서 사용자 경험(UX) 개선에도 직접적으로 영향을 미칩니다.

### ✅ 더 빠른 문제 해결
개발 환경에서 디버그 로그와 상세 오류 메시지를 출력하면 QA 및 개발 중 문제를 빠르게 파악할 수 있습니다. 이는 궁극적으로 사용자에게 더 빠르고 안정적인 피드백을 제공합니다.

### 🌍 API 주소 분기를 통한 자동 전환
빌드나 실행 환경에 따라 자동으로 적절한 API 주소를 로딩하므로, 사용자가 잘못된 서버에 연결되는 실수를 방지할 수 있습니다.

### 📉 성능 최적화
프로덕션 환경에서는 디버그 관련 리소스를 줄이고, 로깅도 최소화하여 앱의 초기 로딩 시간 및 전체 반응 속도를 개선할 수 있습니다.

### 🧪 A/B 테스트 또는 실험 기능 플래그
환경 설정을 기반으로 실험 기능을 `dev`에만 노출하거나, 특정 사용자에게만 활성화하여 사용자 피드백을 유연하게 수집할 수 있습니다.

---

## ✅ 마무리

환경 기반 설정 분리는 **코드의 품질 유지**, **안정적인 배포**, **빠른 피드백 루프**, 그리고 무엇보다도 **사용자 경험 향상**에 중요한 역할을 합니다.  
Avalonia에서도 이러한 모던한 .NET 설정 시스템을 잘 활용하면 데스크톱 앱 개발에서도 생산성과 품질을 모두 챙길 수 있습니다.