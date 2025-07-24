---
layout: post
title: Avalonia - 상태머신 기반 UI 흐름
date: 2025-02-17 21:20:23 +0900
category: Avalonia
---
# 🧠 Avalonia에서 상태머신 기반 UI 흐름 구현하기

---

## 🎯 핵심 목표

| 항목 | 설명 |
|------|------|
| 명확한 상태 정의 | 예: `Step1 → Step2 → Step3` |
| 상태 전이 제어 | `Next()`, `Previous()` 등 전이 제어 함수 |
| 각 상태마다 ViewModel 및 View 할당 | UI도 상태에 따라 바뀜 |
| 유효성 검증 및 흐름 제어 | 유효하지 않으면 전이 불가

---

## 🧱 구조 예시 (회원가입 예제)

```
MyApp/
├── Views/
│   ├── Step1View.axaml
│   ├── Step2View.axaml
│   └── Step3View.axaml
├── ViewModels/
│   ├── Step1ViewModel.cs
│   ├── Step2ViewModel.cs
│   ├── Step3ViewModel.cs
│   └── SignupFlowViewModel.cs
├── Services/
│   └── IStateManager.cs
```

---

## 1️⃣ 상태 정의

### 📄 SignupStep.cs

```csharp
public enum SignupStep
{
    Step1_Email,
    Step2_Password,
    Step3_Complete
}
```

---

## 2️⃣ IStateManager 인터페이스

```csharp
public interface IStateManager<TState>
{
    TState Current { get; }
    bool CanMoveNext { get; }
    bool CanMoveBack { get; }

    void MoveNext();
    void MoveBack();
}
```

---

## 3️⃣ 구현체 작성: SignupFlowStateManager

```csharp
public class SignupFlowStateManager : IStateManager<SignupStep>
{
    private readonly List<SignupStep> _steps = new()
    {
        SignupStep.Step1_Email,
        SignupStep.Step2_Password,
        SignupStep.Step3_Complete
    };

    private int _currentIndex = 0;

    public SignupStep Current => _steps[_currentIndex];
    public bool CanMoveNext => _currentIndex < _steps.Count - 1;
    public bool CanMoveBack => _currentIndex > 0;

    public void MoveNext()
    {
        if (CanMoveNext) _currentIndex++;
    }

    public void MoveBack()
    {
        if (CanMoveBack) _currentIndex--;
    }
}
```

---

## 4️⃣ SignupFlowViewModel 구현

```csharp
public class SignupFlowViewModel : ReactiveObject
{
    private readonly SignupFlowStateManager _stateManager;

    public ReactiveCommand<Unit, Unit> NextCommand { get; }
    public ReactiveCommand<Unit, Unit> BackCommand { get; }

    public object? CurrentViewModel { get; private set; }

    public SignupFlowViewModel()
    {
        _stateManager = new SignupFlowStateManager();

        NextCommand = ReactiveCommand.Create(Next);
        BackCommand = ReactiveCommand.Create(Back);

        UpdateCurrentView();
    }

    private void Next()
    {
        _stateManager.MoveNext();
        UpdateCurrentView();
    }

    private void Back()
    {
        _stateManager.MoveBack();
        UpdateCurrentView();
    }

    private void UpdateCurrentView()
    {
        CurrentViewModel = _stateManager.Current switch
        {
            SignupStep.Step1_Email => new Step1ViewModel(),
            SignupStep.Step2_Password => new Step2ViewModel(),
            SignupStep.Step3_Complete => new Step3ViewModel(),
            _ => null
        };

        this.RaisePropertyChanged(nameof(CurrentViewModel));
    }
}
```

---

## 5️⃣ ShellView와 View 연결

### 📄 SignupFlowView.axaml

```xml
<Grid>
  <ContentControl Content="{Binding CurrentViewModel}" />
  <StackPanel Orientation="Horizontal" HorizontalAlignment="Center" Margin="0,20">
    <Button Content="⬅ 뒤로" Command="{Binding BackCommand}" IsEnabled="{Binding Path=_stateManager.CanMoveBack}" />
    <Button Content="➡ 다음" Command="{Binding NextCommand}" IsEnabled="{Binding Path=_stateManager.CanMoveNext}" />
  </StackPanel>
</Grid>
```

> 또는 버튼 제어를 `SignupFlowViewModel`에서 직접 바인딩으로 노출해도 됨.

---

## 6️⃣ 각 단계 ViewModel 예시

### 📄 Step1ViewModel.cs

```csharp
public class Step1ViewModel : ReactiveObject
{
    public string Email { get; set; } = "";

    public bool IsValid => !string.IsNullOrWhiteSpace(Email) && Email.Contains("@");
}
```

다음 단계로 진행 시 유효성 검증은 `SignupFlowViewModel.Next()`에서 `Step1ViewModel.IsValid`를 검사하는 방식으로 처리할 수 있음.

---

## ✅ 상태머신 확장 아이디어

| 기능 | 설명 |
|------|------|
| ⛔ 유효성 검증에 따라 Next 막기 | 각 StepViewModel에 `IsValid` |
| 🔁 완료 후 다시 처음으로 | Step3에서 완료 후 `Reset()` 호출 |
| 🧠 복잡한 상태 전이 지원 | 상태마다 허용되는 다음 상태 지정 |
| 🗂 상태별 상태 데이터 공유 | Email, Password 등을 Flow ViewModel에서 중앙 관리

---

## 🔄 상태 전이 정의 방식 예시 (전이 매트릭스)

```csharp
private readonly Dictionary<SignupStep, SignupStep[]> _transitions = new()
{
    { SignupStep.Step1_Email, new[] { SignupStep.Step2_Password } },
    { SignupStep.Step2_Password, new[] { SignupStep.Step3_Complete } },
    { SignupStep.Step3_Complete, Array.Empty<SignupStep>() }
};
```

---

## 📦 실제 배포 시 고려사항

| 항목 | 내용 |
|------|------|
| 👉 상태에 따른 진입 제한 | 특정 조건 만족 시만 해당 상태 진입 허용 |
| ✅ 유효성 및 서버 연동 포함 | 이메일 중복 검사 등 |
| 🧪 단위 테스트 | `IStateManager` 단위로 테스트 가능 |
| 🔒 민감 정보 처리 | 패스워드 암호화 등 보안 고려

---

## 📚 참고 개념

- `State Pattern` (GoF 디자인 패턴)
- `Finite State Machine (FSM)` 구조
- Wizard/Stepper UI 구성 방식
- Avalonia의 `ContentControl`, `DataTemplate` 조합

---

## ✍️ 결론

- `enum` 기반 상태 정의로 명확한 흐름 구성
- `IStateManager`로 상태 전이 로직 분리 → 테스트 가능
- `SignupFlowViewModel`이 흐름 제어자 역할
- `ContentControl`로 각 단계 화면 렌더링

---

## 🧭 다음 주제로 추천하는 확장

| 주제 | 설명 |
|------|------|
| 🧾 다단계 Form의 유효성 검증 및 진행 제어 | Step별 검증 실패 시 진행 불가 처리 |
| 🔄 상태 변화 로깅 | Step 이동 히스토리 기록 |
| 🧪 테스트 가능한 상태머신 작성법 | 전이 조건 단위 테스트 |
| 🧩 플러그인별 상태 연결 (모듈형 흐름) | 플러그인이 상태를 등록함

필요하신 항목이 있다면 이어서 정리해드릴게요!