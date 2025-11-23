# RESTful API Design Best Practices

A comprehensive guide to designing robust and maintainable RESTful APIs with C# examples.

**Sources**: 
- Roy Fielding's REST dissertation (https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)
- Microsoft REST API Guidelines (https://github.com/microsoft/api-guidelines)
- RESTful Web Services by Leonard Richardson & Sam Ruby

---

## Table of Contents
1. [REST Principles](#rest-principles)
2. [HTTP Methods](#http-methods)
3. [Resource Naming](#resource-naming)
4. [Status Codes](#status-codes)
5. [Request/Response Examples](#requestresponse-examples)
6. [Versioning](#versioning)
7. [Pagination](#pagination)
8. [Filtering & Sorting](#filtering--sorting)
9. [Error Handling](#error-handling)
10. [Security](#security)

---

## REST Principles

**RE**presentational **S**tate **T**ransfer is an architectural style with six constraints:

1. **Client-Server**: Separation of concerns
2. **Stateless**: Each request contains all information needed
3. **Cacheable**: Responses must define themselves as cacheable or not
4. **Uniform Interface**: Standardized way to interact with resources
5. **Layered System**: Client doesn't know if connected directly to server
6. **Code on Demand** (optional): Server can extend client functionality

---

## HTTP Methods

| Method | Purpose | Idempotent | Safe | Request Body | Response Body |
|--------|---------|------------|------|--------------|---------------|
| **GET** | Retrieve resource(s) | ✅ | ✅ | ❌ | ✅ |
| **POST** | Create new resource | ❌ | ❌ | ✅ | ✅ |
| **PUT** | Update/Replace entire resource | ✅ | ❌ | ✅ | ✅ |
| **PATCH** | Partially update resource | ❌ | ❌ | ✅ | ✅ |
| **DELETE** | Remove resource | ✅ | ❌ | ❌ | ✅/❌ |

**Source**: RFC 7231 - HTTP/1.1 Semantics and Content

---

## Resource Naming

### ✅ Best Practices

```
# Use nouns, not verbs
GET    /api/users              ✅
GET    /api/getUsers           ❌

# Use plural nouns for collections
GET    /api/products           ✅
GET    /api/product            ❌

# Use hierarchical structure for relationships
GET    /api/users/123/orders   ✅
GET    /api/orders?userId=123  ⚠️ (acceptable alternative)

# Use lowercase and hyphens
GET    /api/product-categories ✅
GET    /api/ProductCategories  ❌
GET    /api/product_categories ❌

# Avoid file extensions
GET    /api/users/123          ✅
GET    /api/users/123.json     ❌
```

### C# ASP.NET Core Example

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;

    public UsersController(IUserService userService)
    {
        _userService = userService;
    }

    // GET: api/users
    [HttpGet]
    public async Task<ActionResult<IEnumerable<UserDto>>> GetUsers()
    {
        var users = await _userService.GetAllUsersAsync();
        return Ok(users);
    }

    // GET: api/users/5
    [HttpGet("{id}")]
    public async Task<ActionResult<UserDto>> GetUser(int id)
    {
        var user = await _userService.GetUserByIdAsync(id);
        
        if (user == null)
            return NotFound();
            
        return Ok(user);
    }

    // GET: api/users/5/orders
    [HttpGet("{userId}/orders")]
    public async Task<ActionResult<IEnumerable<OrderDto>>> GetUserOrders(int userId)
    {
        var orders = await _userService.GetUserOrdersAsync(userId);
        return Ok(orders);
    }

    // POST: api/users
    [HttpPost]
    public async Task<ActionResult<UserDto>> CreateUser(CreateUserRequest request)
    {
        var user = await _userService.CreateUserAsync(request);
        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }

    // PUT: api/users/5
    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateUser(int id, UpdateUserRequest request)
    {
        if (id != request.Id)
            return BadRequest();

        var updated = await _userService.UpdateUserAsync(request);
        
        if (!updated)
            return NotFound();
            
        return NoContent();
    }

    // PATCH: api/users/5
    [HttpPatch("{id}")]
    public async Task<IActionResult> PatchUser(int id, JsonPatchDocument<UserDto> patchDoc)
    {
        var user = await _userService.GetUserByIdAsync(id);
        
        if (user == null)
            return NotFound();

        patchDoc.ApplyTo(user, ModelState);
        
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        await _userService.SaveChangesAsync();
        return NoContent();
    }

    // DELETE: api/users/5
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteUser(int id)
    {
        var deleted = await _userService.DeleteUserAsync(id);
        
        if (!deleted)
            return NotFound();
            
        return NoContent();
    }
}
```

---

## Status Codes

### Common HTTP Status Codes

| Code | Name | Use Case |
|------|------|----------|
| **2xx Success** | | |
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE, PUT with no response body |
| **3xx Redirection** | | |
| 301 | Moved Permanently | Resource moved to new URL |
| 304 | Not Modified | Cached response is still valid |
| **4xx Client Errors** | | |
| 400 | Bad Request | Invalid request format/data |
| 401 | Unauthorized | Authentication required |
| 403 | Forbidden | User lacks permission |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Request conflicts with current state |
| 422 | Unprocessable Entity | Validation errors |
| 429 | Too Many Requests | Rate limit exceeded |
| **5xx Server Errors** | | |
| 500 | Internal Server Error | Unexpected server error |
| 502 | Bad Gateway | Invalid response from upstream |
| 503 | Service Unavailable | Server temporarily unavailable |

**Source**: RFC 7231, RFC 6585

### C# Implementation

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;

    public ProductsController(IProductService productService)
    {
        _productService = productService;
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<ProductDto>> GetProduct(int id)
    {
        try
        {
            var product = await _productService.GetByIdAsync(id);
            
            if (product == null)
                return NotFound(new { message = $"Product with ID {id} not found" });
            
            return Ok(product);
        }
        catch (Exception ex)
        {
            // Log the exception
            return StatusCode(500, new { message = "An error occurred while processing your request" });
        }
    }

    [HttpPost]
    public async Task<ActionResult<ProductDto>> CreateProduct(CreateProductRequest request)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        try
        {
            var product = await _productService.CreateAsync(request);
            return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
        }
        catch (InvalidOperationException ex)
        {
            return Conflict(new { message = ex.Message });
        }
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteProduct(int id)
    {
        var deleted = await _productService.DeleteAsync(id);
        
        if (!deleted)
            return NotFound();
        
        return NoContent();
    }
}
```

---

## Request/Response Examples

### GET Request - Retrieve Collection

**Request:**
```http
GET /api/products?category=electronics&inStock=true HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response:**
```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=3600

{
  "data": [
    {
      "id": 1,
      "name": "Laptop",
      "price": 999.99,
      "category": "electronics",
      "inStock": true
    },
    {
      "id": 2,
      "name": "Mouse",
      "price": 29.99,
      "category": "electronics",
      "inStock": true
    }
  ],
  "meta": {
    "total": 2,
    "page": 1,
    "pageSize": 20
  }
}
```

### POST Request - Create Resource

**Request:**
```http
POST /api/products HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

{
  "name": "Wireless Keyboard",
  "price": 79.99,
  "category": "electronics",
  "inStock": true
}
```

**Response:**
```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/products/3

{
  "id": 3,
  "name": "Wireless Keyboard",
  "price": 79.99,
  "category": "electronics",
  "inStock": true,
  "createdAt": "2025-11-23T10:30:00Z"
}
```

### C# Models

```csharp
// DTOs (Data Transfer Objects)
public class ProductDto
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public string Category { get; set; }
    public bool InStock { get; set; }
    public DateTime CreatedAt { get; set; }
}

public class CreateProductRequest
{
    [Required]
    [StringLength(100, MinimumLength = 3)]
    public string Name { get; set; }

    [Required]
    [Range(0.01, 999999.99)]
    public decimal Price { get; set; }

    [Required]
    public string Category { get; set; }

    public bool InStock { get; set; } = true;
}

public class ApiResponse<T>
{
    public T Data { get; set; }
    public MetaData Meta { get; set; }
}

public class MetaData
{
    public int Total { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
}
```

---

## Versioning

### Three Common Approaches

#### 1. URI Versioning (Recommended)

```csharp
// Version 1
[ApiController]
[Route("api/v1/[controller]")]
public class UsersV1Controller : ControllerBase
{
    [HttpGet("{id}")]
    public ActionResult<UserV1Dto> GetUser(int id)
    {
        // Return V1 format
        return Ok(new UserV1Dto { Id = id, Name = "John Doe" });
    }
}

// Version 2
[ApiController]
[Route("api/v2/[controller]")]
public class UsersV2Controller : ControllerBase
{
    [HttpGet("{id}")]
    public ActionResult<UserV2Dto> GetUser(int id)
    {
        // Return V2 format with additional fields
        return Ok(new UserV2Dto 
        { 
            Id = id, 
            FirstName = "John", 
            LastName = "Doe",
            Email = "john@example.com"
        });
    }
}
```

#### 2. Header Versioning

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    [ApiVersion("1.0")]
    public ActionResult<UserV1Dto> GetUserV1(int id)
    {
        return Ok(new UserV1Dto { Id = id, Name = "John Doe" });
    }

    [HttpGet("{id}")]
    [ApiVersion("2.0")]
    public ActionResult<UserV2Dto> GetUserV2(int id)
    {
        return Ok(new UserV2Dto 
        { 
            Id = id, 
            FirstName = "John", 
            LastName = "Doe" 
        });
    }
}

// Request header: api-version: 2.0
```

#### 3. Query Parameter Versioning

```
GET /api/users/123?version=2
```

**Source**: Microsoft API Versioning Guidelines

---

## Pagination

### Offset-Based Pagination

```csharp
[HttpGet]
public async Task<ActionResult<PagedResponse<ProductDto>>> GetProducts(
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 20)
{
    if (page < 1 || pageSize < 1 || pageSize > 100)
        return BadRequest("Invalid pagination parameters");

    var totalCount = await _productService.GetTotalCountAsync();
    var products = await _productService.GetPagedAsync(page, pageSize);

    var response = new PagedResponse<ProductDto>
    {
        Data = products,
        Page = page,
        PageSize = pageSize,
        TotalCount = totalCount,
        TotalPages = (int)Math.Ceiling(totalCount / (double)pageSize)
    };

    return Ok(response);
}

public class PagedResponse<T>
{
    public IEnumerable<T> Data { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalCount { get; set; }
    public int TotalPages { get; set; }
    public bool HasPrevious => Page > 1;
    public bool HasNext => Page < TotalPages;
}
```

**Response with Links (HATEOAS):**

```json
{
  "data": [...],
  "page": 2,
  "pageSize": 20,
  "totalCount": 150,
  "totalPages": 8,
  "links": {
    "self": "/api/products?page=2&pageSize=20",
    "first": "/api/products?page=1&pageSize=20",
    "previous": "/api/products?page=1&pageSize=20",
    "next": "/api/products?page=3&pageSize=20",
    "last": "/api/products?page=8&pageSize=20"
  }
}
```

### Cursor-Based Pagination (For Large Datasets)

```csharp
[HttpGet]
public async Task<ActionResult<CursorPagedResponse<ProductDto>>> GetProducts(
    [FromQuery] string cursor = null,
    [FromQuery] int limit = 20)
{
    var result = await _productService.GetPagedByCursorAsync(cursor, limit);

    return Ok(new CursorPagedResponse<ProductDto>
    {
        Data = result.Items,
        NextCursor = result.NextCursor,
        HasMore = result.HasMore
    });
}

public class CursorPagedResponse<T>
{
    public IEnumerable<T> Data { get; set; }
    public string NextCursor { get; set; }
    public bool HasMore { get; set; }
}
```

**Source**: GraphQL Cursor Connections Specification

---

## Filtering & Sorting

### Query Parameters

```csharp
[HttpGet]
public async Task<ActionResult<IEnumerable<ProductDto>>> GetProducts(
    [FromQuery] string category = null,
    [FromQuery] decimal? minPrice = null,
    [FromQuery] decimal? maxPrice = null,
    [FromQuery] bool? inStock = null,
    [FromQuery] string sortBy = "name",
    [FromQuery] string sortOrder = "asc")
{
    var query = _productService.GetQueryable();

    // Filtering
    if (!string.IsNullOrEmpty(category))
        query = query.Where(p => p.Category == category);

    if (minPrice.HasValue)
        query = query.Where(p => p.Price >= minPrice.Value);

    if (maxPrice.HasValue)
        query = query.Where(p => p.Price <= maxPrice.Value);

    if (inStock.HasValue)
        query = query.Where(p => p.InStock == inStock.Value);

    // Sorting
    query = sortBy.ToLower() switch
    {
        "price" => sortOrder == "desc" 
            ? query.OrderByDescending(p => p.Price) 
            : query.OrderBy(p => p.Price),
        "name" => sortOrder == "desc" 
            ? query.OrderByDescending(p => p.Name) 
            : query.OrderBy(p => p.Name),
        _ => query.OrderBy(p => p.Name)
    };

    var products = await query.ToListAsync();
    return Ok(products);
}
```

**Example Requests:**
```
GET /api/products?category=electronics&minPrice=50&maxPrice=200&sortBy=price&sortOrder=desc
GET /api/products?inStock=true&sortBy=name
```

---

## Error Handling

### Standardized Error Response

```csharp
public class ErrorResponse
{
    public string Type { get; set; }
    public string Title { get; set; }
    public int Status { get; set; }
    public string Detail { get; set; }
    public string Instance { get; set; }
    public Dictionary<string, string[]> Errors { get; set; }
}

// Global exception handler
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(exception, "An error occurred");

        var errorResponse = new ErrorResponse
        {
            Type = "https://tools.ietf.org/html/rfc7231#section-6.5.1",
            Title = "An error occurred",
            Status = 500,
            Instance = httpContext.Request.Path
        };

        switch (exception)
        {
            case ValidationException validationException:
                errorResponse.Status = 400;
                errorResponse.Title = "Validation Error";
                errorResponse.Detail = validationException.Message;
                errorResponse.Errors = validationException.Errors;
                break;

            case NotFoundException notFoundException:
                errorResponse.Status = 404;
                errorResponse.Title = "Resource Not Found";
                errorResponse.Detail = notFoundException.Message;
                break;

            case UnauthorizedAccessException:
                errorResponse.Status = 401;
                errorResponse.Title = "Unauthorized";
                errorResponse.Detail = "Authentication is required";
                break;

            default:
                errorResponse.Detail = "An unexpected error occurred";
                break;
        }

        httpContext.Response.StatusCode = errorResponse.Status;
        await httpContext.Response.WriteAsJsonAsync(errorResponse, cancellationToken);
        
        return true;
    }
}
```

**Error Response Example:**

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "Validation Error",
  "status": 400,
  "detail": "One or more validation errors occurred",
  "instance": "/api/products",
  "errors": {
    "Name": ["The Name field is required"],
    "Price": ["The Price must be greater than 0"]
  }
}
```

**Source**: RFC 7807 - Problem Details for HTTP APIs

---

## Security

### 1. Authentication & Authorization

```csharp
[ApiController]
[Route("api/[controller]")]
[Authorize] // Requires authentication for all endpoints
public class ProductsController : ControllerBase
{
    [HttpGet]
    [AllowAnonymous] // Public endpoint
    public ActionResult<IEnumerable<ProductDto>> GetProducts()
    {
        // Anyone can view products
        return Ok(_products);
    }

    [HttpPost]
    [Authorize(Roles = "Admin")] // Only admins can create
    public ActionResult<ProductDto> CreateProduct(CreateProductRequest request)
    {
        // Create product
        return Created();
    }

    [HttpDelete("{id}")]
    [Authorize(Policy = "RequireAdminRole")] // Policy-based authorization
    public IActionResult DeleteProduct(int id)
    {
        // Delete product
        return NoContent();
    }
}
```

### 2. HTTPS Only

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHttpsRedirection(options =>
{
    options.RedirectStatusCode = StatusCodes.Status307TemporaryRedirect;
    options.HttpsPort = 443;
});

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseHsts(); // HTTP Strict Transport Security
}

app.UseHttpsRedirection();
```

### 3. Rate Limiting

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", config =>
    {
        config.PermitLimit = 100;
        config.Window = TimeSpan.FromMinutes(1);
        config.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        config.QueueLimit = 0;
    });
});

app.UseRateLimiter();

// Apply to controller
[EnableRateLimiting("fixed")]
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // ...
}
```

### 4. CORS Configuration

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowSpecificOrigin",
        policy =>
        {
            policy.WithOrigins("https://example.com")
                  .AllowAnyHeader()
                  .AllowAnyMethod()
                  .AllowCredentials();
        });
});

app.UseCors("AllowSpecificOrigin");
```

### 5. Input Validation

```csharp
public class CreateProductRequest
{
    [Required(ErrorMessage = "Name is required")]
    [StringLength(100, MinimumLength = 3, ErrorMessage = "Name must be between 3 and 100 characters")]
    public string Name { get; set; }

    [Required]
    [Range(0.01, 999999.99, ErrorMessage = "Price must be between 0.01 and 999999.99")]
    public decimal Price { get; set; }

    [Required]
    [RegularExpression(@"^[a-zA-Z0-9-]+$", ErrorMessage = "Category contains invalid characters")]
    public string Category { get; set; }

    [Url(ErrorMessage = "Invalid URL format")]
    public string ImageUrl { get; set; }
}
```

**Sources**: 
- OWASP API Security Top 10 (https://owasp.org/www-project-api-security/)
- Microsoft Security Best Practices

---

## Complete Example: Products API

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Authorization;
using System.ComponentModel.DataAnnotations;

namespace ProductsApi.Controllers
{
    [ApiController]
    [Route("api/v1/[controller]")]
    [Authorize]
    public class ProductsController : ControllerBase
    {
        private readonly IProductService _productService;
        private readonly ILogger<ProductsController> _logger;

        public ProductsController(
            IProductService productService,
            ILogger<ProductsController> logger)
        {
            _productService = productService;
            _logger = logger;
        }

        /// <summary>
        /// Get all products with optional filtering and pagination
        /// </summary>
        [HttpGet]
        [AllowAnonymous]
        [ProducesResponseType(typeof(PagedResponse<ProductDto>), StatusCodes.Status200OK)]
        public async Task<ActionResult<PagedResponse<ProductDto>>> GetProducts(
            [FromQuery] ProductFilterRequest filter,
            [FromQuery] int page = 1,
            [FromQuery] int pageSize = 20)
        {
            _logger.LogInformation("Getting products. Page: {Page}, PageSize: {PageSize}", page, pageSize);

            var result = await _productService.GetPagedProductsAsync(filter, page, pageSize);
            return Ok(result);
        }

        /// <summary>
        /// Get a specific product by ID
        /// </summary>
        [HttpGet("{id}")]
        [AllowAnonymous]
        [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
        [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status404NotFound)]
        public async Task<ActionResult<ProductDto>> GetProduct(int id)
        {
            _logger.LogInformation("Getting product with ID: {ProductId}", id);

            var product = await _productService.GetByIdAsync(id);

            if (product == null)
            {
                return NotFound(new ErrorResponse
                {
                    Status = 404,
                    Title = "Product not found",
                    Detail = $"Product with ID {id} was not found"
                });
            }

            return Ok(product);
        }

        /// <summary>
        /// Create a new product
        /// </summary>
        [HttpPost]
        [Authorize(Roles = "Admin")]
        [ProducesResponseType(typeof(ProductDto), StatusCodes.Status201Created)]
        [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status400BadRequest)]
        public async Task<ActionResult<ProductDto>> CreateProduct(
            [FromBody] CreateProductRequest request)
        {
            _logger.LogInformation("Creating new product: {ProductName}", request.Name);

            var product = await _productService.CreateAsync(request);

            return CreatedAtAction(
                nameof(GetProduct),
                new { id = product.Id },
                product);
        }

        /// <summary>
        /// Update an existing product
        /// </summary>
        [HttpPut("{id}")]
        [Authorize(Roles = "Admin")]
        [ProducesResponseType(StatusCodes.Status204NoContent)]
        [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status404NotFound)]
        [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status400BadRequest)]
        public async Task<IActionResult> UpdateProduct(
            int id,
            [FromBody] UpdateProductRequest request)
        {
            if (id != request.Id)
            {
                return BadRequest(new ErrorResponse
                {
                    Status = 400,
                    Title = "Invalid request",
                    Detail = "ID in URL does not match ID in body"
                });
            }

            _logger.LogInformation("Updating product with ID: {ProductId}", id);

            var updated = await _productService.UpdateAsync(request);

            if (!updated)
                return NotFound();

            return NoContent();
        }

        /// <summary>
        /// Delete a product
        /// </summary>
        [HttpDelete("{id}")]
        [Authorize(Roles = "Admin")]
        [ProducesResponseType(StatusCodes.Status204NoContent)]
        [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status404NotFound)]
        public async Task<IActionResult> DeleteProduct(int id)
        {
            _logger.LogInformation("Deleting product with ID: {ProductId}", id);

            var deleted = await _productService.DeleteAsync(id);

            if (!deleted)
                return NotFound();

            return NoContent();
        }
    }

    // Supporting classes
    public class ProductFilterRequest
    {
        public string Category { get; set; }
        public decimal? MinPrice { get; set; }
        public decimal? MaxPrice { get; set; }
        public bool? InStock { get; set; }
        public string SortBy { get; set; } = "name";
        public string SortOrder { get; set; } = "asc";
    }

    public class ProductDto
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public decimal Price { get; set; }
        public string Category { get; set; }
        public bool InStock { get; set; }
        public DateTime CreatedAt { get; set; }
        public DateTime? UpdatedAt { get; set; }
    }

    public class CreateProductRequest
    {
        [Required]
        [StringLength(100, MinimumLength = 3)]
        public string Name { get; set; }

        [Required]
        [Range(0.01, 999999.99)]
        public decimal Price { get; set; }

        [Required]
        public string Category { get; set; }

        public bool InStock { get; set; } = true;
    }

    public class UpdateProductRequest
    {
        public int Id { get; set; }

        [Required]
        [StringLength(100, MinimumLength = 3)]
        public string Name { get; set; }

        [Required]
        [Range(0.01, 999999.99)]
        public decimal Price { get; set; }

        [Required]
        public string Category { get; set; }

        public bool InStock { get; set; }
    }
}
```

---

## Quick Reference Checklist

- ✅ Use proper HTTP methods (GET, POST, PUT, PATCH, DELETE)
- ✅ Return appropriate status codes
- ✅ Use nouns for resource names, not verbs
- ✅ Use plural nouns for collections
- ✅ Implement proper error handling with consistent format
- ✅ Version your API (URI, header, or query parameter)
- ✅ Implement pagination for large datasets
- ✅ Support filtering and sorting via query parameters
- ✅ Use HTTPS and implement authentication/authorization
- ✅ Validate all input data
- ✅ Implement rate limiting
- ✅ Document your API (Swagger/OpenAPI)
- ✅ Follow REST constraints (stateless, cacheable, etc.)
- ✅ Use HATEOAS for discoverability (optional but recommended)

---

## Additional Resources

- **Roy Fielding's Dissertation**: https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm
- **Microsoft REST API Guidelines**: https://github.com/microsoft/api-guidelines
- **RFC 7231 - HTTP/1.1**: https://tools.ietf.org/html/rfc7231
- **RFC 7807 - Problem Details**: https://tools.ietf.org/html/rfc7807
- **OWASP API Security**: https://owasp.org/www-project-api-security/
- **OpenAPI Specification**: https://swagger.io/specification/
