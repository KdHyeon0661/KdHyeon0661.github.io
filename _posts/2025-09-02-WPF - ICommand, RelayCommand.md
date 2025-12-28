---
layout: post
title: WPF - ICommand, RelayCommand
date: 2025-09-02 15:25:23 +0900
category: WPF
---
# WPF ICommand와 RelayCommand 구현 완전 가이드: 실전에서 바로 활용하는 패턴

WPF의 Command 시스템은 단순히 버튼 클릭을 처리하는 수준을 넘어, UI 상호작용을 체계적으로 구조화하는 핵심 메커니즘입니다. 이 시스템의 중심에 있는 `ICommand` 인터페이스와 그 대표적 구현체인 `RelayCommand`는 MVVM 아키텍처에서 뷰와 뷰모델 간의 깔끔한 분리를 가능하게 합니다. 이 글에서는 단순한 구현 설명을 넘어, 실제 프로젝트에서 마주하게 될 다양한 시나리오와 그에 대한 최적의 해법을 단계별로 살펴보겠습니다.

## ICommand 인터페이스의 철학적 이해

`ICommand`는 사용자의 의도를 캡슐화하는 객체 지향 패턴의 구현체입니다. 단순한 메서드 호출과의 근본적 차이는 상태 기반의 실행 제어와 UI 자동 동기화에 있습니다.

```csharp
// ICommand 인터페이스는 세 가지 핵심 요소로 구성됩니다
public interface ICommand
{
    // Command의 실행 가능 상태가 변경되었음을 UI에 알리는 이벤트
    event EventHandler CanExecuteChanged;
    
    // 현재 상태에서 Command를 실행할 수 있는지 여부를 반환
    bool CanExecute(object parameter);
    
    // Command의 실제 동작을 수행
    void Execute(object parameter);
}
```

이 인터페이스의 아름다움은 `CanExecute`와 `CanExecuteChanged`의 조합에 있습니다. 버튼이 `ICommand` 구현체에 바인딩되면, `CanExecute`가 `false`를 반환할 때 자동으로 비활성화됩니다. 이는 단순한 편의 기능이 아니라, UI 상태 관리를 선언적으로 처리할 수 있는 패러다임 전환을 의미합니다.

## 실용적인 RelayCommand 구현 패턴

### 기본 동기 Command 구현

실제 프로젝트에서 사용할 수 있는 완전한 기능의 `RelayCommand` 구현은 다음과 같은 요소를 고려해야 합니다:

```csharp
using System;
using System.Diagnostics;
using System.Windows;
using System.Windows.Input;

/// <summary>
/// 명령 실행을 Action 델리게이트로 위임하는 기본적인 ICommand 구현체
/// </summary>
public sealed class RelayCommand : ICommand
{
    private readonly Action<object> _execute;
    private readonly Func<object, bool> _canExecute;
    private readonly string _commandName;
    
    /// <summary>
    /// 파라미터가 없는 Execute와 CanExecute를 사용하는 생성자
    /// </summary>
    public RelayCommand(Action execute, Func<bool> canExecute = null, string commandName = null)
        : this(
            execute != null ? (Action<object>)(_ => execute()) : null,
            canExecute != null ? (Func<object, bool>)(_ => canExecute()) : null,
            commandName
        )
    {
    }
    
    /// <summary>
    /// 파라미터를 받는 Execute와 CanExecute를 사용하는 생성자
    /// </summary>
    public RelayCommand(Action<object> execute, Func<object, bool> canExecute = null, string commandName = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
        _commandName = commandName ?? GetDefaultCommandName();
        
        // CommandManager의 RequerySuggested 이벤트를 약한 참조로 구독
        // 이렇게 하면 UI 포커스 변경 등 시스템 이벤트에 자동으로 반응
        WeakEventManager<CommandManager, EventArgs>.AddHandler(
            null, 
            nameof(CommandManager.RequerySuggested), 
            OnRequerySuggested
        );
    }
    
    /// <summary>
    /// 현재 Command를 실행할 수 있는지 여부를 확인
    /// </summary>
    public bool CanExecute(object parameter)
    {
        try
        {
            return _canExecute?.Invoke(parameter) ?? true;
        }
        catch (Exception ex)
        {
            // CanExecute에서 예외가 발생하면 실행 불가로 처리
            Debug.WriteLine($"{_commandName}.CanExecute 예외: {ex.Message}");
            return false;
        }
    }
    
    /// <summary>
    /// Command를 실행
    /// </summary>
    public void Execute(object parameter)
    {
        if (!CanExecute(parameter))
        {
            Debug.WriteLine($"{_commandName} 실행 거부: CanExecute가 false를 반환함");
            return;
        }
        
        try
        {
            Debug.WriteLine($"{_commandName} 실행 시작");
            _execute(parameter);
            Debug.WriteLine($"{_commandName} 실행 완료");
        }
        catch (Exception ex)
        {
            Debug.WriteLine($"{_commandName} 실행 중 예외: {ex}");
            
            // 실제 애플리케이션에서는 여기에 예외 처리 로직을 추가
            // 예: 사용자에게 알림, 로깅 등
            throw new CommandExecutionException(_commandName, ex);
        }
    }
    
    /// <summary>
    /// Command의 실행 가능 상태가 변경되었음을 UI에 알림
    /// UI 스레드에서 안전하게 호출됨
    /// </summary>
    public void RaiseCanExecuteChanged()
    {
        if (Application.Current?.Dispatcher?.CheckAccess() == true)
        {
            CanExecuteChanged?.Invoke(this, EventArgs.Empty);
        }
        else
        {
            Application.Current?.Dispatcher?.Invoke(() => 
                CanExecuteChanged?.Invoke(this, EventArgs.Empty)
            );
        }
    }
    
    /// <summary>
    /// Command의 실행 가능 상태 변경 이벤트
    /// </summary>
    public event EventHandler CanExecuteChanged;
    
    /// <summary>
    /// 시스템에서 Command 재평가를 요청할 때 호출되는 핸들러
    /// </summary>
    private void OnRequerySuggested(object sender, EventArgs e)
    {
        RaiseCanExecuteChanged();
    }
    
    /// <summary>
    /// Command의 기본 이름 생성 (디버깅 용도)
    /// </summary>
    private string GetDefaultCommandName()
    {
        return $"RelayCommand_{Guid.NewGuid().ToString("N").Substring(0, 8)}";
    }
    
    public override string ToString() => _commandName;
}

/// <summary>
/// Command 실행 중 발생한 예외를 캡슐화하는 클래스
/// </summary>
public class CommandExecutionException : Exception
{
    public string CommandName { get; }
    
    public CommandExecutionException(string commandName, Exception innerException)
        : base($"'{commandName}' 명령 실행 중 오류 발생", innerException)
    {
        CommandName = commandName;
    }
}
```

