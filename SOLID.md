# SOLID Principles in C#

The SOLID principles are five design principles that make software designs more understandable, flexible, and maintainable.

## 1. Single Responsibility Principle (SRP)

**Definition**: A class should have only one reason to change, meaning it should have only one job or responsibility.

### ❌ Bad Example (Violates SRP)

```csharp
// This class has multiple responsibilities
public class UserManager
{
    public void CreateUser(string username, string email)
    {
        // Create user logic
        Console.WriteLine($"User {username} created");
    }

    public void SendWelcomeEmail(string email)
    {
        // Email sending logic
        Console.WriteLine($"Welcome email sent to {email}");
    }

    public void LogUserActivity(string username, string activity)
    {
        // Logging logic
        Console.WriteLine($"[LOG] {username}: {activity}");
    }

    public void SaveToDatabase(string username, string email)
    {
        // Database logic
        Console.WriteLine($"Saved {username} to database");
    }
}
```

### ✅ Good Example (Follows SRP)

```csharp
// Each class has a single responsibility
public class User
{
    public string Username { get; set; }
    public string Email { get; set; }
}

public class UserService
{
    private readonly IEmailService _emailService;
    private readonly IUserRepository _userRepository;
    private readonly ILogger _logger;

    public UserService(IEmailService emailService, IUserRepository userRepository, ILogger logger)
    {
        _emailService = emailService;
        _userRepository = userRepository;
        _logger = logger;
    }

    public void CreateUser(string username, string email)
    {
        var user = new User { Username = username, Email = email };
        _userRepository.Save(user);
        _logger.Log($"User {username} created");
        _emailService.SendWelcomeEmail(email);
    }
}

public interface IEmailService
{
    void SendWelcomeEmail(string email);
}

public class EmailService : IEmailService
{
    public void SendWelcomeEmail(string email)
    {
        Console.WriteLine($"Welcome email sent to {email}");
    }
}

public interface IUserRepository
{
    void Save(User user);
}

public class UserRepository : IUserRepository
{
    public void Save(User user)
    {
        Console.WriteLine($"Saved {user.Username} to database");
    }
}

public interface ILogger
{
    void Log(string message);
}

public class Logger : ILogger
{
    public void Log(string message)
    {
        Console.WriteLine($"[LOG] {message}");
    }
}
```

---

## 2. Open/Closed Principle (OCP)

**Definition**: Software entities should be open for extension but closed for modification.

### ❌ Bad Example (Violates OCP)

```csharp
public class DiscountCalculator
{
    public decimal CalculateDiscount(string customerType, decimal amount)
    {
        if (customerType == "Regular")
        {
            return amount * 0.05m;
        }
        else if (customerType == "Premium")
        {
            return amount * 0.10m;
        }
        else if (customerType == "VIP")
        {
            return amount * 0.20m;
        }
        
        return 0;
    }
}
// Adding a new customer type requires modifying this class
```

### ✅ Good Example (Follows OCP)

```csharp
public interface IDiscountStrategy
{
    decimal CalculateDiscount(decimal amount);
}

public class RegularCustomerDiscount : IDiscountStrategy
{
    public decimal CalculateDiscount(decimal amount)
    {
        return amount * 0.05m;
    }
}

public class PremiumCustomerDiscount : IDiscountStrategy
{
    public decimal CalculateDiscount(decimal amount)
    {
        return amount * 0.10m;
    }
}

public class VIPCustomerDiscount : IDiscountStrategy
{
    public decimal CalculateDiscount(decimal amount)
    {
        return amount * 0.20m;
    }
}

public class DiscountCalculator
{
    public decimal CalculateDiscount(IDiscountStrategy strategy, decimal amount)
    {
        return strategy.CalculateDiscount(amount);
    }
}

// Usage:
// var calculator = new DiscountCalculator();
// var discount = calculator.CalculateDiscount(new VIPCustomerDiscount(), 1000);

// Adding a new customer type doesn't require modifying existing code
public class GoldCustomerDiscount : IDiscountStrategy
{
    public decimal CalculateDiscount(decimal amount)
    {
        return amount * 0.15m;
    }
}
```

---

## 3. Liskov Substitution Principle (LSP)

**Definition**: Objects of a superclass should be replaceable with objects of a subclass without breaking the application.

### ❌ Bad Example (Violates LSP)

```csharp
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }

    public int GetArea()
    {
        return Width * Height;
    }
}

public class Square : Rectangle
{
    // Square violates LSP because it changes the behavior
    public override int Width
    {
        get => base.Width;
        set
        {
            base.Width = value;
            base.Height = value; // Side effect!
        }
    }

    public override int Height
    {
        get => base.Height;
        set
        {
            base.Height = value;
            base.Width = value; // Side effect!
        }
    }
}

// This breaks:
// Rectangle rect = new Square();
// rect.Width = 5;
// rect.Height = 10;
// Console.WriteLine(rect.GetArea()); // Expected: 50, Actual: 100
```

### ✅ Good Example (Follows LSP)

