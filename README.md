# Prueba Técnica - API de Banca en Línea

## Objetivo

Desarrollar una API RESTful para gestión de operaciones bancarias que demuestre tus habilidades en arquitectura limpia, CQRS, y manejo de stored procedures en .NET.

---

## Descripción del Proyecto

Crear una API que permita realizar operaciones bancarias básicas sobre cuentas de clientes. El proyecto debe seguir los **mismos patrones arquitectónicos** que usamos en producción.

---

## Funcionalidades Requeridas

### 1. Consultar Información de Cuenta
**Endpoint:** `GET /api/v1/accounts/{accountId}`

Debe retornar:
- Número de cuenta
- Saldo actual
- Lista de últimas 10 transacciones
- Intereses acumulados

### 2. Realizar Depósito
**Endpoint:** `POST /api/v1/accounts/{accountId}/deposit`

Debe:
- Validar que el monto sea mayor a 0
- Actualizar el saldo de la cuenta
- Registrar la transacción en el historial
- Retornar el nuevo saldo

### 3. Realizar Retiro
**Endpoint:** `POST /api/v1/accounts/{accountId}/withdrawal`

Debe:
- Validar que el monto sea mayor a 0
- Validar que haya saldo suficiente
- Actualizar el saldo de la cuenta
- Registrar la transacción en el historial
- Retornar el nuevo saldo

### 4. Transferencia entre Cuentas
**Endpoint:** `POST /api/v1/transfers`

Debe:
- Validar que ambas cuentas existan
- Validar saldo suficiente en cuenta origen
- **Usar un Stored Procedure** que maneje la transacción completa (débito + crédito)
- Garantizar atomicidad (todo o nada)
- Registrar la transferencia en ambas cuentas
- Retornar confirmación de la transferencia

---

## Requisitos Técnicos Obligatorios

### Stack Tecnológico

- **.NET 9.0** (o .NET 8 mínimo)
- **ASP.NET Core Web API**
- **SQL Server** (LocalDB o instancia local)
- **ADO.NET** (OBLIGATORIO - NO usar Entity Framework ni Dapper)
- **C#**

### Arquitectura y Patrones (CRÍTICO)

El proyecto **DEBE** seguir esta estructura:

#### 1. Clean Architecture

```
📁 YourProject/
├── 📁 Controllers/
│   ├── BaseApiController.cs          # Usa MediatR
│   └── 📁 v1/
│       ├── AccountsController.cs
│       └── TransfersController.cs
├── 📁 Core/
│   ├── 📁 Application/
│   │   ├── 📁 Services/
│   │   │   ├── 📁 Accounts/
│   │   │   │   ├── 📁 Commands/      # CreateDeposit, CreateWithdrawal
│   │   │   │   └── 📁 Queries/       # GetAccountInfo
│   │   │   └── 📁 Transfers/
│   │   │       └── 📁 Commands/      # CreateTransfer
│   │   ├── 📁 Dtos/                  # DTOs para requests/responses
│   │   └── 📁 Mappings/              # AutoMapper Profiles
│   └── 📁 Domain/
│       ├── 📁 Interfaces/
│       │   └── 📁 Repositories/
│       │       ├── IAccountRepository.cs
│       │       └── ITransferRepository.cs
│       └── 📁 Models/                # Entidades de dominio
│           ├── Account.cs
│           ├── Transaction.cs
│           └── Transfer.cs
├── 📁 Infrastructure/
│   ├── 📁 Repositories/
│   │   ├── AccountRepository.cs      # Implementación con ADO.NET + SPs
│   │   └── TransferRepository.cs
│   └── DependencyInjection.cs        # Registro de repositorios
└── Program.cs
```

#### 2. CQRS con MediatR

**OBLIGATORIO:** Todas las operaciones deben usar el patrón CQRS:

- **Commands:** Para operaciones de escritura (Deposit, Withdrawal, Transfer)
- **Queries:** Para operaciones de lectura (GetAccountInfo)
- **Handlers:** Cada Command/Query tiene su propio Handler

**Ejemplo esperado:**

```csharp
// Command
public class CreateDepositCommand : IRequest<Response<AccountDto>>
{
    public string AccountId { get; set; }
    public decimal Amount { get; set; }
}

// Handler
public class CreateDepositCommandHandler : IRequestHandler<CreateDepositCommand, Response<AccountDto>>
{
    private readonly IAccountRepository _accountRepository;
    private readonly IMapper _mapper;

    public CreateDepositCommandHandler(IAccountRepository accountRepository, IMapper mapper)
    {
        _accountRepository = accountRepository;
        _mapper = mapper;
    }

    public async Task<Response<AccountDto>> Handle(CreateDepositCommand request, CancellationToken cancellationToken)
    {
        // Validaciones básicas
        if (request.Amount <= 0)
            return Response<AccountDto>.Fail("El monto debe ser mayor a 0");

        // Ejecutar depósito
        var account = await _accountRepository.CreateDepositAsync(
            request.AccountId,
            request.Amount,
            cancellationToken);

        if (account == null)
            return Response<AccountDto>.Fail("Cuenta no encontrada");

        var accountDto = _mapper.Map<AccountDto>(account);
        return Response<AccountDto>.Success(accountDto);
    }
}

// Controller
[ApiController]
[Route("api/v1/[controller]")]
public class AccountsController : BaseApiController
{
    [HttpPost("{accountId}/deposit")]
    public async Task<IActionResult> Deposit(string accountId, [FromBody] DepositRequest request)
    {
        var command = new CreateDepositCommand
        {
            AccountId = accountId,
            Amount = request.Amount
        };

        var result = await Mediator.Send(command);

        if (!result.Succeeded)
            return BadRequest(result);

        return Ok(result);
    }
}
```

