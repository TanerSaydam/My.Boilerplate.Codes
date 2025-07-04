# .NET Boilerplate Kodları

Bu documentation, .NET projelerimde sıklıkla kullandığım temel kod parçalarını içermektedir.

## İçindekiler

1. [Entity Base Sınıfı](#entity-base-sınıfı)
2. [SaveChanges Override](#savechanges-override)
3. [Global Query Filters](#global-query-filters)
4. [Password Hashing](#password-hashing)
5. [Validation Behavior](#validation-behavior)
6. [Permission Behavior](#permission-behavior)
7. [Exception Handler](#exception-handler)
8. [DbContext Configuration](#dbcontext-configuration)
9. [OData](#odata)

---

## Entity Base Sınıfı

Tüm entity'lerin türetileceği temel sınıf ve IdentityId value object'i içerir. Soft delete özelliği ve audit alanlarını yönetir.

```csharp
public abstract class Entity
{
    public Entity()
    {
        IdentityId identity = new(Guid.CreateVersion7());
        Id = identity;
    }
    public IdentityId Id { get; private set; }
    public IdentityId CreatedBy { get; private set; } = default!;
    public DateTimeOffset CreatedAt { get; private set; } = default!;
    public IdentityId? UpdatedBy { get; private set; }
    public DateTimeOffset? UpdatedAt { get; private set; }
    public IdentityId? DeletedBy { get; private set; }
    public DateTimeOffset? DeletedAt { get; private set; }
    public bool IsDeleted { get; private set; }

    public void Delete()
    {
        DeletedAt = DateTimeOffset.Now;
        IsDeleted = true;
    }
}

public record IdentityId(Guid Value)
{
    public static implicit operator Guid(IdentityId id) => id.Value;
    public static implicit operator string(IdentityId id) => id.Value.ToString();
}
```

---

## SaveChanges Override

DbContext'te SaveChangesAsync metodunu override ederek audit alanlarının otomatik doldurulmasını sağlar.

```csharp
public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    var entries = ChangeTracker.Entries<Entity>();

    HttpContextAccessor httpContextAccessor = new();
    string userIdString =
	    httpContextAccessor
	    .HttpContext!
	    .User
	    .Claims
	    .First(p => p.Type == ClaimTypes.NameIdentifier)
	    .Value;

    Guid userId = Guid.Parse(userIdString);
    IdentityId identityId = new(userId);

    foreach (var entry in entries)
    {
        if (entry.State == EntityState.Added)
        {
            entry.Property(p => p.CreatedAt)
                .CurrentValue = DateTimeOffset.Now;
            entry.Property(p => p.CreatedBy)
                .CurrentValue = identityId;
        }

        if (entry.State == EntityState.Modified)
        {
            if (entry.Property(p => p.IsDeleted).CurrentValue == true)
            {
                entry.Property(p => p.DeletedAt)
                .CurrentValue = DateTimeOffset.Now;
                entry.Property(p => p.DeletedBy)
                .CurrentValue = identityId;
            }
            else
            {
                entry.Property(p => p.UpdatedAt)
                    .CurrentValue = DateTimeOffset.Now;
                entry.Property(p => p.UpdatedBy)
                .CurrentValue = identityId;
            }
        }

        if (entry.State == EntityState.Deleted)
        {
            throw new ArgumentException("Db'den direkt silme işlemi yapamazsınız");
        }
    }

    return base.SaveChangesAsync(cancellationToken);
}
```

---

## Global Query Filters

Soft delete özelliğine sahip tüm entity'lere otomatik olarak query filter uygular.

```csharp
public static class ExtensionMethods
{
    public static void ApplyGlobalFilters(this ModelBuilder modelBuilder)
    {
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            var clrType = entityType.ClrType;

            if (typeof(Entity).IsAssignableFrom(clrType))
            {
                var parameter = Expression.Parameter(clrType, "e");
                var property = Expression.Property(parameter, nameof(IHasSoftDelete.IsDeleted));
                var condition = Expression.Equal(property, Expression.Constant(false));
                var lambda = Expression.Lambda(condition, parameter);

                entityType.SetQueryFilter(lambda);
            }
        }
    }
}
```

Uygulamak için Dbcontext OnModelCreating'de bu metodu çağırıyoruz.
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyGlobalFilters();
    base.OnModelCreating(modelBuilder);
}
```

---

## Password Hashing

Şifreleri güvenli bir şekilde hash'lemek için kullanılan Password value object'i.

```csharp
public sealed record Password
{
    private Password()
    {
    }
    public Password(string password)
    {
        CreatePasswordHash(password);
    }
    public byte[] PasswordHash { get; private set; } = default!;
    public byte[] PasswordSalt { get; private set; } = default!;

    private void CreatePasswordHash(string password)
    {
        using var hmac = new System.Security.Cryptography.HMACSHA512();
        PasswordSalt = hmac.Key;
        PasswordHash = hmac.ComputeHash(System.Text.Encoding.UTF8.GetBytes(password));
    }
}
```

### Password Verification

User tablosunda çağırabilir bir metot olarak tasarlayıp şifre doğrulama için kullanıyorum.

```csharp
public bool VerifyPasswordHash(string password)
{
    using var hmac = new System.Security.Cryptography.HMACSHA512(Password.PasswordSalt);
    var computedHash = hmac.ComputeHash(System.Text.Encoding.UTF8.GetBytes(password));
    return computedHash.SequenceEqual(Password.PasswordHash);
}
```

---

## Validation Behavior

MediatR pipeline'ında FluentValidation ile otomatik doğrulama yapan behavior.

```csharp
public sealed class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken cancellationToken)
    {
        if (!_validators.Any())
        {
            return await next();
        }

        var context = new ValidationContext<TRequest>(request);

        var errorDictionary = _validators
            .Select(s => s.Validate(context))
            .SelectMany(s => s.Errors)
            .Where(s => s != null)
            .GroupBy(
            s => s.PropertyName,
            s => s.ErrorMessage, (propertyName, errorMessage) => new
            {
                Key = propertyName,
                Values = errorMessage.Distinct().ToArray()
            })
            .ToDictionary(s => s.Key, s => s.Values[0]);

        if (errorDictionary.Any())
        {
            var errors = errorDictionary.Select(s => new ValidationFailure
            {
                PropertyName = s.Value,
                ErrorCode = s.Key
            });
            throw new ValidationException(errors);
        }

        return await next();
    }
}
```

---

## Permission Behavior

MediatR pipeline'ında yetki kontrolü yapan behavior.

```csharp
public sealed class PermissionBehavior<TRequest, TResponse>(
    IUserContext userContext,
    IUserRepository userRepository) : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken cancellationToken = default)
    {
        var attr = request.GetType().GetCustomAttribute<PermissionAttribute>(inherit: true);

        if (attr is null) return await next();

        var userId = userContext.GetUserId();
        var user = await userRepository.FirstOrDefaultAsync(p => p.Id == userId, cancellationToken);
        if (user is null)
        {
            throw new ArgumentException("User bulunamadı");
        }

        // Eğer permission string'i varsa kontrol et
        if (!string.IsNullOrEmpty(attr.Permission))
        {
            var hasPermission = user.Permissions.Any(p => p.Name == attr.Permission);
            if (!hasPermission)
            {
                throw new AuthorizationException($"'{attr.Permission}' yetkisine sahip değilsiniz.");
            }
        }

        // Eğer permission string'i yoksa sadece admin kontrolü yap
        else if (!user.IsAdmin.Value)
        {
            throw new AuthorizationException("Bu işlem için admin yetkisi gereklidir.");
        }

        return await next();
    }
}

public sealed class PermissionAttribute : Attribute
{
    public string? Permission { get; }

    public PermissionAttribute()
    {
    }

    public PermissionAttribute(string permission)
    {
        Permission = permission;
    }
}

public sealed class AuthorizationException : Exception
{
    public AuthorizationException() : base("Yetkiniz bulunmamaktadır.")
    {
    }

    public AuthorizationException(string message) : base(message)
    {
    }
}
```

Kullanmak istediğiniz CQRS Request classınızın üstünde çağırıyorsunuz.

```csharp
[Permission]
public sealed record DeveloperCreateCommand(
    string Name
) : IRequest<Result<string>>;

[Permission("permission.create")]
public sealed record DeveloperCreateCommand(
    string Name
) : IRequest<Result<string>>;
```

---

## Exception Handler

Global exception handling için kullanılan handler.

```csharp
public sealed class ExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
    {
        Result<string> errorResult;

        httpContext.Response.ContentType = "application/json";
        httpContext.Response.StatusCode = 500;

        var actualException = exception is AggregateException agg && agg.InnerException != null
        ? agg.InnerException
        : exception;

        var exceptionType = actualException.GetType();
        var validationExceptionType = typeof(ValidationException);
        var authorizationExceptionType = typeof(AuthorizationException);

        if (exceptionType == validationExceptionType)
        {
            httpContext.Response.StatusCode = 422;

            errorResult = Result<string>.Failure(422, ((ValidationException)exception).Errors.Select(s => s.PropertyName).ToList());

            await httpContext.Response.WriteAsJsonAsync(errorResult);

            return true;
        }

        if (exceptionType == authorizationExceptionType)
        {
            httpContext.Response.StatusCode = 403;
            errorResult = Result<string>.Failure(403, "Bu işlem için yetkiniz yok");
            await httpContext.Response.WriteAsJsonAsync(errorResult);
            return true;
        }

        errorResult = Result<string>.Failure(exception.Message);

        await httpContext.Response.WriteAsJsonAsync(errorResult);

        return true;
    }
}
```

---

## DbContext Configuration

IdentityId value object'i için value converter configurationu.

```csharp
internal sealed class IdentityIdValueConverter : ValueConverter<IdentityId, Guid>
{
    public IdentityIdValueConverter() : base(m => m.Value, m => new IdentityId(m)) { }
}

protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
{
    configurationBuilder.Properties<IdentityId>().HaveConversion<IdentityIdValueConverter>();
    base.ConfigureConventions(configurationBuilder);
}
```

---

## OData

OData yı kullanmak istiyorsak yapmamız gereken IoC service registration işlemi ve örnek controller

IoC Container (program.cs)
```csharp
builder.Services
            .AddControllers()
            .AddOData(opt =>
            opt.Select()
                .Filter()
                .Count()
                .Expand()
                .OrderBy()
                .SetMaxTop(null)
            );
```

OData Controller
```csharp
[Route("api/[controller]")]
[ApiController]
[EnableQuery]
public class ODataController : ControllerBase
{
    public static IEdmModel GetEdmModel()
    {
        ODataConventionModelBuilder builder = new();
        builder.EnableLowerCamelCase();
        //builder.EntitySet<UserResponse>("users");
        return builder.GetEdmModel();
    }
}
```

---
## Kullanım Notları

- Bu kod parçaları Clean Architecture ve DDD prensipleri göz önünde bulundurularak hazırlanmıştır
- Entity Framework Core kullanımı için optimize edilmiştir
- MediatR kütüphanesi ile CQRS pattern'i uygulamalarını destekler
- Soft delete özelliği varsayılan olarak tüm entity'lerde aktiftir
- Audit alanları otomatik olarak doldurulur
- Permission attribute ile method bazlı yetkilendirme yapılabilir
