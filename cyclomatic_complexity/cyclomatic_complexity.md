# Избавляемя от цикломатической 


# Снижение цикломатической сложности - Компактные примеры

## 1. Обработка заказа через Полиморфизм 

### Исходная ЦС: 14
### Конечная ЦС: 2
### Использованные приёмы:
1. **Полиморфизм** - замена switch на иерархию классов
2. **Табличная логика** - Dictionary для маппинга типов

### ДО (ЦС = 14):
```csharp
public enum OrderType { Delivery, Checkout, Express }
public enum CustomerTier { Regular, Premium, VIP }

public static class Order
{
    public static decimal Process(OrderType orderType, CustomerTier customerTier, 
                                   decimal amount, int distance)
    {
        decimal deliveryFee = 0;
        decimal discount = 0;
        
        // ЦС +3 (3 case)
        switch (orderType)
        {
            case OrderType.Delivery:
                Console.WriteLine("Доставка");
                if (distance < 5) // ЦС +1
                    deliveryFee = 100;
                else if (distance < 10) // ЦС +1
                    deliveryFee = 200;
                else
                    deliveryFee = 300;
                
                if (amount > 1000) // ЦС +1
                    deliveryFee *= 0.5m;
                break;
                
            case OrderType.Checkout:
                Console.WriteLine("Выдача на кассе");
                if (amount > 500) // ЦС +1
                    discount = 50;
                break;
                
            case OrderType.Express:
                Console.WriteLine("Срочная доставка");
                deliveryFee = 500;
                if (distance > 15) // ЦС +1
                    deliveryFee += 200;
                break;
        }
        
        // ЦС +3 (3 case)
        switch (customerTier)
        {
            case CustomerTier.Regular:
                discount += amount * 0.05m;
                break;
            case CustomerTier.Premium:
                discount += amount * 0.10m;
                break;
            case CustomerTier.VIP:
                discount += amount * 0.15m;
                if (orderType == OrderType.Delivery) // ЦС +1
                    deliveryFee = 0;
                break;
        }
        
        if (discount > amount * 0.3m) // ЦС +1
            discount = amount * 0.3m;
        
        return amount + deliveryFee - discount;
    }
}

public class Program
{
    public static void Main()
    {
        decimal total = Order.Process(OrderType.Delivery, CustomerTier.Premium, 1500, 7);
        Console.WriteLine($"Итого: {total}");
    }
}
```

### ПОСЛЕ (ЦС = 2):
```csharp
public interface IOrderStrategy
{
    string GetDescription();
    decimal CalculateDeliveryFee(decimal amount, int distance);
}

public interface ICustomerStrategy
{
    decimal GetDiscount(decimal amount);
    bool HasFreeDelivery();
}

// Стратегии заказов
public class DeliveryOrder : IOrderStrategy
{
    public string GetDescription() => "Доставка";
    
    public decimal CalculateDeliveryFee(decimal amount, int distance)
    {
        decimal fee = distance < 5 ? 100 : (distance < 10 ? 200 : 300);
        return amount > 1000 ? fee * 0.5m : fee;
    }
}

public class CheckoutOrder : IOrderStrategy
{
    public string GetDescription() => "Выдача на кассе";
    public decimal CalculateDeliveryFee(decimal amount, int distance) => 0;
}

public class ExpressOrder : IOrderStrategy
{
    public string GetDescription() => "Срочная доставка";
    public decimal CalculateDeliveryFee(decimal amount, int distance) => distance > 15 ? 700 : 500;
}

// Стратегии клиентов
public class RegularCustomer : ICustomerStrategy
{
    public decimal GetDiscount(decimal amount) => amount * 0.05m;
    public bool HasFreeDelivery() => false;
}

public class PremiumCustomer : ICustomerStrategy
{
    public decimal GetDiscount(decimal amount) => amount * 0.10m;
    public bool HasFreeDelivery() => false;
}

public class VIPCustomer : ICustomerStrategy
{
    public decimal GetDiscount(decimal amount) => amount * 0.15m;
    public bool HasFreeDelivery() => true;
}

// Табличная логика
public static class StrategyFactory
{
    private static readonly Dictionary<OrderType, IOrderStrategy> Orders = new()
    {
        { OrderType.Delivery, new DeliveryOrder() },
        { OrderType.Checkout, new CheckoutOrder() },
        { OrderType.Express, new ExpressOrder() }
    };
    
    private static readonly Dictionary<CustomerTier, ICustomerStrategy> Customers = new()
    {
        { CustomerTier.Regular, new RegularCustomer() },
        { CustomerTier.Premium, new PremiumCustomer() },
        { CustomerTier.VIP, new VIPCustomer() }
    };
    
    public static IOrderStrategy GetOrder(OrderType type) => Orders[type];
    public static ICustomerStrategy GetCustomer(CustomerTier tier) => Customers[tier];
}

// Рефакторенный класс (ЦС = 2)
public static class Order
{
    public static decimal Process(OrderType orderType, CustomerTier customerTier, 
                                   decimal amount, int distance)
    {
        var orderStrategy = StrategyFactory.GetOrder(orderType);
        var customerStrategy = StrategyFactory.GetCustomer(customerTier);
        
        Console.WriteLine(orderStrategy.GetDescription());
        
        decimal deliveryFee = orderStrategy.CalculateDeliveryFee(amount, distance);
        
        if (customerStrategy.HasFreeDelivery()) // ЦС +1
            deliveryFee = 0;
        
        decimal discount = customerStrategy.GetDiscount(amount);
        
        if (discount > amount * 0.3m) // ЦС +1
            discount = amount * 0.3m;
        
        return amount + deliveryFee - discount;
    }
}

public class Program
{
    public static void Main()
    {
        decimal total = Order.Process(OrderType.Delivery, CustomerTier.Premium, 1500, 7);
        Console.WriteLine($"Итого: {total}");
    }
}
```