#### 3. Stored Procedures + ADO.NET (CRÍTICO)

**OBLIGATORIO:** TODAS las operaciones con base de datos deben ejecutarse mediante Stored Procedures usando **ADO.NET puro**.

**NO usar:**
- ❌ Entity Framework
- ❌ Dapper
- ❌ Ningún otro ORM

**¿Por qué?**
- Lógica de negocio esta en Stored Procedures

```sql
CREATE PROCEDURE sp_ExecuteTransfer
    @FromAccountId VARCHAR(50),
    @ToAccountId VARCHAR(50),
    @Amount DECIMAL(18,2)
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRANSACTION;

    BEGIN TRY
        -- code here


        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;

    END CATCH
END
```

#### 4. AutoMapper para DTOs

```csharp
public class AccountProfile : Profile
{
    public AccountProfile()
    {
        CreateMap<Account, AccountDto>();
        CreateMap<Transaction, TransactionDto>();
    }
}
```

#### 5. Inyección de Dependencias

Registrar todos los servicios correctamente:

```csharp
// Program.cs
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly()));
builder.Services.AddAutoMapper(Assembly.GetExecutingAssembly());

// Repositories
builder.Services.AddScoped<IAccountRepository, AccountRepository>();
builder.Services.AddScoped<ITransferRepository, TransferRepository>();
```

---

## Base de Datos

### Scripts SQL Requeridos

Debes incluir un archivo `database-setup.sql` con:

1. **Creación de tablas:**
   - `Accounts` (AccountId, CustomerName, Balance, CreatedDate)
   - `Transactions` (TransactionId, AccountId, Type, Amount, Date, Description)
   - `InterestHistory` (Id, AccountId, InterestRate, CalculatedInterest, CalculationDate)

2. **Stored Procedures:**
   - `sp_ExecuteTransfer` (obligatorio)
   - `sp_GetAccountInfo` (obligatorio)
   - `sp_CreateDeposit` (obligatorio)
   - `sp_CreateWithdrawal` (obligatorio)
   - `sp_CalculateDailyInterest` (bonus)

3. **Datos de prueba:**
   - Al menos 3 cuentas con saldos iniciales

**Ejemplo de tablas:**

```sql
CREATE TABLE Accounts (
    AccountId NVARCHAR(50) PRIMARY KEY,
    CustomerName NVARCHAR(200) NOT NULL,
    Balance DECIMAL(18,2) NOT NULL DEFAULT 0,
    CreatedDate DATETIME NOT NULL DEFAULT GETDATE(),
    CONSTRAINT CK_Accounts_Balance CHECK (Balance >= 0)
);

CREATE TABLE Transactions (
    TransactionId INT IDENTITY(1,1) PRIMARY KEY,
    AccountId NVARCHAR(50) NOT NULL,
    Type NVARCHAR(20) NOT NULL, -- DEPOSIT, WITHDRAWAL, TRANSFER_IN, TRANSFER_OUT
    Amount DECIMAL(18,2) NOT NULL,
    Date DATETIME NOT NULL DEFAULT GETDATE(),
    Description NVARCHAR(500),
    FOREIGN KEY (AccountId) REFERENCES Accounts(AccountId)
);

CREATE TABLE InterestHistory (
    Id INT IDENTITY(1,1) PRIMARY KEY,
    AccountId NVARCHAR(50) NOT NULL,
    InterestRate DECIMAL(5,2) NOT NULL,
    CalculatedInterest DECIMAL(18,2) NOT NULL,
    CalculationDate DATETIME NOT NULL DEFAULT GETDATE(),
    FOREIGN KEY (AccountId) REFERENCES Accounts(AccountId)
);

-- Índices
CREATE INDEX IX_Transactions_AccountId ON Transactions(AccountId);
CREATE INDEX IX_Transactions_Date ON Transactions(Date DESC);
```


---

## Funcionalidades Bonus (Opcionales)

### 1. Cálculo Automático de Intereses
- Stored Procedure `sp_CalculateDailyInterest` que calcula interés diario (0.05% sobre saldo)
- Endpoint `POST /api/v1/interest/calculate` que ejecuta el cálculo
- Guardar historial en tabla `InterestHistory`

### 2. Consulta de Historial de Intereses
- Endpoint `GET /api/v1/accounts/{accountId}/interest-history`
- Retornar cálculos históricos de intereses mediante SP

