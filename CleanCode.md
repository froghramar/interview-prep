# Clean Code Principles in C#

A comprehensive guide to writing clean, maintainable, and professional code.

**Sources**:
- "Clean Code: A Handbook of Agile Software Craftsmanship" by Robert C. Martin
- "Code Complete" by Steve McConnell
- Microsoft C# Coding Conventions (https://docs.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions)
- ".NET Framework Design Guidelines" by Krzysztof Cwalina & Brad Abrams

---

## Table of Contents
1. [Meaningful Names](#meaningful-names)
2. [Functions](#functions)
3. [Comments](#comments)
4. [Formatting](#formatting)
5. [Error Handling](#error-handling)
6. [Classes](#classes)
7. [Code Smells](#code-smells)
8. [Refactoring Examples](#refactoring-examples)

---

## Meaningful Names

### Use Intention-Revealing Names

Names should reveal intent and make code self-documenting.

#### ❌ Bad Examples

```csharp
int d; // elapsed time in days
List<int[]> lst1;
public void DoStuff() { }

// What does this do?
if (x == 1)
{
    y = 50;
}
```

#### ✅ Good Examples

```csharp
int elapsedTimeInDays;
List<Customer> activeCustomers;
public void SendWelcomeEmail() { }

// Crystal clear intent
if (user.IsAdmin())
{
    maxLoginAttempts = 50;
}
```

### Use Pronounceable Names

```csharp
// ❌ Bad
class DtaRcrd102
{
    private DateTime genymdhms;
    private DateTime modymdhms;
}

// ✅ Good
class Customer
{
    private DateTime generationTimestamp;
    private DateTime modificationTimestamp;
}
```

### Use Searchable Names

```csharp
// ❌ Bad - Magic numbers
for (int i = 0; i < 7; i++)
{
    // What is 7?
}
if (status == 4)
{
    // What is 4?
}

// ✅ Good - Named constants
const int DAYS_IN_WEEK = 7;
const int STATUS_COMPLETED = 4;

for (int i = 0; i < DAYS_IN_WEEK; i++)
{
    // Clear meaning
}
if (status == STATUS_COMPLETED)
{
    // Self-documenting
}
```

### Avoid Mental Mapping

```csharp
// ❌ Bad - What is 'r'?
foreach (var r in records)
{
    // Process r
}

// ✅ Good - Explicit and clear
foreach (var customer in customers)
{
    ProcessCustomer(customer);
}
```

### Class Names Should Be Nouns

```csharp
// ❌ Bad
public class ProcessData { }
public class Manager { }

// ✅ Good
public class Customer { }
public class Account { }
public class OrderProcessor { }
public class UserManager { }
```

### Method Names Should Be Verbs

```csharp
// ❌ Bad
public void Data() { }
public void Information() { }

// ✅ Good
public void SaveCustomer() { }
public void DeleteOrder() { }
public void CalculateTotal() { }
public bool IsValid() { }
public Customer GetCustomerById(int id) { }
```

### Pick One Word Per Concept

```csharp
// ❌ Bad - Inconsistent terminology
public Customer FetchCustomer(int id) { }
public Order RetrieveOrder(int id) { }
public Product GetProduct(int id) { }

// ✅ Good - Consistent terminology
public Customer GetCustomer(int id) { }
public Order GetOrder(int id) { }
public Product GetProduct(int id) { }
```

**Source**: Clean Code by Robert C. Martin, Chapter 2

---

## Functions

### Small Functions

Functions should be small and do one thing well.

#### ❌ Bad Example - Too Long, Does Too Much

```csharp
public void ProcessOrder(Order order)
{
    // Validate order
    if (order == null)
        throw new ArgumentNullException(nameof(order));
    
    if (order.Items.Count == 0)
        throw new InvalidOperationException("Order has no items");
    
    foreach (var item in order.Items)
    {
        if (item.Quantity <= 0)
            throw new InvalidOperationException("Invalid quantity");
    }
    
    // Calculate total
    decimal total = 0;
    foreach (var item in order.Items)
    {
        total += item.Price * item.Quantity;
    }
    
    // Apply discount
    if (order.Customer.IsPremium)
    {
        total *= 0.9m; // 10% discount
    }
    
    // Check inventory
    foreach (var item in order.Items)
    {
        var stock = _db.Inventory.Find(item.ProductId);
        if (stock.Quantity < item.Quantity)
            throw new InvalidOperationException("Insufficient stock");
    }
    
    // Update inventory
    foreach (var item in order.Items)
    {
        var stock = _db.Inventory.Find(item.ProductId);
        stock.Quantity -= item.Quantity;
        _db.SaveChanges();
    }
    
    // Save order
    order.Total = total;
    order.Status = OrderStatus.Confirmed;
    _db.Orders.Add(order);
    _db.SaveChanges();
    
    // Send email
    var emailBody = $"Order #{order.Id} confirmed. Total: ${total}";
    _emailService.Send(order.Customer.Email, "Order Confirmed", emailBody);
    
    // Log
    _logger.Log($"Order {order.Id} processed for customer {order.Customer.Id}");
}
```

#### ✅ Good Example - Single Responsibility Functions

```csharp
public void ProcessOrder(Order order)
{
    ValidateOrder(order);
    CalculateOrderTotal(order);
    CheckInventoryAvailability(order);
    UpdateInventory(order);
    SaveOrder(order);
    SendConfirmationEmail(order);
    LogOrderProcessing(order);
}

private void ValidateOrder(Order order)
{
    if (order == null)
        throw new ArgumentNullException(nameof(order));
    
    if (!order.Items.Any())
        throw new InvalidOperationException("Order must contain at least one item");
    
    if (order.Items.Any(item => item.Quantity <= 0))
        throw new InvalidOperationException("All items must have a positive quantity");
}

private void CalculateOrderTotal(Order order)
{
    decimal subtotal = order.Items.Sum(item => item.Price * item.Quantity);
    decimal discount = CalculateDiscount(order.Customer, subtotal);
    order.Total = subtotal - discount;
}

private decimal CalculateDiscount(Customer customer, decimal subtotal)
{
    return customer.IsPremium ? subtotal * 0.1m : 0;
}

private void CheckInventoryAvailability(Order order)
{
    foreach (var item in order.Items)
    {
        var stock = _inventoryService.GetStock(item.ProductId);
        if (stock < item.Quantity)
            throw new InsufficientStockException(item.ProductId);
    }
}

private void UpdateInventory(Order order)
{
    foreach (var item in order.Items)
    {
        _inventoryService.DeductStock(item.ProductId, item.Quantity);
    }
}

private void SaveOrder(Order order)
{
    order.Status = OrderStatus.Confirmed;
    _orderRepository.Add(order);
}

private void SendConfirmationEmail(Order order)
{
    var message = _emailFormatter.FormatOrderConfirmation(order);
    _emailService.SendAsync(order.Customer.Email, message);
}

private void LogOrderProcessing(Order order)
{
    _logger.LogInformation(
        "Order {OrderId} processed for customer {CustomerId}",
        order.Id,
        order.Customer.Id);
}
```

### Function Arguments

Limit the number of arguments (ideally 0-2, max 3).

```csharp
// ❌ Bad - Too many parameters
public Order CreateOrder(
    int customerId,
    string customerName,
    string customerEmail,
    string shippingAddress,
    string shippingCity,
    string shippingZip,
    string billingAddress,
    string billingCity,
    string billingZip,
    decimal total)
{
    // ...
}

// ✅ Good - Use objects to encapsulate related data
public Order CreateOrder(OrderRequest request)
{
    // ...
}

public class OrderRequest
{
    public int CustomerId { get; set; }
    public CustomerInfo Customer { get; set; }
    public Address ShippingAddress { get; set; }
    public Address BillingAddress { get; set; }
    public decimal Total { get; set; }
}
```

### No Side Effects

Functions should do what their name suggests and nothing more.

```csharp
// ❌ Bad - Has hidden side effect
public bool CheckPassword(string username, string password)
{
    var user = _db.Users.Find(username);
    if (user != null && user.Password == password)
    {
        Session.Initialize(user); // Side effect!
        return true;
    }
    return false;
}

// ✅ Good - Separate concerns
public bool IsPasswordValid(string username, string password)
{
    var user = _userRepository.GetByUsername(username);
    return user != null && _passwordService.Verify(password, user.PasswordHash);
}

public void Login(string username, string password)
{
    if (IsPasswordValid(username, password))
    {
        var user = _userRepository.GetByUsername(username);
        _sessionService.Initialize(user);
    }
}
```

### Command Query Separation

A function should either do something or answer something, not both.

```csharp
// ❌ Bad - Does both
public bool SetAttribute(string attribute, string value)
{
    if (AttributeExists(attribute))
    {
        _attributes[attribute] = value;
        return true;
    }
    return false;
}

// Usage is confusing
if (SetAttribute("username", "john"))
{
    // Did we set it or was it already set?
}

// ✅ Good - Separate command and query
public bool AttributeExists(string attribute)
{
    return _attributes.ContainsKey(attribute);
}

public void SetAttribute(string attribute, string value)
{
    _attributes[attribute] = value;
}

// Clear usage
if (AttributeExists("username"))
{
    SetAttribute("username", "john");
}
```

### Don't Repeat Yourself (DRY)

```csharp
// ❌ Bad - Duplication
public decimal CalculateEmployeeDiscount(Employee emp, decimal price)
{
    if (emp.YearsOfService > 5)
        return price * 0.9m;
    return price;
}

public decimal CalculateManagerDiscount(Manager mgr, decimal price)
{
    if (mgr.YearsOfService > 5)
        return price * 0.9m;
    return price;
}

// ✅ Good - Extract common logic
public decimal CalculateLoyaltyDiscount(IStaffMember staff, decimal price)
{
    const decimal LOYALTY_THRESHOLD_YEARS = 5;
    const decimal LOYALTY_DISCOUNT_RATE = 0.9m;
    
    if (staff.YearsOfService > LOYALTY_THRESHOLD_YEARS)
        return price * LOYALTY_DISCOUNT_RATE;
    
    return price;
}
```

**Source**: Clean Code by Robert C. Martin, Chapter 3

---

## Comments

### Good Code > Comments

Code should be self-explanatory. Comments are often a sign of bad code.

#### ❌ Bad - Unnecessary Comments

```csharp
// Check if employee is eligible for benefits
if (employee.Age > 18 && employee.HoursWorked > 30)
{
    // ...
}

// Increment counter by 1
counter++;

// Loop through all users
foreach (var user in users)
{
    // Process user
    ProcessUser(user);
}
```

#### ✅ Good - Self-Documenting Code

```csharp
if (employee.IsEligibleForBenefits())
{
    // ...
}

counter++;

foreach (var user in users)
{
    ProcessUser(user);
}

// In Employee class:
public bool IsEligibleForBenefits()
{
    const int MINIMUM_AGE = 18;
    const int MINIMUM_HOURS = 30;
    
    return Age > MINIMUM_AGE && HoursWorked > MINIMUM_HOURS;
}
```

### When Comments Are Useful

```csharp
// ✅ Legal comments
// Copyright (c) 2025 Company Name. All rights reserved.

// ✅ Explanation of intent
// We're using a hash here because performance is critical
// and we need O(1) lookups for millions of records
private Dictionary<string, User> _userCache = new();

// ✅ Warning of consequences
// DO NOT change this without updating the mobile app!
// The app depends on this exact format.
public string SerializeForMobile()
{
    return JsonConvert.SerializeObject(data);
}

// ✅ TODO comments
// TODO: Implement caching for better performance
public List<Product> GetProducts()
{
    return _db.Products.ToList();
}

// ✅ XML documentation for public APIs
/// <summary>
/// Calculates the discount based on customer loyalty tier.
/// </summary>
/// <param name="customer">The customer to calculate discount for</param>
/// <param name="amount">The original amount before discount</param>
/// <returns>The discounted amount</returns>
/// <exception cref="ArgumentNullException">Thrown when customer is null</exception>
public decimal CalculateDiscount(Customer customer, decimal amount)
{
    // Implementation
}
```

### Comments to Avoid

```csharp
// ❌ Redundant comments
// Default constructor
public Customer() { }

// ❌ Misleading comments
// Returns the customer name
public int GetCustomerId() { } // Returns ID, not name!

// ❌ Commented-out code (use version control instead)
public void ProcessOrder(Order order)
{
    // var total = CalculateTotal(order);
    // ApplyDiscount(total);
    ValidateOrder(order);
    SaveOrder(order);
}

// ❌ Journal comments (use git history)
// Changes:
// 2025-01-15 - Added validation
// 2025-02-03 - Fixed bug in calculation
// 2025-03-20 - Refactored for performance

// ❌ Noise comments
// The day of the month
private int dayOfMonth;
```

**Source**: Clean Code by Robert C. Martin, Chapter 4

---

## Formatting

### Vertical Formatting

```csharp
// ✅ Good - Related concepts are close together
public class OrderService
{
    // Dependencies at the top
    private readonly IOrderRepository _orderRepository;
    private readonly IEmailService _emailService;
    private readonly ILogger<OrderService> _logger;

    // Constructor
    public OrderService(
        IOrderRepository orderRepository,
        IEmailService emailService,
        ILogger<OrderService> logger)
    {
        _orderRepository = orderRepository;
        _emailService = emailService;
        _logger = logger;
    }

    // Related public methods together
    public async Task<Order> CreateOrderAsync(OrderRequest request)
    {
        var order = await BuildOrderAsync(request);
        await ValidateOrderAsync(order);
        await SaveOrderAsync(order);
        await NotifyCustomerAsync(order);
        return order;
    }

    public async Task<Order> GetOrderAsync(int orderId)
    {
        return await _orderRepository.GetByIdAsync(orderId);
    }

    // Private helper methods near their callers
    private async Task<Order> BuildOrderAsync(OrderRequest request)
    {
        // Implementation
    }

    private async Task ValidateOrderAsync(Order order)
    {
        // Implementation
    }

    private async Task SaveOrderAsync(Order order)
    {
        // Implementation
    }

    private async Task NotifyCustomerAsync(Order order)
    {
        // Implementation
    }
}
```

### Horizontal Formatting

```csharp
// ✅ Good formatting
public class Product
{
    // Align related assignments
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    
    // Spacing around operators
    public decimal CalculateTotal(int quantity, decimal discount)
    {
        decimal subtotal = Price * quantity;
        decimal discountAmount = subtotal * discount;
        return subtotal - discountAmount;
    }
    
    // Clear hierarchy with indentation
    public void ProcessItems(List<Item> items)
    {
        foreach (var item in items)
        {
            if (item.IsValid())
            {
                if (item.RequiresProcessing())
                {
                    ProcessItem(item);
                }
            }
        }
    }
}
```

### Naming Conventions (Microsoft C# Standards)

```csharp
// ✅ PascalCase for classes, methods, properties
public class CustomerService
{
    public string FirstName { get; set; }
    
    public void ProcessOrder() { }
}

// ✅ camelCase for local variables and parameters
public void CalculateTotal(decimal itemPrice)
{
    decimal salesTax = 0.08m;
    decimal totalAmount = itemPrice * (1 + salesTax);
}

// ✅ _camelCase for private fields
public class OrderProcessor
{
    private readonly ILogger _logger;
    private readonly decimal _taxRate;
}

// ✅ UPPERCASE for constants
public class Constants
{
    public const int MAX_LOGIN_ATTEMPTS = 3;
    public const string DEFAULT_CULTURE = "en-US";
}

// ✅ Interfaces start with 'I'
public interface ICustomerRepository { }

// ✅ Async methods end with 'Async'
public async Task<Customer> GetCustomerAsync(int id)
{
    return await _repository.GetByIdAsync(id);
}
```

**Source**: Microsoft C# Coding Conventions

---

## Error Handling

### Use Exceptions, Not Return Codes

```csharp
// ❌ Bad - Return codes
public int DeleteUser(int userId)
{
    if (userId <= 0)
        return ERROR_INVALID_ID;
    
    var user = _db.Users.Find(userId);
    if (user == null)
        return ERROR_NOT_FOUND;
    
    _db.Users.Remove(user);
    _db.SaveChanges();
    return SUCCESS;
}

// Caller must check codes
var result = DeleteUser(123);
if (result == ERROR_INVALID_ID)
{
    // Handle error
}
else if (result == ERROR_NOT_FOUND)
{
    // Handle error
}

// ✅ Good - Exceptions
public void DeleteUser(int userId)
{
    if (userId <= 0)
        throw new ArgumentException("User ID must be positive", nameof(userId));
    
    var user = _db.Users.Find(userId);
    if (user == null)
        throw new NotFoundException($"User with ID {userId} not found");
    
    _db.Users.Remove(user);
    _db.SaveChanges();
}

// Clean caller code
try
{
    DeleteUser(123);
}
catch (ArgumentException ex)
{
    // Handle invalid input
}
catch (NotFoundException ex)
{
    // Handle not found
}
```

### Create Informative Error Messages

```csharp
// ❌ Bad - Vague exceptions
throw new Exception("Error");
throw new Exception("Something went wrong");

// ✅ Good - Specific and informative
throw new InvalidOperationException(
    $"Cannot process order {orderId} because it has already been shipped");

throw new ArgumentOutOfRangeException(
    nameof(quantity),
    quantity,
    "Quantity must be between 1 and 1000");
```

### Use Custom Exceptions

```csharp
// Custom exceptions for domain-specific errors
public class InsufficientStockException : Exception
{
    public int ProductId { get; }
    public int RequestedQuantity { get; }
    public int AvailableQuantity { get; }

    public InsufficientStockException(
        int productId,
        int requested,
        int available)
        : base($"Insufficient stock for product {productId}. " +
               $"Requested: {requested}, Available: {available}")
    {
        ProductId = productId;
        RequestedQuantity = requested;
        AvailableQuantity = available;
    }
}

// Usage
if (availableStock < requestedQuantity)
{
    throw new InsufficientStockException(
        productId,
        requestedQuantity,
        availableStock);
}
```

### Don't Return Null

```csharp
// ❌ Bad - Returns null
public Customer GetCustomer(int id)
{
    var customer = _db.Customers.Find(id);
    return customer; // Could be null!
}

// Caller must always check
var customer = GetCustomer(123);
if (customer != null) // Easy to forget!
{
    ProcessCustomer(customer);
}

// ✅ Good - Option 1: Throw exception
public Customer GetCustomer(int id)
{
    var customer = _db.Customers.Find(id);
    if (customer == null)
        throw new NotFoundException($"Customer {id} not found");
    
    return customer;
}

// ✅ Good - Option 2: Return Maybe/Option type
public Maybe<Customer> FindCustomer(int id)
{
    var customer = _db.Customers.Find(id);
    return Maybe<Customer>.From(customer);
}

// ✅ Good - Option 3: Use Try pattern
public bool TryGetCustomer(int id, out Customer customer)
{
    customer = _db.Customers.Find(id);
    return customer != null;
}

// Usage
if (TryGetCustomer(123, out var customer))
{
    ProcessCustomer(customer);
}
```

### Don't Pass Null

```csharp
// ❌ Bad - Accepts null
public void ProcessOrder(Order order)
{
    if (order != null) // Defensive check everywhere
    {
        // Process
    }
}

// ✅ Good - Validate at boundaries
public void ProcessOrder(Order order)
{
    if (order == null)
        throw new ArgumentNullException(nameof(order));
    
    // Now we can safely use order without checks
    CalculateTotal(order);
    ValidateOrder(order);
}

// ✅ Even better - Use nullable reference types (C# 8+)
#nullable enable

public void ProcessOrder(Order order) // order is non-nullable
{
    // Compiler enforces non-null
    CalculateTotal(order);
}

public void FindOrder(int? orderId) // orderId is explicitly nullable
{
    if (orderId.HasValue)
    {
        // Handle
    }
}
```

**Source**: Clean Code by Robert C. Martin, Chapter 7

---

## Classes

### Single Responsibility Principle

A class should have one reason to change.

```csharp
// ❌ Bad - Multiple responsibilities
public class User
{
    public int Id { get; set; }
    public string Username { get; set; }
    public string Email { get; set; }
    
    // Database access
    public void Save()
    {
        using var connection = new SqlConnection(_connectionString);
        // SQL code
    }
    
    // Email sending
    public void SendWelcomeEmail()
    {
        var smtp = new SmtpClient();
        // Email sending code
    }
    
    // Password hashing
    public void SetPassword(string password)
    {
        // Hashing logic
    }
    
    // Validation
    public bool IsValid()
    {
        return !string.IsNullOrEmpty(Username) &&
               Email.Contains("@");
    }
}

// ✅ Good - Separated responsibilities
public class User
{
    public int Id { get; set; }
    public string Username { get; set; }
    public string Email { get; set; }
    public string PasswordHash { get; set; }
}

public class UserRepository
{
    public void Save(User user)
    {
        // Database code
    }
    
    public User GetById(int id)
    {
        // Database code
    }
}

public class UserEmailService
{
    public void SendWelcomeEmail(User user)
    {
        // Email code
    }
}

public class PasswordService
{
    public string HashPassword(string password)
    {
        // Hashing logic
    }
    
    public bool VerifyPassword(string password, string hash)
    {
        // Verification logic
    }
}

public class UserValidator
{
    public ValidationResult Validate(User user)
    {
        // Validation logic
    }
}
```

### Small Classes

Classes should be small and focused.

```csharp
// ✅ Good - Small, focused class
public class EmailAddress
{
    private readonly string _value;

    public EmailAddress(string email)
    {
        if (string.IsNullOrWhiteSpace(email))
            throw new ArgumentException("Email cannot be empty");
        
        if (!email.Contains("@"))
            throw new ArgumentException("Invalid email format");
        
        _value = email;
    }

    public string Value => _value;
    
    public string Domain => _value.Split('@')[1];
    
    public override string ToString() => _value;
}

// ✅ Good - Single-purpose class
public class TaxCalculator
{
    private readonly decimal _taxRate;

    public TaxCalculator(decimal taxRate)
    {
        _taxRate = taxRate;
    }

    public decimal CalculateTax(decimal amount)
    {
        return amount * _taxRate;
    }

    public decimal CalculateTotal(decimal amount)
    {
        return amount + CalculateTax(amount);
    }
}
```

### Cohesion

Classes should have high cohesion - methods and variables should be related.

```csharp
// ✅ High cohesion - all methods use most of the fields
public class BankAccount
{
    private decimal _balance;
    private readonly string _accountNumber;
    private readonly string _owner;

    public BankAccount(string accountNumber, string owner)
    {
        _accountNumber = accountNumber;
        _owner = owner;
        _balance = 0;
    }

    public void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Amount must be positive");
        
        _balance += amount;
    }

    public void Withdraw(decimal amount)
    {
        if (amount > _balance)
            throw new InvalidOperationException("Insufficient funds");
        
        _balance -= amount;
    }

    public decimal GetBalance() => _balance;
    
    public string GetAccountInfo() => $"{_owner} - {_accountNumber}";
}
```

**Source**: Clean Code by Robert C. Martin, Chapter 10

---

## Code Smells

### Long Method

```csharp
// ❌ Smell: Method is too long
public void ProcessCustomerOrder() { /* 100+ lines */ }

// ✅ Fix: Extract smaller methods
public void ProcessCustomerOrder()
{
    ValidateCustomer();
    CalculateOrderTotal();
    ApplyDiscounts();
    ProcessPayment();
    SendConfirmation();
}
```

### Duplicate Code

```csharp
// ❌ Smell: Same code in multiple places
public class ReportGenerator
{
    public void GenerateSalesReport()
    {
        var header = "Company Name\nSales Report\n" + DateTime.Now;
        // Use header
    }
    
    public void GenerateInventoryReport()
    {
        var header = "Company Name\nInventory Report\n" + DateTime.Now;
        // Use header
    }
}

// ✅ Fix: Extract method
public class ReportGenerator
{
    private string GenerateHeader(string reportType)
    {
        return $"Company Name\n{reportType}\n{DateTime.Now}";
    }
    
    public void GenerateSalesReport()
    {
        var header = GenerateHeader("Sales Report");
        // Use header
    }
    
    public void GenerateInventoryReport()
    {
        var header = GenerateHeader("Inventory Report");
        // Use header
    }
}
```

### Large Class (God Object)

```csharp
// ❌ Smell: Class does everything
public class OrderManager
{
    // Handles orders, customers, inventory, shipping, billing, etc.
    // 50+ methods, 1000+ lines
}

// ✅ Fix: Split into focused classes
public class OrderService { }
public class CustomerService { }
public class InventoryService { }
public class ShippingService { }
public class BillingService { }
```

### Long Parameter List

```csharp
// ❌ Smell: Too many parameters
public void CreateUser(
    string firstName,
    string lastName,
    string email,
    string phone,
    string address,
    string city,
    string state,
    string zip)
{ }

// ✅ Fix: Use parameter object
public void CreateUser(UserRegistration registration) { }

public class UserRegistration
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public ContactInfo Contact { get; set; }
    public Address Address { get; set; }
}
```

### Magic Numbers

```csharp
// ❌ Smell: What do these numbers mean?
if (age > 65)
{
    discount = price * 0.15;
}

// ✅ Fix: Named constants
private const int SENIOR_AGE_THRESHOLD = 65;
private const decimal SENIOR_DISCOUNT_RATE = 0.15m;

if (age > SENIOR_AGE_THRESHOLD)
{
    discount = price * SENIOR_DISCOUNT_RATE;
}
```

### Dead Code

```csharp
// ❌ Smell: Unused code
public class UserService
{
    public void ProcessUser() { }
    
    // This method is never called
    public void OldProcessUser() { }
    
    // Commented code
    // public void SomeOldFeature() { }
}

// ✅ Fix: Delete it (version control keeps history)
public class UserService
{
    public void ProcessUser() { }
}
```

**Source**: "Refactoring: Improving the Design of Existing Code" by Martin Fowler

---

## Refactoring Examples

### Example 1: Extract Method

```csharp
// ❌ Before
public void PrintOwing()
{
    PrintBanner();
    
    // Print details
    Console.WriteLine($"Name: {_name}");
    Console.WriteLine($"Amount: {GetOutstanding()}");
}

// ✅ After
public void PrintOwing()
{
    PrintBanner();
    PrintDetails(GetOutstanding());
}

private void PrintDetails(decimal outstanding)
{
    Console.WriteLine($"Name: {_name}");
    Console.WriteLine($"Amount: {outstanding}");
}
```

### Example 2: Replace Conditional with Polymorphism

```csharp
// ❌ Before
public decimal GetSpeed(VehicleType type)
{
    switch (type)
    {
        case VehicleType.Car:
            return 120;
        case VehicleType.Truck:
            return 80;
        case VehicleType.Motorcycle:
            return 150;
        default:
            throw new ArgumentException();
    }
}

// ✅ After
public abstract class Vehicle
{
    public abstract decimal GetSpeed();
}

public class Car : Vehicle
{
    public override decimal GetSpeed() => 120;
}

public class Truck : Vehicle
{
    public override decimal GetSpeed() => 80;
}

public class Motorcycle : Vehicle
{
    public override decimal GetSpeed() => 150;
}
```

### Example 3: Replace Magic Number with Constant

```csharp
// ❌ Before
public decimal CalculateCircumference(decimal radius)
{
    return 2 * 3.14159 * radius;
}

// ✅ After
private const decimal PI = 3.14159m;

public decimal CalculateCircumference(decimal radius)
{
    return 2 * PI * radius;
}

// ✅ Even better - use built-in constant
public decimal CalculateCircumference(decimal radius)
{
    return 2 * (decimal)Math.PI * radius;
}
```

### Example 4: Introduce Explaining Variable

```csharp
// ❌ Before
if ((platform.ToUpper().Contains("MAC") ||
     platform.ToUpper().Contains("WIN")) &&
    browser.ToUpper().Contains("IE") &&
    wasInitialized() && resize > 0)
{
    // Do something
}

// ✅ After
bool isSupportedPlatform = platform.ToUpper().Contains("MAC") ||
                           platform.ToUpper().Contains("WIN");
bool isIEBrowser = browser.ToUpper().Contains("IE");
bool wasResized = resize > 0;

if (isSupportedPlatform && isIEBrowser && wasInitialized() && wasResized)
{
    // Do something
}
```

### Example 5: Replace Temp with Query

```csharp
// ❌ Before
public decimal CalculateTotal()
{
    decimal basePrice = _quantity * _itemPrice;
    decimal discountFactor = _quantity > 100 ? 0.95m : 0.98m;
    return basePrice * discountFactor;
}

// ✅ After
public decimal CalculateTotal()
{
    return BasePrice() * DiscountFactor();
}

private decimal BasePrice()
{
    return _quantity * _itemPrice;
}

private decimal DiscountFactor()
{
    return _quantity > 100 ? 0.95m : 0.98m;
}
```

---

## Clean Code Checklist

### Naming
- ✅ Names reveal intent
- ✅ Names are pronounceable and searchable
- ✅ Classes are nouns, methods are verbs
- ✅ Consistent terminology throughout codebase

### Functions
- ✅ Functions are small (< 20 lines ideally)
- ✅ Functions do one thing
- ✅ One level of abstraction per function
- ✅ Few arguments (0-2 ideally, max 3)
- ✅ No side effects
- ✅ Command-query separation

### Comments
- ✅ Code is self-documenting
- ✅ Comments explain "why", not "what"
- ✅ No commented-out code
- ✅ No redundant comments

### Formatting
- ✅ Consistent indentation
- ✅ Related code is grouped together
- ✅ Vertical spacing between concepts
- ✅ Following team/company style guide

### Error Handling
- ✅ Use exceptions, not return codes
- ✅ Informative error messages
- ✅ Don't return null
- ✅ Don't pass null
- ✅ Custom exceptions for domain errors

### Classes
- ✅ Single responsibility
- ✅ Small and focused
- ✅ High cohesion
- ✅ Low coupling

### General
- ✅ No duplicate code (DRY)
- ✅ No magic numbers
- ✅ No dead code
- ✅ Code is testable
- ✅ Dependencies point inward

---

## Key Principles Summary

| Principle | Description |
|-----------|-------------|
| **DRY** | Don't Repeat Yourself |
| **KISS** | Keep It Simple, Stupid |
| **YAGNI** | You Aren't Gonna Need It |
| **SRP** | Single Responsibility Principle |
| **Boy Scout Rule** | Leave code cleaner than you found it |
| **Premature Optimization** | Focus on clarity first, optimize later |

---

## Further Reading

- **"Clean Code"** by Robert C. Martin - The definitive guide
- **"The Pragmatic Programmer"** by Hunt & Thomas
- **"Code Complete"** by Steve McConnell
- **"Refactoring"** by Martin Fowler
- **Microsoft C# Coding Conventions**: https://docs.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions
- **".NET Framework Design Guidelines"** by Cwalina & Abrams

---

> "Any fool can write code that a computer can understand. Good programmers write code that humans can understand."
> — Martin Fowler

> "Truth can only be found in one place: the code."
> — Robert C. Martin