```csharp
public abstract class Shape
{
    public abstract int GetArea();
}

public class Rectangle : Shape
{
    public int Width { get; set; }
    public int Height { get; set; }

    public Rectangle(int width, int height)
    {
        Width = width;
        Height = height;
    }

    public override int GetArea()
    {
        return Width * Height;
    }
}

public class Square : Shape
{
    public int Side { get; set; }

    public Square(int side)
    {
        Side = side;
    }

    public override int GetArea()
    {
        return Side * Side;
    }
}

// Usage:
// Shape shape1 = new Rectangle(5, 10);
// Shape shape2 = new Square(5);
// Console.WriteLine(shape1.GetArea()); // 50
// Console.WriteLine(shape2.GetArea()); // 25
```

---

## 4. Interface Segregation Principle (ISP)

**Definition**: Clients should not be forced to depend on interfaces they do not use.

### ❌ Bad Example (Violates ISP)

```csharp
// Fat interface that forces implementations to implement methods they don't need
public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
    void GetPaid();
}

public class HumanWorker : IWorker
{
    public void Work() => Console.WriteLine("Working...");
    public void Eat() => Console.WriteLine("Eating lunch...");
    public void Sleep() => Console.WriteLine("Sleeping...");
    public void GetPaid() => Console.WriteLine("Getting paid...");
}

public class RobotWorker : IWorker
{
    public void Work() => Console.WriteLine("Working...");
    public void GetPaid() => Console.WriteLine("Getting paid...");
    
    // Robots don't eat or sleep!
    public void Eat() => throw new NotImplementedException();
    public void Sleep() => throw new NotImplementedException();
}
```

### ✅ Good Example (Follows ISP)

```csharp
// Segregated interfaces
public interface IWorkable
{
    void Work();
}

public interface IFeedable
{
    void Eat();
}

public interface ISleepable
{
    void Sleep();
}

public interface IPayable
{
    void GetPaid();
}

public class HumanWorker : IWorkable, IFeedable, ISleepable, IPayable
{
    public void Work() => Console.WriteLine("Working...");
    public void Eat() => Console.WriteLine("Eating lunch...");
    public void Sleep() => Console.WriteLine("Sleeping...");
    public void GetPaid() => Console.WriteLine("Getting paid...");
}

public class RobotWorker : IWorkable, IPayable
{
    public void Work() => Console.WriteLine("Working...");
    public void GetPaid() => Console.WriteLine("Getting paid...");
    // No need to implement Eat() or Sleep()
}

public class Manager
{
    public void ManageWork(IWorkable worker)
    {
        worker.Work();
    }

    public void ProcessPayroll(IPayable worker)
    {
        worker.GetPaid();
    }
}
```

---

## 5. Dependency Inversion Principle (DIP)

**Definition**: High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details. Details should depend on abstractions.

### ❌ Bad Example (Violates DIP)

```csharp
// High-level module depends on low-level module directly
public class EmailNotification
{
    public void Send(string message)
    {
        Console.WriteLine($"Email sent: {message}");
    }
}

public class NotificationService
{
    private EmailNotification _emailNotification;

    public NotificationService()
    {
        // Tight coupling to concrete implementation
        _emailNotification = new EmailNotification();
    }

    public void Notify(string message)
    {
        _emailNotification.Send(message);
    }
}
// Difficult to add SMS or Push notifications without modifying NotificationService
```

### ✅ Good Example (Follows DIP)

```csharp
// Abstraction
public interface INotification
{
    void Send(string message);
}

// Low-level modules
public class EmailNotification : INotification
{
    public void Send(string message)
    {
        Console.WriteLine($"Email sent: {message}");
    }
}

public class SMSNotification : INotification
{
    public void Send(string message)
    {
        Console.WriteLine($"SMS sent: {message}");
    }
}

public class PushNotification : INotification
{
    public void Send(string message)
    {
        Console.WriteLine($"Push notification sent: {message}");
    }
}

// High-level module depends on abstraction
public class NotificationService
{
    private readonly INotification _notification;

    // Dependency injection
    public NotificationService(INotification notification)
    {
        _notification = notification;
    }

    public void Notify(string message)
    {
        _notification.Send(message);
    }
}

// Usage:
// var emailService = new NotificationService(new EmailNotification());
// emailService.Notify("Hello via Email!");

// var smsService = new NotificationService(new SMSNotification());
// smsService.Notify("Hello via SMS!");
```

---

## Summary

| Principle | Key Takeaway |
|-----------|-------------|
| **Single Responsibility** | One class = One responsibility |
| **Open/Closed** | Open for extension, closed for modification |
| **Liskov Substitution** | Subtypes must be substitutable for their base types |
| **Interface Segregation** | Many specific interfaces > One general interface |
| **Dependency Inversion** | Depend on abstractions, not concretions |

## Benefits of SOLID Principles

- **Maintainability**: Easier to understand and modify code
- **Testability**: Easier to write unit tests
- **Flexibility**: Code can adapt to changing requirements
- **Reusability**: Components can be reused in different contexts
- **Scalability**: Easier to extend functionality without breaking existing code