### 3. Unit Tests
- Pruebas unitarias para Handlers
- Mock de repositorios

### 4. Logging con Serilog
- Configurar Serilog
- Logs estructurados de operaciones críticas (transfers, withdrawals)

### 5. Manejo de Errores Global
- Middleware de manejo de excepciones
- Retornar respuestas consistentes

---

## Criterios de Evaluación

### Arquitectura y Código (40%)
- ✅ Implementación correcta de CQRS con MediatR
- ✅ Clean Architecture (separación de capas)
- ✅ Código limpio, legible y bien organizado
- ✅ Nombres descriptivos y convenciones de C#
- ✅ Principios SOLID aplicados
- ✅ Uso correcto de async/await

### Manejo de Base de Datos (35%) - CRÍTICO
- ✅ TODAS las operaciones usan Stored Procedures
- ✅ Uso correcto de ADO.NET (SqlConnection, SqlCommand, SqlDataReader)
- ✅ Manejo de transacciones (atomicidad)
- ✅ Disposición correcta de recursos (using statements)
- ✅ Lectura correcta de múltiples result sets
- ✅ Uso de parámetros (protección contra SQL Injection)
- ✅ Scripts SQL bien organizados y funcionales

### Validación y Manejo de Errores (15%)
- ✅ Validaciones de negocio en Handlers
- ✅ Validación de datos de entrada
- ✅ Manejo apropiado de excepciones
- ✅ Mensajes de error descriptivos
- ✅ Respuestas HTTP correctas (200, 400, 404, 500)

### Funcionalidad (10%)
- ✅ Todos los endpoints funcionan correctamente
- ✅ La API compila y ejecuta sin errores
- ✅ Casos de prueba cubiertos
- ✅ Transferencias son atómicas (todo o nada)

---

## Entregables

1. **Código fuente completo** en repositorio Git (GitHub, GitLab, Bitbucket)
2. **README.md** con:
   - Instrucciones para ejecutar el proyecto
   - Requisitos previos (.NET 9, SQL Server)
   - Cómo ejecutar los scripts de BD
   - Cómo probar los endpoints (ejemplos de requests)
   - Connection string de ejemplo
3. **Scripts SQL** en carpeta `/Database`
   - `01-create-tables.sql`
   - `02-create-stored-procedures.sql`
   - `03-seed-data.sql`
4. **Colección de Postman** o documentación Swagger para probar la API

---

## Instrucciones de Entrega

1. Crear un repositorio Git público o privado (si es privado, dar acceso al evaluador)
2. El proyecto debe compilar sin errores con `dotnet build`
3. La API debe ejecutarse correctamente con `dotnet run`
4. Los scripts SQL deben ejecutarse en SQL Server sin errores (en orden numérico)
5. Incluir un `.gitignore` apropiado (no subir bin, obj, appsettings con secrets)

---

## Tiempo Estimado

**3-5 días** para completar los requisitos obligatorios.

Los features bonus son opcionales y dependen de tu disponibilidad.

---

## Notas Importantes

- **Prioriza calidad sobre cantidad:** Es mejor tener las funcionalidades básicas bien implementadas que muchas features a medias
- **Sigue los patrones:** La adherencia a CQRS + MediatR + Clean Architecture + ADO.NET es más importante que features extras
- **Piensa en producción:** Escribe código como si fuera a producción (manejo de errores, logging, validaciones)
- **Documenta decisiones:** Si tomas alguna decisión arquitectónica importante, documéntala en el README
- **ADO.NET es obligatorio:** NO uses ORMs. Queremos ver tu dominio en Stored Procedures

---

## Preguntas Frecuentes

**P: ¿Puedo usar Entity Framework o Dapper?**
R: **NO.** Debes usar únicamente ADO.NET para ejecutar Stored Procedures. Así trabajamos en producción y necesitamos evaluar tu dominio de esta tecnología.

**P: ¿Todas las operaciones deben usar SPs?**
R: **SÍ.** Consultas, inserts, updates - todo debe ejecutarse mediante Stored Procedures.

**P: ¿Debo implementar autenticación/autorización?**
R: No es obligatorio, pero es un buen bonus si tienes tiempo.

**P: ¿Qué base de datos usar?**
R: SQL Server LocalDB es suficiente. Incluye la connection string en appsettings.Development.json.

**P: ¿Cuántos endpoints debo implementar?**
R: Mínimo 4: GetAccountInfo, Deposit, Withdrawal, Transfer. El resto son bonus.

---

## Recursos de Referencia

- [MediatR Documentation](https://github.com/jbogard/MediatR)
- [AutoMapper](https://docs.automapper.org/)
- [ADO.NET Documentation](https://learn.microsoft.com/en-us/dotnet/framework/data/adonet/)
- [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [SQL Server Stored Procedures](https://learn.microsoft.com/en-us/sql/relational-databases/stored-procedures/stored-procedures-database-engine)

---

**¡Buena suerte! Esperamos ver tu solución.**