### 강력한 형식의 제네릭 Command

타입 안정성을 확보하기 위한 제네릭 버전의 구현은 다음과 같습니다:

```csharp
using System;
using System.Windows;
using System.Windows.Input;

/// <summary>
/// 강력한 형식의 파라미터를 받는 Command 구현체
/// </summary>
/// <typeparam name="T">Command 파라미터의 타입</typeparam>
public sealed class RelayCommand<T> : ICommand
{
    private readonly Action<T> _execute;
    private readonly Predicate<T> _canExecute;
    private readonly IValueConverter _parameterConverter;
    
    /// <summary>
    /// 강력한 형식의 Command 생성자
    /// </summary>
    public RelayCommand(
        Action<T> execute, 
        Predicate<T> canExecute = null,
        IValueConverter parameterConverter = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
        _parameterConverter = parameterConverter;
        
        WeakEventManager<CommandManager, EventArgs>.AddHandler(
            null,
            nameof(CommandManager.RequerySuggested),
            OnRequerySuggested
        );
    }
    
    /// <summary>
    /// 약한 형식의 CanExecute 구현
    /// </summary>
    public bool CanExecute(object parameter)
    {
        if (_canExecute == null)
            return true;
        
        try
        {
            // 파라미터 변환 시도
            T typedParameter = ConvertParameter(parameter);
            return _canExecute(typedParameter);
        }
        catch
        {
            // 변환 실패 시 실행 불가로 처리
            return false;
        }
    }
    
    /// <summary>
    /// 약한 형식의 Execute 구현
    /// </summary>
    public void Execute(object parameter)
    {
        try
        {
            T typedParameter = ConvertParameter(parameter);
            
            if (CanExecute(parameter))
            {
                _execute(typedParameter);
            }
        }
        catch (Exception ex)
        {
            // 파라미터 변환 실패 등 예외 처리
            throw new InvalidOperationException(
                $"Command 파라미터 변환 실패: {ex.Message}", 
                ex
            );
        }
    }
    
    /// <summary>
    /// object 파라미터를 T 타입으로 안전하게 변환
    /// </summary>
    private T ConvertParameter(object parameter)
    {
        // 1. 이미 올바른 타입인 경우
        if (parameter is T typedParam)
            return typedParam;
        
        // 2. null이고 T가 nullable인 경우
        if (parameter == null)
        {
            // 참조 타입이나 Nullable<T>인 경우 null 반환
            Type type = typeof(T);
            if (!type.IsValueType || Nullable.GetUnderlyingType(type) != null)
                return default;
            
            // 값 타입인 경우 예외
            throw new InvalidCastException(
                $"null을 {typeof(T).Name} 타입으로 변환할 수 없습니다."
            );
        }
        
        // 3. 변환기(Converter)가 있는 경우 사용
        if (_parameterConverter != null)
        {
            object converted = _parameterConverter.Convert(
                parameter, 
                typeof(T), 
                null, 
                System.Globalization.CultureInfo.CurrentCulture
            );
            
            if (converted is T convertedTyped)
                return convertedTyped;
        }
        
        // 4. 직접 변환 시도
        try
        {
            return (T)System.Convert.ChangeType(
                parameter, 
                typeof(T), 
                System.Globalization.CultureInfo.CurrentCulture
            );
        }
        catch (InvalidCastException)
        {
            throw new InvalidCastException(
                $"'{parameter}'({parameter.GetType().Name})을 " +
                $"{typeof(T).Name} 타입으로 변환할 수 없습니다."
            );
        }
    }
    
    /// <summary>
    /// Command 상태 변경 이벤트
    /// </summary>
    public event EventHandler CanExecuteChanged;
    
    /// <summary>
    /// 실행 가능 상태 변경 알림
    /// </summary>
    public void RaiseCanExecuteChanged()
    {
        if (Application.Current?.Dispatcher?.CheckAccess() == true)
        {
            CanExecuteChanged?.Invoke(this, EventArgs.Empty);
        }
        else
        {
            Application.Current?.Dispatcher?.Invoke(() =>
                CanExecuteChanged?.Invoke(this, EventArgs.Empty)
            );
        }
    }
    
    /// <summary>
    /// 시스템 재평가 요청 핸들러
    /// </summary>
    private void OnRequerySuggested(object sender, EventArgs e)
    {
        RaiseCanExecuteChanged();
    }
}
```

## 비동기 Command의 안전한 구현

비동기 작업을 처리하는 Command는 여러 가지 추가적인 고려사항이 필요합니다. 동시 실행 방지, 취소 지원, 예외 처리 등의 기능을 포함한 완전한 구현은 다음과 같습니다:

```csharp
using System;
using System.Diagnostics;
using System.Threading;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Input;

/// <summary>
/// 비동기 작업을 안전하게 처리하는 Command 구현체
/// </summary>
public sealed class AsyncRelayCommand : ICommand
{
    private readonly Func<CancellationToken, Task> _asyncExecute;
    private readonly Func<bool> _canExecute;
    private readonly Action<Exception> _errorHandler;
    private readonly Action _completedHandler;
    
    private CancellationTokenSource _cancellationTokenSource;
    private Task _executingTask;
    private bool _isExecuting;
    
    /// <summary>
    /// 비동기 Command의 현재 실행 상태
    /// </summary>
    public bool IsExecuting
    {
        get => _isExecuting;
        private set
        {
            if (_isExecuting != value)
            {
                _isExecuting = value;
                RaiseCanExecuteChanged();
            }
        }
    }
    
    /// <summary>
    /// 현재 실행 중인 작업
    /// </summary>
    public Task ExecutingTask => _executingTask;
    
    /// <summary>
    /// 비동기 Command 생성자
    /// </summary>
    public AsyncRelayCommand(
        Func<CancellationToken, Task> asyncExecute,
        Func<bool> canExecute = null,
        Action<Exception> errorHandler = null,
        Action completedHandler = null)
    {
        _asyncExecute = asyncExecute ?? throw new ArgumentNullException(nameof(asyncExecute));
        _canExecute = canExecute ?? (() => true);
        _errorHandler = errorHandler;
        _completedHandler = completedHandler;
        
        WeakEventManager<CommandManager, EventArgs>.AddHandler(
            null,
            nameof(CommandManager.RequerySuggested),
            OnRequerySuggested
        );
    }
    
    /// <summary>
    /// Command 실행 가능 여부 확인
    /// </summary>
    public bool CanExecute(object parameter)
    {
        return !IsExecuting && _canExecute();
    }
    
    /// <summary>
    /// Command 실행
    /// </summary>
    public async void Execute(object parameter)
    {
        if (!CanExecute(parameter))
            return;
        
        IsExecuting = true;
        _cancellationTokenSource = new CancellationTokenSource();
        
        try
        {
            _executingTask = _asyncExecute(_cancellationTokenSource.Token);
            await _executingTask;
            
            _completedHandler?.Invoke();
        }
        catch (OperationCanceledException)
        {
            // 작업 취소는 정상적인 흐름으로 처리
            Debug.WriteLine("비동기 Command 작업이 취소되었습니다.");
        }
        catch (Exception ex)
        {
            Debug.WriteLine($"비동기 Command 실행 중 예외: {ex}");
            _errorHandler?.Invoke(ex);
        }
        finally
        {
            _cancellationTokenSource.Dispose();
            _cancellationTokenSource = null;
            _executingTask = null;
            IsExecuting = false;
        }
    }
    
    /// <summary>
    /// 실행 중인 비동기 작업 취소
    /// </summary>
    public void Cancel()
    {
        _cancellationTokenSource?.Cancel();
    }
    
    /// <summary>
    /// 상태 변경 이벤트
    /// </summary>
    public event EventHandler CanExecuteChanged;
    
    /// <summary>
    /// 실행 가능 상태 변경 알림
    /// </summary>
    public void RaiseCanExecuteChanged()
    {
        if (Application.Current?.Dispatcher?.CheckAccess() == true)
        {
            CanExecuteChanged?.Invoke(this, EventArgs.Empty);
        }
        else
        {
            Application.Current?.Dispatcher?.Invoke(() =>
                CanExecuteChanged?.Invoke(this, EventArgs.Empty)
            );
        }
    }
    
    /// <summary>
    /// 시스템 재평가 요청 핸들러
    /// </summary>
    private void OnRequerySuggested(object sender, EventArgs e)
    {
        RaiseCanExecuteChanged();
    }
}

/// <summary>
/// 파라미터를 받는 비동기 Command 구현체
/// </summary>
public sealed class AsyncRelayCommand<T> : ICommand
{
    private readonly Func<T, CancellationToken, Task> _asyncExecute;
    private readonly Predicate<T> _canExecute;
    private readonly Action<Exception> _errorHandler;
    
    private CancellationTokenSource _cancellationTokenSource;
    private bool _isExecuting;
    
    public bool IsExecuting
    {
        get => _isExecuting;
        private set
        {
            if (_isExecuting != value)
            {
                _isExecuting = value;
                RaiseCanExecuteChanged();
            }
        }
    }
    
    public AsyncRelayCommand(
        Func<T, CancellationToken, Task> asyncExecute,
        Predicate<T> canExecute = null,
        Action<Exception> errorHandler = null)
    {
        _asyncExecute = asyncExecute ?? throw new ArgumentNullException(nameof(asyncExecute));
        _canExecute = canExecute;
        _errorHandler = errorHandler;
        
        WeakEventManager<CommandManager, EventArgs>.AddHandler(
            null,
            nameof(CommandManager.RequerySuggested),
            OnRequerySuggested
        );
    }
    
    public bool CanExecute(object parameter)
    {
        if (IsExecuting)
            return false;
        
        if (_canExecute == null)
            return true;
        
        try
        {
            T typedParameter = ConvertParameter(parameter);
            return _canExecute(typedParameter);
        }
        catch
        {
            return false;
        }
    }
    
    public async void Execute(object parameter)
    {
        if (!CanExecute(parameter))
            return;
        
        IsExecuting = true;
        _cancellationTokenSource = new CancellationTokenSource();
        
        try
        {
            T typedParameter = ConvertParameter(parameter);
            await _asyncExecute(typedParameter, _cancellationTokenSource.Token);
        }
        catch (OperationCanceledException)
        {
            // 작업 취소는 정상 처리
        }
        catch (Exception ex)
        {
            _errorHandler?.Invoke(ex);
        }
        finally
        {
            _cancellationTokenSource.Dispose();
            _cancellationTokenSource = null;
            IsExecuting = false;
        }
    }
    
    public void Cancel()
    {
        _cancellationTokenSource?.Cancel();
    }
    
    private T ConvertParameter(object parameter)
    {
        if (parameter is T typedParam)
            return typedParam;
        
        if (parameter == null)
        {
            Type type = typeof(T);
            if (!type.IsValueType || Nullable.GetUnderlyingType(type) != null)
                return default;
            
            throw new InvalidCastException($"null을 {typeof(T).Name} 타입으로 변환할 수 없습니다.");
        }
        
        try
        {
            return (T)System.Convert.ChangeType(
                parameter,
                typeof(T),
                System.Globalization.CultureInfo.CurrentCulture
            );
        }
        catch (InvalidCastException)
        {
            throw new InvalidCastException(
                $"'{parameter}'을 {typeof(T).Name} 타입으로 변환할 수 없습니다."
            );
        }
    }
    
    public event EventHandler CanExecuteChanged;
    
    public void RaiseCanExecuteChanged()
    {
        if (Application.Current?.Dispatcher?.CheckAccess() == true)
        {
            CanExecuteChanged?.Invoke(this, EventArgs.Empty);
        }
        else
        {
            Application.Current?.Dispatcher?.Invoke(() =>
                CanExecuteChanged?.Invoke(this, EventArgs.Empty)
            );
        }
    }
    
    private void OnRequerySuggested(object sender, EventArgs e)
    {
        RaiseCanExecuteChanged();
    }
}
```

