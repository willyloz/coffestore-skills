---
name: dotnet-microsoft-stack
description: >
  Expertise en el ecosistema Microsoft: C# .NET 6/7/8, ASP.NET Core Web APIs,
  Entity Framework Core, MS SQL Server, Azure DevOps, Blazor y código legacy
  Visual Basic 6. Usar cuando se desarrollen APIs REST en .NET, consultas SQL Server,
  migraciones EF Core, pipelines Azure DevOps, componentes Blazor o cuando se
  necesite leer, explicar o migrar código Visual Basic 6 a .NET moderno.
  Trigger: .NET, C#, ASP.NET, Entity Framework, SQL Server, T-SQL, stored procedure,
  Azure, Blazor, MVC, Razor, Visual Basic, VB6, migration, pipeline, YAML.
---

# Stack Microsoft — .NET + SQL Server + Azure

## C# .NET — Patrones modernos

### API REST con ASP.NET Core (Minimal API)
```csharp
// Program.cs — .NET 8 Minimal API
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

var app = builder.Build();

app.MapGet("/orders", async (AppDbContext db) =>
    await db.Orders.Where(o => o.Status == "pending")
                   .OrderByDescending(o => o.CreatedAt)
                   .ToListAsync());

app.MapPost("/orders", async (Order order, AppDbContext db) => {
    db.Orders.Add(order);
    await db.SaveChangesAsync();
    return Results.Created($"/orders/{order.Id}", order);
});

app.Run();
```

### Entity Framework Core — Patrones
```csharp
// DbContext
public class AppDbContext : DbContext
{
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<Order>(e => {
            e.HasKey(o => o.Id);
            e.Property(o => o.Total).HasPrecision(18, 2);
            e.HasOne(o => o.Customer).WithMany(c => c.Orders)
             .HasForeignKey(o => o.CustomerId);
        });
    }
}

// Migrations
// dotnet ef migrations add NombreMigracion
// dotnet ef database update
```

### Clean Architecture — Estructura
```
src/
  Api/           ← Controllers, Program.cs, appsettings.json
  Application/   ← Use cases, DTOs, Interfaces, MediatR handlers
  Domain/        ← Entities, Value Objects, Domain Events
  Infrastructure/← DbContext, Repositories, External Services
```

---

## MS SQL Server — T-SQL avanzado

### Stored Procedures — estructura estándar
```sql
CREATE OR ALTER PROCEDURE sp_GetOrdersByStatus
    @Status     NVARCHAR(50),
    @StartDate  DATETIME2 = NULL,
    @EndDate    DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        o.Id,
        o.OrderNumber,
        o.Total,
        c.Email AS CustomerEmail,
        o.CreatedAt
    FROM Orders o
    INNER JOIN Customers c ON o.CustomerId = c.Id
    WHERE o.Status = @Status
      AND (@StartDate IS NULL OR o.CreatedAt >= @StartDate)
      AND (@EndDate   IS NULL OR o.CreatedAt <= @EndDate)
    ORDER BY o.CreatedAt DESC;
END
```

### Window Functions
```sql
SELECT
    OrderNumber,
    Total,
    SUM(Total)  OVER (PARTITION BY CustomerId) AS TotalPorCliente,
    RANK()      OVER (ORDER BY Total DESC)      AS RankingGlobal,
    LAG(Total)  OVER (ORDER BY CreatedAt)       AS PedidoAnterior
FROM Orders;
```

### Optimización — Índices
```sql
-- Índice para filtros frecuentes
CREATE NONCLUSTERED INDEX IX_Orders_Status_Date
ON Orders (Status, CreatedAt DESC)
INCLUDE (OrderNumber, Total, CustomerId);

-- Ver planes de ejecución
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
-- Ejecutar query
-- Revisar: Logical reads, CPU time, elapsed time
```

### Diagnóstico de performance
```sql
-- Queries más costosas actualmente en ejecución
SELECT TOP 10
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed,
    qs.execution_count,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 200) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY avg_elapsed DESC;
```

---

## Visual Basic 6 — Lectura y migración

### Patrones VB6 comunes y su equivalente en C#