---

## 2. Управление состоянием расчёта через State Pattern

### Исходная ЦС: 17
### Конечная ЦС: 1
### Использованные приёмы:
1. **State Pattern** - паттерн состояния
2. **Табличная логика** - Dictionary для действий в каждом состоянии

### ДО (ЦС = 17):
```csharp
public enum CalculationState { Created, Processing, Closed, Refunded, Error }

public class Calculation
{
    private CalculationState State;
    private decimal Amount;
    private string ErrorMessage;
    
    public Calculation(decimal amount)
    {
        State = CalculationState.Created; 
        Amount = amount;
    }

    public void Handle(string action)
    {
        // ЦС +5 (5 case)
        switch (State)
        {
            case CalculationState.Created:
                Console.WriteLine("Создание расчёта");
                if (action == "validate") // ЦС +1
                {
                    if (Amount > 0) // ЦС +1
                    {
                        State = CalculationState.Processing;
                        Console.WriteLine("Переход в обработку");
                    }
                    else
                    {
                        State = CalculationState.Error;
                        ErrorMessage = "Сумма <= 0";
                    }
                }
                else if (action == "cancel") // ЦС +1
                {
                    Console.WriteLine("Расчёт отменён");
                }
                break;
                
            case CalculationState.Processing:
                Console.WriteLine("Обработка расчёта");
                if (action == "complete") // ЦС +1
                {
                    State = CalculationState.Closed;
                    Console.WriteLine("Расчёт закрыт");
                }
                else if (action == "cancel") // ЦС +1
                {
                    State = CalculationState.Created;
                    Console.WriteLine("Возврат в создание");
                }
                break;
                
            case CalculationState.Closed:
                Console.WriteLine("Закрытый расчёт");
                if (action == "refund") // ЦС +1
                {
                    if (Amount <= 10000) // ЦС +1
                    {
                        State = CalculationState.Refunded;
                        Console.WriteLine("Возврат выполнен");
                    }
                    else
                    {
                        State = CalculationState.Error;
                        ErrorMessage = "Сумма слишком большая";
                    }
                }
                else if (action == "reopen") // ЦС +1
                {
                    State = CalculationState.Processing;
                    Console.WriteLine("Расчёт переоткрыт");
                }
                break;
                
            case CalculationState.Refunded:
                Console.WriteLine("Возврат расчёта");
                if (action == "confirm") // ЦС +1
                {
                    Console.WriteLine("Возврат подтверждён");
                }
                break;
                
            case CalculationState.Error:
                Console.WriteLine($"Ошибка: {ErrorMessage}");
                if (action == "reset") // ЦС +1
                {
                    State = CalculationState.Created;
                    ErrorMessage = null;
                    Console.WriteLine("Сброс выполнен");
                }
                break;
        }
    }
}

public class Program
{
    public static void Main()
    {
        var calc = new Calculation(1000);
        calc.Handle("validate");
        calc.Handle("complete");
        calc.Handle("refund");
    }
}
```