## 실전 프로젝트 통합 사례

이제 위에서 구현한 Command들을 실제 비즈니스 시나리오에 적용해 보겠습니다. 데이터 관리 애플리케이션의 ViewModel을 예로 들어보죠.

### 완전한 기능의 ViewModel 구현

```csharp
using System;
using System.Collections.Generic;
using System.Collections.ObjectModel;
using System.ComponentModel;
using System.Linq;
using System.Runtime.CompilerServices;
using System.Threading;
using System.Threading.Tasks;
using System.Windows;

public class ProductManagementViewModel : INotifyPropertyChanged
{
    // 상태 속성들
    private string _productName = string.Empty;
    private decimal _productPrice;
    private Product _selectedProduct;
    private string _searchTerm = string.Empty;
    private bool _isLoading;
    private string _statusMessage = "준비됨";
    
    // 속성 정의
    public string ProductName
    {
        get => _productName;
        set
        {
            if (_productName != value)
            {
                _productName = value;
                OnPropertyChanged();
                
                // ProductName이 변경되면 관련 Command들의 상태 업데이트
                AddProductCommand.RaiseCanExecuteChanged();
                UpdateProductCommand.RaiseCanExecuteChanged();
            }
        }
    }
    
    public decimal ProductPrice
    {
        get => _productPrice;
        set
        {
            if (_productPrice != value)
            {
                _productPrice = value;
                OnPropertyChanged();
                
                AddProductCommand.RaiseCanExecuteChanged();
                UpdateProductCommand.RaiseCanExecuteChanged();
            }
        }
    }
    
    public Product SelectedProduct
    {
        get => _selectedProduct;
        set
        {
            if (_selectedProduct != value)
            {
                _selectedProduct = value;
                OnPropertyChanged();
                
                // 선택된 제품이 변경되면 관련 UI 업데이트
                if (_selectedProduct != null)
                {
                    ProductName = _selectedProduct.Name;
                    ProductPrice = _selectedProduct.Price;
                }
                
                // Command 상태 업데이트
                DeleteProductCommand.RaiseCanExecuteChanged();
                UpdateProductCommand.RaiseCanExecuteChanged();
            }
        }
    }
    
    public string SearchTerm
    {
        get => _searchTerm;
        set
        {
            if (_searchTerm != value)
            {
                _searchTerm = value;
                OnPropertyChanged();
                
                // 검색어가 변경되면 자동으로 필터링 적용
                FilterProducts();
            }
        }
    }
    
    public bool IsLoading
    {
        get => _isLoading;
        private set
        {
            if (_isLoading != value)
            {
                _isLoading = value;
                OnPropertyChanged();
                
                // 로딩 상태가 변경되면 모든 Command 재평가
                CommandManager.InvalidateRequerySuggested();
            }
        }
    }
    
    public string StatusMessage
    {
        get => _statusMessage;
        private set
        {
            if (_statusMessage != value)
            {
                _statusMessage = value;
                OnPropertyChanged();
            }
        }
    }
    
    // 데이터 컬렉션
    public ObservableCollection<Product> Products { get; } = new ObservableCollection<Product>();
    public ObservableCollection<Product> FilteredProducts { get; } = new ObservableCollection<Product>();
    
    // Command 정의
    public RelayCommand AddProductCommand { get; }
    public RelayCommand DeleteProductCommand { get; }
    public RelayCommand ClearFormCommand { get; }
    public AsyncRelayCommand LoadProductsCommand { get; }
    public RelayCommand<Product> SelectProductCommand { get; }
    public RelayCommand UpdateProductCommand { get; }
    public AsyncRelayCommand ExportProductsCommand { get; }
    
    // 생성자
    public ProductManagementViewModel()
    {
        // Command 초기화
        AddProductCommand = new RelayCommand(
            execute: AddProduct,
            canExecute: CanAddProduct,
            commandName: "제품 추가"
        );
        
        DeleteProductCommand = new RelayCommand(
            execute: DeleteProduct,
            canExecute: CanDeleteProduct,
            commandName: "제품 삭제"
        );
        
        ClearFormCommand = new RelayCommand(
            execute: ClearForm,
            commandName: "폼 초기화"
        );
        
        LoadProductsCommand = new AsyncRelayCommand(
            asyncExecute: LoadProductsAsync,
            canExecute: () => !IsLoading,
            errorHandler: OnLoadError,
            completedHandler: () => StatusMessage = "제품 목록 로드 완료"
        );
        
        SelectProductCommand = new RelayCommand<Product>(
            execute: SelectProduct,
            canExecute: product => product != null,
            commandName: "제품 선택"
        );
        
        UpdateProductCommand = new RelayCommand(
            execute: UpdateProduct,
            canExecute: CanUpdateProduct,
            commandName: "제품 수정"
        );
        
        ExportProductsCommand = new AsyncRelayCommand(
            asyncExecute: ExportProductsAsync,
            canExecute: () => !IsLoading && Products.Any(),
            errorHandler: OnExportError,
            commandName: "제품 내보내기"
        );
        
        // 초기 데이터 로드
        InitializeData();
    }
    
    // Command 실행 로직
    private bool CanAddProduct()
    {
        // 제품 이름이 비어있지 않고, 가격이 양수이며, 로딩 중이 아닐 때만 추가 가능
        return !string.IsNullOrWhiteSpace(ProductName) &&
               ProductPrice > 0 &&
               !IsLoading;
    }
    
    private void AddProduct()
    {
        var newProduct = new Product
        {
            Id = Products.Count + 1,
            Name = ProductName,
            Price = ProductPrice,
            CreatedDate = DateTime.Now
        };
        
        Products.Add(newProduct);
        FilterProducts(); // 필터링된 목록도 업데이트
        
        StatusMessage = $"'{ProductName}' 제품이 추가되었습니다.";
        ClearForm();
    }
    
    private bool CanDeleteProduct()
    {
        return SelectedProduct != null && !IsLoading;
    }
    
    private void DeleteProduct()
    {
        if (SelectedProduct == null)
            return;
        
        var productName = SelectedProduct.Name;
        
        // 확인 대화상자 (실제 구현에서는 MessageBox 사용)
        Debug.WriteLine($"'{productName}' 제품을 삭제하시겠습니까?");
        
        Products.Remove(SelectedProduct);
        FilterProducts();
        
        StatusMessage = $"'{productName}' 제품이 삭제되었습니다.";
        ClearForm();
    }
    
    private void ClearForm()
    {
        ProductName = string.Empty;
        ProductPrice = 0;
        SelectedProduct = null;
    }
    
    private bool CanUpdateProduct()
    {
        return SelectedProduct != null &&
               !string.IsNullOrWhiteSpace(ProductName) &&
               ProductPrice > 0 &&
               !IsLoading;
    }
    
    private void UpdateProduct()
    {
        if (SelectedProduct == null)
            return;
        
        SelectedProduct.Name = ProductName;
        SelectedProduct.Price = ProductPrice;
        
        // ObservableCollection 업데이트를 알리기 위해 임시 조치
        var index = Products.IndexOf(SelectedProduct);
        Products.RemoveAt(index);
        Products.Insert(index, SelectedProduct);
        
        FilterProducts();
        
        StatusMessage = $"제품 정보가 업데이트되었습니다.";
    }
    
    private void SelectProduct(Product product)
    {
        SelectedProduct = product;
    }
    
    private async Task LoadProductsAsync(CancellationToken cancellationToken)
    {
        IsLoading = true;
        StatusMessage = "제품 목록 로드 중...";
        
        try
        {
            await Task.Delay(1000, cancellationToken); // 시뮬레이션
            
            // 실제 구현에서는 데이터베이스나 API에서 데이터 로드
            Products.Clear();
            
            // 샘플 데이터 추가
            var sampleProducts = new List<Product>
            {
                new Product { Id = 1, Name = "노트북", Price = 1200000, CreatedDate = DateTime.Now.AddDays(-10) },
                new Product { Id = 2, Name = "스마트폰", Price = 800000, CreatedDate = DateTime.Now.AddDays(-5) },
                new Product { Id = 3, Name = "태블릿", Price = 600000, CreatedDate = DateTime.Now.AddDays(-2) },
                new Product { Id = 4, Name = "키보드", Price = 120000, CreatedDate = DateTime.Now },
                new Product { Id = 5, Name = "마우스", Price = 50000, CreatedDate = DateTime.Now }
            };
            
            foreach (var product in sampleProducts)
            {
                Products.Add(product);
            }
            
            FilterProducts();
        }
        finally
        {
            IsLoading = false;
        }
    }
    
    private async Task ExportProductsAsync(CancellationToken cancellationToken)
    {
        IsLoading = true;
        StatusMessage = "제품 데이터 내보내는 중...";
        
        try
        {
            // 실제 구현에서는 Excel, CSV 등으로 내보내기
            await Task.Delay(1500, cancellationToken);
            
            // 내보내기 로직 시뮬레이션
            var exportData = string.Join("\n", 
                Products.Select(p => $"{p.Id},{p.Name},{p.Price},{p.CreatedDate:yyyy-MM-dd}"));
            
            Debug.WriteLine($"내보내기 데이터:\n{exportData}");
            
            StatusMessage = $"{Products.Count}개의 제품 데이터가 내보내기 되었습니다.";
        }
        finally
        {
            IsLoading = false;
        }
    }
    
    // Helper 메서드들
    private void FilterProducts()
    {
        FilteredProducts.Clear();
        
        var filtered = string.IsNullOrWhiteSpace(SearchTerm)
            ? Products
            : Products.Where(p => p.Name.Contains(SearchTerm, StringComparison.OrdinalIgnoreCase));
        
        foreach (var product in filtered)
        {
            FilteredProducts.Add(product);
        }
    }
    
    private void InitializeData()
    {
        // 초기 샘플 데이터
        Products.Add(new Product { Id = 1, Name = "컴퓨터", Price = 1500000 });
        Products.Add(new Product { Id = 2, Name = "모니터", Price = 300000 });
        Products.Add(new Product { Id = 3, Name = "프린터", Price = 250000 });
        
        FilterProducts();
    }
    
    private void OnLoadError(Exception ex)
    {
        StatusMessage = $"제품 목록 로드 실패: {ex.Message}";
    }
    
    private void OnExportError(Exception ex)
    {
        StatusMessage = $"제품 내보내기 실패: {ex.Message}";
    }
    
    // INotifyPropertyChanged 구현
    public event PropertyChangedEventHandler PropertyChanged;
    
    protected virtual void OnPropertyChanged([CallerMemberName] string propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}

// Product 모델 클래스
public class Product : INotifyPropertyChanged
{
    private int _id;
    private string _name = string.Empty;
    private decimal _price;
    private DateTime _createdDate;
    
    public int Id
    {
        get => _id;
        set
        {
            if (_id != value)
            {
                _id = value;
                OnPropertyChanged();
            }
        }
    }
    
    public string Name
    {
        get => _name;
        set
        {
            if (_name != value)
            {
                _name = value;
                OnPropertyChanged();
            }
        }
    }
    
    public decimal Price
    {
        get => _price;
        set
        {
            if (_price != value)
            {
                _price = value;
                OnPropertyChanged();
            }
        }
    }
    
    public DateTime CreatedDate
    {
        get => _createdDate;
        set
        {
            if (_createdDate != value)
            {
                _createdDate = value;
                OnPropertyChanged();
            }
        }
    }
    
    public event PropertyChangedEventHandler PropertyChanged;
    
    protected virtual void OnPropertyChanged([CallerMemberName] string propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
    
    public override string ToString() => $"{Name} (₩{Price:N0})";
}
```