**Manejo de errores**
```vb
' VB6
On Error GoTo ErrorHandler
    ' código
    Exit Sub
ErrorHandler:
    MsgBox "Error: " & Err.Description
```
```csharp
// C# equivalente
try {
    // código
} catch (Exception ex) {
    MessageBox.Show($"Error: {ex.Message}");
}
```

**Conexión a base de datos (ADO)**
```vb
' VB6 con ADO
Dim conn As ADODB.Connection
Dim rs   As ADODB.Recordset
Set conn = New ADODB.Connection
conn.Open "Provider=SQLOLEDB;Server=.;Database=MiDB;Trusted_Connection=yes"
Set rs = conn.Execute("SELECT * FROM Orders WHERE Status='P'")
Do While Not rs.EOF
    Debug.Print rs("OrderNumber")
    rs.MoveNext
Loop
```
```csharp
// C# con Dapper (equivalente moderno)
using var conn = new SqlConnection(connectionString);
var orders = conn.Query<Order>("SELECT * FROM Orders WHERE Status='P'");
foreach (var o in orders) Console.WriteLine(o.OrderNumber);
```

**Formularios y controles**
```vb
' VB6 — evento click de botón
Private Sub cmdGuardar_Click()
    Dim nombre As String
    nombre = txtNombre.Text
    If nombre = "" Then
        MsgBox "El nombre es requerido"
        Exit Sub
    End If
    ' guardar lógica
End Sub
```

### Estrategia de migración VB6 → .NET
```
Complejidad BAJA:  Módulos de utilidades, funciones matemáticas
                   → Migrar directamente a C# static methods

Complejidad MEDIA: Formularios simples, acceso a datos básico
                   → Blazor o WinForms .NET con Dapper

Complejidad ALTA:  ActiveX, COM, APIs Win32, Crystal Reports
                   → Migración incremental con COM Interop
                   → Reescritura gradual por módulos
```

---

## Azure DevOps — Pipelines YAML

### Pipeline CI/CD básico para .NET
```yaml
trigger:
  branches:
    include: [main, develop]

pool:
  vmImage: 'windows-latest'

variables:
  buildConfiguration: 'Release'

stages:
- stage: Build
  jobs:
  - job: BuildAndTest
    steps:
    - task: DotNetCoreCLI@2
      displayName: 'Restore'
      inputs:
        command: 'restore'

    - task: DotNetCoreCLI@2
      displayName: 'Build'
      inputs:
        command: 'build'
        arguments: '--configuration $(buildConfiguration)'

    - task: DotNetCoreCLI@2
      displayName: 'Test'
      inputs:
        command: 'test'
        arguments: '--configuration $(buildConfiguration) --collect:"XPlat Code Coverage"'

    - task: DotNetCoreCLI@2
      displayName: 'Publish'
      inputs:
        command: 'publish'
        publishWebProjects: true
        arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'

    - task: PublishBuildArtifacts@1
      inputs:
        pathToPublish: '$(Build.ArtifactStagingDirectory)'
        artifactName: 'drop'

- stage: Deploy
  dependsOn: Build
  jobs:
  - deployment: DeployWebApp
    environment: 'Production'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            inputs:
              azureSubscription: 'Mi-Subscription'
              appName: 'mi-app-service'
              package: '$(Pipeline.Workspace)/drop/**/*.zip'
```

---

## Blazor — Componentes

### Componente con parámetros y eventos
```razor
@* OrderCard.razor *@
<div class="card">
    <h3>@Order.OrderNumber</h3>
    <p>Total: @Order.Total.ToString("C")</p>
    <button @onclick="HandleClick">Ver detalle</button>
</div>

@code {
    [Parameter] public Order Order { get; set; } = default!;
    [Parameter] public EventCallback<Order> OnSelect { get; set; }

    private async Task HandleClick() =>
        await OnSelect.InvokeAsync(Order);
}
```

### Inyección de servicios y llamadas HTTP
```csharp
@inject HttpClient Http
@inject NavigationManager Nav

@code {
    private List<Order>? orders;

    protected override async Task OnInitializedAsync()
    {
        orders = await Http.GetFromJsonAsync<List<Order>>("api/orders");
    }
}
```