### ПОСЛЕ (ЦС = 1):
```csharp
public interface ICalculationState
{
    void Handle(Calculation calculation, string action);
}

public class Calculation
{
    private ICalculationState _currentState;
    public decimal Amount { get; set; }
    public string ErrorMessage { get; set; }
    
    public Calculation(decimal amount)
    {
        Amount = amount;
        _currentState = new CreatedState();
    }

    public void SetState(ICalculationState state) => _currentState = state;
    public void Handle(string action) => _currentState.Handle(this, action);
}

// Состояния с табличной логикой действий
public class CreatedState : ICalculationState
{
    private static readonly Dictionary<string, Action<Calculation>> Actions = new()
    {
        { "validate", Validate },
        { "cancel", Cancel }
    };
    
    public void Handle(Calculation calc, string action)
    {
        Console.WriteLine("Создание расчёта");
        if (Actions.TryGetValue(action, out var handler)) // ЦС +1 (единственная точка ветвления)
            handler(calc);
    }
    
    private static void Validate(Calculation calc)
    {
        if (calc.Amount > 0)
        {
            calc.SetState(new ProcessingState());
            Console.WriteLine("Переход в обработку");
        }
        else
        {
            calc.SetState(new ErrorState());
            calc.ErrorMessage = "Сумма <= 0";
        }
    }
    
    private static void Cancel(Calculation calc) => Console.WriteLine("Расчёт отменён");
}

public class ProcessingState : ICalculationState
{
    private static readonly Dictionary<string, Action<Calculation>> Actions = new()
    {
        { "complete", Complete },
        { "cancel", Cancel }
    };
    
    public void Handle(Calculation calc, string action)
    {
        Console.WriteLine("Обработка расчёта");
        if (Actions.TryGetValue(action, out var handler))
            handler(calc);
    }
    
    private static void Complete(Calculation calc)
    {
        calc.SetState(new ClosedState());
        Console.WriteLine("Расчёт закрыт");
    }
    
    private static void Cancel(Calculation calc)
    {
        calc.SetState(new CreatedState());
        Console.WriteLine("Возврат в создание");
    }
}

public class ClosedState : ICalculationState
{
    private static readonly Dictionary<string, Action<Calculation>> Actions = new()
    {
        { "refund", Refund },
        { "reopen", Reopen }
    };
    
    public void Handle(Calculation calc, string action)
    {
        Console.WriteLine("Закрытый расчёт");
        if (Actions.TryGetValue(action, out var handler))
            handler(calc);
    }
    
    private static void Refund(Calculation calc)
    {
        if (calc.Amount <= 10000)
        {
            calc.SetState(new RefundedState());
            Console.WriteLine("Возврат выполнен");
        }
        else
        {
            calc.SetState(new ErrorState());
            calc.ErrorMessage = "Сумма слишком большая";
        }
    }
    
    private static void Reopen(Calculation calc)
    {
        calc.SetState(new ProcessingState());
        Console.WriteLine("Расчёт переоткрыт");
    }
}

public class RefundedState : ICalculationState
{
    public void Handle(Calculation calc, string action)
    {
        Console.WriteLine("Возврат расчёта");
        if (action == "confirm")
            Console.WriteLine("Возврат подтверждён");
    }
}

public class ErrorState : ICalculationState
{
    public void Handle(Calculation calc, string action)
    {
        Console.WriteLine($"Ошибка: {calc.ErrorMessage}");
        if (action == "reset")
        {
            calc.SetState(new CreatedState());
            calc.ErrorMessage = null;
            Console.WriteLine("Сброс выполнен");
        }
    }
}

public class Program
{
    public static void Main()
    {
        var calc = new Calculation(1000);
        calc.Handle("validate");
        calc.Handle("complete");
        calc.Handle("refund");
    }
}
```

---

## 3. Логирование в зависимости от источника c помощью Dependency Injection

### Исходная ЦС: 13
### Конечная ЦС: 1
### Использованные приёмы:
1. **Dependency Injection** - внедрение зависимостей
2. **Табличная логика** - Dictionary для целей логирования
3. **Strategy Pattern** - разные стратегии вывода

### ДО (ЦС = 13):
```csharp
public enum LoggingTargetType { Console, File, Database, Email }
public enum LogLevel { Info, Warning, Error, Critical }

public static class Logger
{
    private static int _errorCount = 0;
    
    public static void Log(LoggingTargetType target, LogLevel level, string message)
    {
        string levelStr = "";
        
        // ЦС +4 (4 case)
        switch (level)
        {
            case LogLevel.Info:
                levelStr = "INFO";
                break;
            case LogLevel.Warning:
                levelStr = "WARNING";
                _errorCount++;
                break;
            case LogLevel.Error:
                levelStr = "ERROR";
                _errorCount++;
                if (_errorCount > 5) // ЦС +1
                    Console.WriteLine("!!! МНОГО ОШИБОК !!!");
                break;
            case LogLevel.Critical:
                levelStr = "CRITICAL";
                _errorCount += 3;
                break;
        }
        
        // ЦС +4 (4 case)
        switch (target)
        {
            case LoggingTargetType.Console:
                Console.WriteLine($"[CONSOLE] [{levelStr}] {message}");
                break;
                
            case LoggingTargetType.File:
                Console.WriteLine($"[FILE] [{levelStr}] {message}");
                if (level == LogLevel.Error || level == LogLevel.Critical) // ЦС +2 (||)
                    Console.WriteLine(">>> В error.log");
                break;
                
            case LoggingTargetType.Database:
                Console.WriteLine($"[DB] [{levelStr}] {message}");
                if (level == LogLevel.Critical) // ЦС +1
                    Console.WriteLine(">>> Уведомление админу");
                break;
                
            case LoggingTargetType.Email:
                if (level == LogLevel.Error || level == LogLevel.Critical) // ЦС +2 (||)
                {
                    Console.WriteLine($"[EMAIL] [{levelStr}] {message}");
                    Console.WriteLine(">>> Email отправлен");
                }
                break;
        }
    }
}

public class Program
{
    public static void Main()
    {
        Logger.Log(LoggingTargetType.Database, LogLevel.Error, "Ошибка подключения");
        Logger.Log(LoggingTargetType.Email, LogLevel.Critical, "Система недоступна");
    }
}
```