### XAML에서의 Command 바인딩 활용

```xml
<!-- ProductManagementView.xaml -->
<Window x:Class="MyApp.Views.ProductManagementView"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="clr-namespace:MyApp.ViewModels"
        Title="제품 관리 시스템" Height="600" Width="900"
        WindowStartupLocation="CenterScreen">
    
    <Window.DataContext>
        <vm:ProductManagementViewModel/>
    </Window.DataContext>
    
    <Window.Resources>
        <!-- 스타일 정의 -->
        <Style TargetType="Button">
            <Setter Property="Padding" Value="8,4"/>
            <Setter Property="Margin" Value="4"/>
            <Setter Property="MinWidth" Value="80"/>
        </Style>
        
        <Style TargetType="TextBox">
            <Setter Property="Margin" Value="4"/>
            <Setter Property="VerticalAlignment" Value="Center"/>
        </Style>
        
        <Style TargetType="TextBlock">
            <Setter Property="Margin" Value="4"/>
            <Setter Property="VerticalAlignment" Value="Center"/>
        </Style>
        
        <!-- 제품 목록 아이템 템플릿 -->
        <DataTemplate x:Key="ProductItemTemplate">
            <Border Padding="8" Background="White" BorderBrush="#E0E0E0" 
                    BorderThickness="1" CornerRadius="4" Margin="2">
                <Grid>
                    <Grid.ColumnDefinitions>
                        <ColumnDefinition Width="Auto"/>
                        <ColumnDefinition Width="*"/>
                        <ColumnDefinition Width="Auto"/>
                        <ColumnDefinition Width="Auto"/>
                    </Grid.ColumnDefinitions>
                    
                    <!-- 제품 정보 -->
                    <TextBlock Grid.Column="0" Text="{Binding Id}" 
                               FontWeight="Bold" Width="40"/>
                    
                    <StackPanel Grid.Column="1" Margin="8,0">
                        <TextBlock Text="{Binding Name}" FontWeight="SemiBold"/>
                        <TextBlock Text="{Binding Price, StringFormat='₩{0:N0}'}" 
                                   Foreground="Green"/>
                    </StackPanel>
                    
                    <!-- 생성일 -->
                    <TextBlock Grid.Column="2" 
                               Text="{Binding CreatedDate, StringFormat='yyyy-MM-dd'}"
                               Foreground="Gray" Width="100"/>
                    
                    <!-- 액션 버튼 -->
                    <StackPanel Grid.Column="3" Orientation="Horizontal">
                        <Button Content="선택"
                                Command="{Binding DataContext.SelectProductCommand, 
                                          RelativeSource={RelativeSource AncestorType=Window}}"
                                CommandParameter="{Binding}"
                                ToolTip="이 제품 선택"/>
                                
                        <Button Content="삭제"
                                Command="{Binding DataContext.DeleteProductCommand, 
                                          RelativeSource={RelativeSource AncestorType=Window}}"
                                ToolTip="이 제품 삭제"/>
                    </StackPanel>
                </Grid>
            </Border>
        </DataTemplate>
    </Window.Resources>
    
    <Grid Margin="10">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>
        
        <!-- 제어 패널 -->
        <Border Grid.Row="0" Padding="12" Background="#F8F9FA" 
                CornerRadius="6" BorderBrush="#DEE2E6" BorderThickness="1">
            <Grid>
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="Auto"/>
                    <ColumnDefinition Width="*"/>
                    <ColumnDefinition Width="Auto"/>
                </Grid.ColumnDefinitions>
                
                <!-- 검색 영역 -->
                <StackPanel Grid.Column="0" Orientation="Horizontal">
                    <TextBlock Text="검색:" Margin="0,0,8,0" 
                               VerticalAlignment="Center"/>
                    <TextBox Text="{Binding SearchTerm, UpdateSourceTrigger=PropertyChanged}" 
                             Width="200" 
                             ToolTip="제품 이름으로 검색"/>
                </StackPanel>
                
                <!-- 폼 입력 영역 -->
                <StackPanel Grid.Column="1" Orientation="Horizontal" 
                            HorizontalAlignment="Center" Spacing="12">
                    <StackPanel Orientation="Horizontal">
                        <TextBlock Text="이름:" Width="40"/>
                        <TextBox Text="{Binding ProductName, UpdateSourceTrigger=PropertyChanged}" 
                                 Width="150"/>
                    </StackPanel>
                    
                    <StackPanel Orientation="Horizontal">
                        <TextBlock Text="가격:" Width="40"/>
                        <TextBox Text="{Binding ProductPrice, UpdateSourceTrigger=PropertyChanged}" 
                                 Width="100"/>
                    </StackPanel>
                </StackPanel>
                
                <!-- 액션 버튼 영역 -->
                <StackPanel Grid.Column="2" Orientation="Horizontal">
                    <Button Content="추가" 
                            Command="{Binding AddProductCommand}"
                            ToolTip="새 제품 추가"/>
                            
                    <Button Content="수정" 
                            Command="{Binding UpdateProductCommand}"
                            ToolTip="선택된 제품 정보 수정"/>
                            
                    <Button Content="초기화" 
                            Command="{Binding ClearFormCommand}"
                            ToolTip="입력 폼 초기화"/>
                            
                    <Separator Width="20" Visibility="Hidden"/>
                    
                    <Button Content="새로고침" 
                            Command="{Binding LoadProductsCommand}"
                            ToolTip="제품 목록 새로고침"/>
                            
                    <Button Content="내보내기" 
                            Command="{Binding ExportProductsCommand}"
                            ToolTip="제품 데이터 내보내기"/>
                </StackPanel>
            </Grid>
        </Border>
        
        <!-- 제품 목록 영역 -->
        <Border Grid.Row="1" Margin="0,10,0,0" Padding="10" 
                BorderBrush="#E0E0E0" BorderThickness="1" CornerRadius="6">
            <Grid>
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="*"/>
                </Grid.RowDefinitions>
                
                <TextBlock Grid.Row="0" Text="제품 목록" 
                           FontSize="16" FontWeight="Bold" Margin="0,0,0,10"/>
                
                <!-- 제품 목록 -->
                <ScrollViewer Grid.Row="1" VerticalScrollBarVisibility="Auto">
                    <ItemsControl ItemsSource="{Binding FilteredProducts}"
                                  ItemTemplate="{StaticResource ProductItemTemplate}">
                        <ItemsControl.ItemsPanel>
                            <ItemsPanelTemplate>
                                <StackPanel Orientation="Vertical" Spacing="4"/>
                            </ItemsPanelTemplate>
                        </ItemsControl.ItemsPanel>
                    </ItemsControl>
                </ScrollViewer>
            </Grid>
        </Border>
        
        <!-- 상태 표시줄 -->
        <StatusBar Grid.Row="2" Margin="0,10,0,0">
            <StatusBarItem>
                <TextBlock>
                    <Run Text="상태:" FontWeight="Bold"/>
                    <Run Text="{Binding StatusMessage}"/>
                </TextBlock>
            </StatusBarItem>
            
            <Separator/>
            
            <StatusBarItem>
                <TextBlock>
                    <Run Text="제품:"/>
                    <Run Text="{Binding Products.Count}"/>
                    <Run Text="개"/>
                </TextBlock>
            </StatusBarItem>
            
            <Separator/>
            
            <StatusBarItem>
                <ProgressBar Width="100" Height="16" 
                             IsIndeterminate="{Binding IsLoading}"
                             Visibility="{Binding IsLoading, 
                                        Converter={StaticResource BooleanToVisibilityConverter}}"/>
            </StatusBarItem>
        </StatusBar>
        
        <!-- 단축키 설정 -->
        <Window.InputBindings>
            <KeyBinding Command="{Binding AddProductCommand}" 
                        Key="Insert" Modifiers="None"/>
                        
            <KeyBinding Command="{Binding DeleteProductCommand}" 
                        Key="Delete" Modifiers="None"/>
                        
            <KeyBinding Command="{Binding ClearFormCommand}" 
                        Key="Escape" Modifiers="None"/>
                        
            <KeyBinding Command="{Binding LoadProductsCommand}" 
                        Key="F5" Modifiers="None"/>
        </Window.InputBindings>
        
        <!-- 컨텍스트 메뉴 -->
        <Window.ContextMenu>
            <ContextMenu>
                <MenuItem Header="제품 추가" 
                          Command="{Binding AddProductCommand}"/>
                          
                <MenuItem Header="새로고침" 
                          Command="{Binding LoadProductsCommand}"/>
                          
                <Separator/>
                
                <MenuItem Header="내보내기" 
                          Command="{Binding ExportProductsCommand}"/>
            </ContextMenu>
        </Window.ContextMenu>
    </Grid>
</Window>
```

## 고급 Command 패턴과 디자인 패턴

### Command 팩토리 패턴

대규모 애플리케이션에서는 Command 생성을 체계적으로 관리하는 패턴이 필요합니다:

```csharp
// Command 팩토리 인터페이스
public interface ICommandFactory
{
    ICommand CreateCommand(Action execute, Func<bool> canExecute = null);
    ICommand CreateCommand<T>(Action<T> execute, Predicate<T> canExecute = null);
    IAsyncCommand CreateAsyncCommand(Func<CancellationToken, Task> execute, Func<bool> canExecute = null);
    IAsyncCommand CreateAsyncCommand<T>(Func<T, CancellationToken, Task> execute, Predicate<T> canExecute = null);
}

// Command 팩토리 구현
public class CommandFactory : ICommandFactory
{
    private readonly IExceptionHandler _exceptionHandler;
    private readonly ILogger _logger;
    
    public CommandFactory(IExceptionHandler exceptionHandler, ILogger logger)
    {
        _exceptionHandler = exceptionHandler;
        _logger = logger;
    }
    
    public ICommand CreateCommand(Action execute, Func<bool> canExecute = null)
    {
        return new RelayCommand(
            execute: execute,
            canExecute: canExecute,
            commandName: GetCallerMethodName()
        );
    }
    
    public ICommand CreateCommand<T>(Action<T> execute, Predicate<T> canExecute = null)
    {
        return new RelayCommand<T>(
            execute: execute,
            canExecute: canExecute
        );
    }
    
    public IAsyncCommand CreateAsyncCommand(Func<CancellationToken, Task> execute, Func<bool> canExecute = null)
    {
        return new AsyncRelayCommand(
            asyncExecute: execute,
            canExecute: canExecute,
            errorHandler: _exceptionHandler.HandleCommandException
        );
    }
    
    public IAsyncCommand CreateAsyncCommand<T>(Func<T, CancellationToken, Task> execute, Predicate<T> canExecute = null)
    {
        return new AsyncRelayCommand<T>(
            asyncExecute: execute,
            canExecute: canExecute,
            errorHandler: _exceptionHandler.HandleCommandException
        );
    }
    
    private string GetCallerMethodName([CallerMemberName] string caller = null)
    {
        return caller ?? "UnknownCommand";
    }
}

// 비동기 Command 인터페이스
public interface IAsyncCommand : ICommand
{
    bool IsExecuting { get; }
    Task ExecuteAsync(object parameter);
    void Cancel();
}

// ViewModel에서의 팩토리 사용
public class OrderViewModel : ViewModelBase
{
    private readonly ICommandFactory _commandFactory;
    
    public OrderViewModel(ICommandFactory commandFactory)
    {
        _commandFactory = commandFactory;
        InitializeCommands();
    }
    
    private void InitializeCommands()
    {
        // 팩토리를 통한 Command 생성
        SubmitOrderCommand = _commandFactory.CreateCommand(
            execute: SubmitOrder,
            canExecute: CanSubmitOrder
        );
        
        // 비동기 Command 생성
        LoadOrderHistoryCommand = _commandFactory.CreateAsyncCommand(
            execute: LoadOrderHistoryAsync,
            canExecute: () => !IsLoading
        );
    }
    
    public ICommand SubmitOrderCommand { get; private set; }
    public IAsyncCommand LoadOrderHistoryCommand { get; private set; }
    
    // ... 나머지 구현
}
```