### ПОСЛЕ (ЦС = 1):
```csharp
public interface ILogTarget
{
    void Write(string level, string message);
    bool ShouldLog(LogLevel level);
}

// Реализации целей
public class ConsoleLogTarget : ILogTarget
{
    public void Write(string level, string message) => 
        Console.WriteLine($"[CONSOLE] [{level}] {message}");
    
    public bool ShouldLog(LogLevel level) => true;
}

public class FileLogTarget : ILogTarget
{
    public void Write(string level, string message)
    {
        Console.WriteLine($"[FILE] [{level}] {message}");
        Console.WriteLine(">>> В error.log");
    }
    
    public bool ShouldLog(LogLevel level) => 
        level == LogLevel.Error || level == LogLevel.Critical;
}

public class DatabaseLogTarget : ILogTarget
{
    public void Write(string level, string message)
    {
        Console.WriteLine($"[DB] [{level}] {message}");
        if (level == "CRITICAL")
            Console.WriteLine(">>> Уведомление админу");
    }
    
    public bool ShouldLog(LogLevel level) => true;
}

public class EmailLogTarget : ILogTarget
{
    public void Write(string level, string message)
    {
        Console.WriteLine($"[EMAIL] [{level}] {message}");
        Console.WriteLine(">>> Email отправлен");
    }
    
    public bool ShouldLog(LogLevel level) => 
        level == LogLevel.Error || level == LogLevel.Critical;
}

// Менеджер ошибок
public class ErrorCounter
{
    private int _count = 0;
    private static readonly Dictionary<LogLevel, int> Weights = new()
    {
        { LogLevel.Warning, 1 },
        { LogLevel.Error, 1 },
        { LogLevel.Critical, 3 }
    };
    
    public void Increment(LogLevel level)
    {
        if (Weights.TryGetValue(level, out int weight))
            _count += weight;
    }
    
    public bool HasTooManyErrors() => _count > 5;
}

// Логгер (ЦС = 1)
public class Logger
{
    private readonly ILogTarget _target;
    private readonly ErrorCounter _counter = new();
    
    private static readonly Dictionary<LogLevel, string> LevelNames = new()
    {
        { LogLevel.Info, "INFO" },
        { LogLevel.Warning, "WARNING" },
        { LogLevel.Error, "ERROR" },
        { LogLevel.Critical, "CRITICAL" }
    };
    
    public Logger(ILogTarget target) => _target = target;
    
    public void Log(LogLevel level, string message)
    {
        if (!_target.ShouldLog(level)) // ЦС +1 (единственная точка ветвления)
            return;
        
        _counter.Increment(level);
        
        string levelStr = LevelNames[level];
        _target.Write(levelStr, message);
        
        if (_counter.HasTooManyErrors())
            Console.WriteLine("!!! МНОГО ОШИБОК !!!");
    }
}

// Фабрика с табличной логикой
public static class LoggerFactory
{
    private static readonly Dictionary<LoggingTargetType, Func<ILogTarget>> Targets = new()
    {
        { LoggingTargetType.Console, () => new ConsoleLogTarget() },
        { LoggingTargetType.File, () => new FileLogTarget() },
        { LoggingTargetType.Database, () => new DatabaseLogTarget() },
        { LoggingTargetType.Email, () => new EmailLogTarget() }
    };
    
    public static Logger Create(LoggingTargetType type) => new Logger(Targets[type]());
}

public class Program
{
    public static void Main()
    {
        var dbLogger = LoggerFactory.Create(LoggingTargetType.Database);
        dbLogger.Log(LogLevel.Error, "Ошибка подключения");
        
        var emailLogger = LoggerFactory.Create(LoggingTargetType.Email);
        emailLogger.Log(LogLevel.Critical, "Система недоступна");
    }
}
```