### Chain of Responsibility 패턴을 활용한 Command

복잡한 비즈니스 로직을 여러 단계로 분리하는 패턴:

```csharp
// Command 체인 인터페이스
public interface ICommandChain
{
    void AddCommand(ICommand command);
    bool CanExecute(object parameter);
    void Execute(object parameter);
}

// Command 체인 구현
public class CommandChain : ICommandChain
{
    private readonly List<ICommand> _commands = new List<ICommand>();
    private int _currentIndex = -1;
    
    public void AddCommand(ICommand command)
    {
        _commands.Add(command);
    }
    
    public bool CanExecute(object parameter)
    {
        return _commands.All(cmd => cmd.CanExecute(parameter));
    }
    
    public void Execute(object parameter)
    {
        if (!CanExecute(parameter))
            return;
        
        foreach (var command in _commands)
        {
            command.Execute(parameter);
        }
    }
}

// 사용 예시
public class OrderProcessingViewModel : ViewModelBase
{
    public ICommand ProcessOrderCommand { get; }
    
    public OrderProcessingViewModel()
    {
        var commandChain = new CommandChain();
        
        // 단계별 Command 추가
        commandChain.AddCommand(new RelayCommand(ValidateOrder));
        commandChain.AddCommand(new RelayCommand(CheckInventory));
        commandChain.AddCommand(new RelayCommand(CalculateTotal));
        commandChain.AddCommand(new RelayCommand(CreateInvoice));
        commandChain.AddCommand(new AsyncRelayCommand(SendConfirmationAsync));
        
        ProcessOrderCommand = new RelayCommand(
            execute: parameter => commandChain.Execute(parameter),
            canExecute: parameter => commandChain.CanExecute(parameter)
        );
    }
    
    private void ValidateOrder() { /* 주문 유효성 검증 */ }
    private void CheckInventory() { /* 재고 확인 */ }
    private void CalculateTotal() { /* 총액 계산 */ }
    private void CreateInvoice() { /* 청구서 생성 */ }
    private async Task SendConfirmationAsync(CancellationToken ct) 
    { /* 확인 메일 전송 */ }
}
```

## 성능 최적화와 모니터링

### Command 실행 모니터링

프로덕션 환경에서 Command의 성능을 모니터링하는 패턴:

```csharp
public class MonitoredRelayCommand : ICommand
{
    private readonly ICommand _innerCommand;
    private readonly IPerformanceMonitor _monitor;
    private readonly string _commandName;
    
    public MonitoredRelayCommand(
        ICommand innerCommand, 
        IPerformanceMonitor monitor, 
        string commandName)
    {
        _innerCommand = innerCommand ?? throw new ArgumentNullException(nameof(innerCommand));
        _monitor = monitor;
        _commandName = commandName;
    }
    
    public bool CanExecute(object parameter)
    {
        var watch = Stopwatch.StartNew();
        try
        {
            return _innerCommand.CanExecute(parameter);
        }
        finally
        {
            watch.Stop();
            _monitor.RecordCanExecuteTime(_commandName, watch.ElapsedMilliseconds);
        }
    }
    
    public void Execute(object parameter)
    {
        var watch = Stopwatch.StartNew();
        try
        {
            _innerCommand.Execute(parameter);
        }
        finally
        {
            watch.Stop();
            _monitor.RecordExecuteTime(_commandName, watch.ElapsedMilliseconds);
        }
    }
    
    public event EventHandler CanExecuteChanged
    {
        add => _innerCommand.CanExecuteChanged += value;
        remove => _innerCommand.CanExecuteChanged -= value;
    }
}

// 성능 모니터링 서비스
public interface IPerformanceMonitor
{
    void RecordCanExecuteTime(string commandName, long milliseconds);
    void RecordExecuteTime(string commandName, long milliseconds);
    IDictionary<string, CommandPerformanceStats> GetStats();
}

public class CommandPerformanceStats
{
    public string CommandName { get; set; }
    public long CanExecuteCalls { get; set; }
    public long ExecuteCalls { get; set; }
    public long TotalCanExecuteTime { get; set; }
    public long TotalExecuteTime { get; set; }
    public double AverageCanExecuteTime => CanExecuteCalls > 0 ? TotalCanExecuteTime / (double)CanExecuteCalls : 0;
    public double AverageExecuteTime => ExecuteCalls > 0 ? TotalExecuteTime / (double)ExecuteCalls : 0;
}
```

## 결론: Command 패턴의 미래와 진화

WPF의 `ICommand` 패턴은 단순한 기술적 구현을 넘어 소프트웨어 설계 철학을 반영합니다. 이 패턴의 진정한 가치는 다음과 같은 원칙들을 실천할 수 있게 해준다는 점에 있습니다:

**첫째, 관심사의 명확한 분리입니다.** Command 패턴은 사용자 입력 처리라는 관심사를 UI에서 비즈니스 로직으로 명확하게 분리합니다. 버튼 클릭, 메뉴 선택, 키보드 단축키 등 다양한 입력 소스가 동일한 Command로 수렴되면서, UI의 물리적 표현과 실제 동작의 논리가 독립적으로 진화할 수 있습니다.

**둘째, 상태 기반의 실행 제어입니다.** `CanExecute` 메커니즘은 단순한 활성화/비활성화를 넘어, 애플리케이션의 현재 상태에 기반한 동적 의사결정을 가능하게 합니다. 이는 사용자에게 직관적인 피드백을 제공하면서도, 복잡한 비즈니스 규칙을 깔끔하게 캡슐화합니다.

**셋째, 테스트 가능성과 유지보수성의 극대화입니다.** Command 객체는 순수한 .NET 객체이기 때문에 단위 테스트가 용이합니다. ViewModel의 Command 속성을 테스트하는 것은 곧 해당 뷰의 모든 사용자 상호작용을 테스트하는 것과 동일한 효과를 가집니다.

앞으로의 발전 방향을 살펴보면, Command 패턴은 더욱 정교한 비동기 처리, 취소 메커니즘, 실행 상태 추적, 성능 모니터링 등의 기능으로 진화하고 있습니다. 특히 마이크로서비스 아키텍처와 클라우드 네이티브 애플리케이션에서는 Command가 단순한 UI 상호작용을 넘어 분산 시스템 간의 통신 패턴으로 확장될 가능성도 있습니다.

WPF 개발자로서 Command 패턴을 깊이 이해하고 적절히 활용하는 것은 단순한 기술 습득이 아니라, 더 나은 소프트웨어 설계에 대한 사고방식을 체화하는 과정입니다. 올바르게 구현된 Command 시스템은 애플리케이션의 확장성, 테스트 용이성, 유지보수성을 동시에 향상시키는 강력한 도구가 될 것입니다.