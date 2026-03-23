---
name: test-integration
description: Estándares para Pruebas de Integración de Web APIs usando WebApplicationFactory, base de datos SQLite en memoria, y aislamiento por IClassFixture. Aplica a cualquier API REST con DbContext, UnitOfWork y Repository.
---

# Reglas Estrictas de Pruebas de Integración

Los tests de integración validan el flujo completo de un endpoint: desde la request HTTP hasta la respuesta, pasando por controlador, servicios, repositorios y base de datos. **NUNCA** se conectan a bases de datos de producción ni a entornos externos reales.

> ⚠️ **NOTA PARA EL AGENTE:** Los bloques de código en esta skill son **SOLO ILUSTRATIVOS**. Sirven para explicar el concepto. **NUNCA copies los nombres de clases, métodos o variables de estos ejemplos al proyecto real.** Siempre adaptá el patrón al código concreto que estás analizando o generando.

---

## 1. Configuración del Proyecto y GlobalUsings

Si se crea un proyecto nuevo de integración, es OBLIGATORIO generar un archivo `GlobalUsings.cs` en la raíz:

```csharp
global using Xunit;
global using FluentAssertions;
global using System.Net;
global using System.Net.Http.Json;
```

Agregar más `global using` según las necesidades del proyecto (namespaces del proyecto base, NSubstitute, etc.).

Agregar en `properties/AssemblyInfo.cs` para evitar conflictos de paralelismo entre clases:

```csharp
[assembly: CollectionBehavior(DisableTestParallelization = true)]
```

---

## 2. Estructura de Carpetas

```
{Proyecto}.IntegrationTests/
├── GlobalUsings.cs
├── CustomWebApplicationFactory.cs
├── TestHostEnvironment.cs
├── properties/
│   └── AssemblyInfo.cs
├── Helpers/
│   └── ServiceCollectionExtensions.cs
├── Mocks/
│   └── {Feature}Service/
│       └── {Feature}ServiceMock.cs      ← solo si se necesitan mocks de NSubstitute
└── Features/
    └── {Feature}/
        ├── Data/
        │   └── Scenarios/
        │       └── {Feature}TheoryData.cs
        └── {Feature}Tests.cs
```

- Cada feature tiene su propia subcarpeta bajo `Features/`.
- Los helpers de infraestructura van en `Helpers/`.
- Los mocks de NSubstitute van en `Mocks/{Feature}Service/` y solo se crean si son necesarios.

---

## 3. CustomWebApplicationFactory

Hereda de `WebApplicationFactory<TProgram>` donde `TProgram` es la clase `Program` (o el punto de entrada) de la API bajo test.

El objetivo es **reemplazar el DbContext de producción** con uno que use SQLite en memoria, junto con el UnitOfWork y el Repository que dependen de él.

```csharp
public class CustomWebApplicationFactory<TProgram> : WebApplicationFactory<TProgram>
    where TProgram : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            // 1. Eliminar el DbContext original
            services.RemoveDbContext<AppDbContext>();

            // 2. Eliminar UnitOfWork y otros servicios que dependen del DbContext original
            services.RemoveService<UnitOfWorkBuilder>();

            // 3. Reemplazar el entorno de host para evitar conflictos de configuración
            services.AddSingleton<IHostEnvironment>(sp =>
                new TestHostEnvironment { EnvironmentName = "test" });

            // 4. Crear y mantener abierta la conexión SQLite en memoria durante toda la factory
            SQLitePCL.Batteries_V2.Init();
            var connection = new SqliteConnection("DataSource=:memory:");
            connection.Open();

            // 5. Registrar el DbContext apuntando a la conexión en memoria
            services.AddDbContext<AppDbContext>(options =>
            {
                options.UseSqlite(connection);
            }, ServiceLifetime.Scoped);

            // 6. Crear el esquema de la base de datos en memoria
            using var scope = services.BuildServiceProvider().CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            db.Database.EnsureCreated();
            // Desactivar FK para facilitar la inserción de datos de prueba independientes
            db.Database.ExecuteSqlRaw("PRAGMA foreign_keys = OFF;");

            // 7. Re-registrar UnitOfWork y Repository apuntando al nuevo DbContext
            services.AddScoped<UnitOfWorkBuilder>();
            services.AddScoped<IUnitOfWork>(sp =>
            {
                var dbContext = sp.GetRequiredService<AppDbContext>();
                return new UnitOfWork(dbContext);
            });
        });

        builder.UseEnvironment("test");
    }
}
```

**Reglas clave:**
- La `SqliteConnection` se abre UNA SOLA VEZ y se mantiene abierta durante la vida de la factory; si se cierra, la base de datos en memoria se destruye.
- `EnsureCreated()` crea las tablas usando el modelo definido en el DbContext (respeta relaciones, tipos, índices).
- `PRAGMA foreign_keys = OFF` se usa SOLO en test para poder insertar datos de prueba sin tener que respetar el orden de las FK; si el test requiere validar FK, se activa con `ON`.
- Adaptar los nombres `AppDbContext`, `UnitOfWorkBuilder`, `IUnitOfWork`, `UnitOfWork` a los que use la API real.

---

## 4. TestHostEnvironment

Clase auxiliar para inyectar un entorno de host controlado en los tests:

```csharp
public class TestHostEnvironment : IHostEnvironment
{
    public string EnvironmentName { get; set; } = "test";
    public string ApplicationName { get; set; } = "TestApp";
    public string ContentRootPath { get; set; } = Directory.GetCurrentDirectory();
    public IFileProvider ContentRootFileProvider { get; set; } =
        new PhysicalFileProvider(Directory.GetCurrentDirectory());
}
```

---

## 5. Helpers de ServiceCollection

Crear extensiones para quitar servicios de forma legible y reutilizable:

```csharp
public static class ServiceCollectionExtensions
{
    public static void RemoveService<T>(this IServiceCollection services)
    {
        var descriptor = services.SingleOrDefault(d => d.ServiceType == typeof(T));
        if (descriptor != null) services.Remove(descriptor);
    }

    public static void RemoveDbContext<TContext>(this IServiceCollection services)
        where TContext : DbContext
    {
        var descriptors = services
            .Where(d => d.ServiceType == typeof(TContext) ||
                        d.ServiceType == typeof(DbContextOptions<TContext>))
            .ToList();
        foreach (var descriptor in descriptors) services.Remove(descriptor);
    }

    public static void RemoveAll(this IServiceCollection services,
        Func<ServiceDescriptor, bool> predicate)
    {
        var toRemove = services.Where(predicate).ToList();
        foreach (var descriptor in toRemove) services.Remove(descriptor);
    }
}
```

---

## 6. Datos de Prueba con TheoryData

Usar `TheoryData<TInput, TExpectedStatus>` para parametrizar escenarios de un mismo endpoint. El builder de datos centraliza la construcción de DTOs válidos e inválidos:

```csharp
public class {Feature}TheoryData : TheoryData<{FeatureDtoInsertar}, int>
{
    public {Feature}TheoryData()
    {
        Add(EntidadCorrecta(), 200);                        // caso válido → 200
        Add(EntidadCorrecta(x => x.Nombre = null!), 400);  // caso inválido → 400
    }

    public {FeatureDtoInsertar} EntidadCorrecta(
        Action<{FeatureDtoInsertar}>? configure = null)
    {
        var dto = new {FeatureDtoInsertar}
        {
            Nombre = "Valor de prueba",
            UsuarioCreacion = 1,
            FechaCreacion = DateTime.UtcNow,
            EsActivo = true
        };
        configure?.Invoke(dto);
        return dto;
    }
}
```

- El parámetro `configure` permite sobrescribir campos para escenarios negativos sin duplicar código.
- Usar `[Theory]` + `[ClassData(typeof({Feature}TheoryData))]` cuando haya múltiples escenarios.
- Usar `[Fact]` cuando el test solo cubre un escenario específico.

---

## 7. Mocks con NSubstitute (cuando aplica)

Usar NSubstitute para simular comportamientos del DbContext cuando la lógica a testear no depende de datos reales (por ejemplo, simular errores de base de datos):

```csharp
public static class {Feature}ServiceMock
{
    // Escenario exitoso con datos hardcodeados
    public static void SetupSuccess(AppDbContext dbContext)
    {
        var mockSet = Substitute.For<DbSet<{Entidad}>>();
        var data = new List<{Entidad}>
        {
            new() { Id = 1, Nombre = "Dato de prueba", EsActivo = true }
        }.AsQueryable();

        mockSet.AsQueryable().Provider.Returns(data.Provider);
        mockSet.AsQueryable().Expression.Returns(data.Expression);
        mockSet.AsQueryable().ElementType.Returns(data.ElementType);
        mockSet.AsQueryable().GetEnumerator().Returns(data.GetEnumerator());

        dbContext.{Entidades}.Returns(mockSet);
    }

    // Escenario de error de base de datos
    public static void SetupDatabaseError(AppDbContext dbContext)
    {
        dbContext.{Entidades}
            .Returns(_ => throw new InvalidOperationException("Database error"));
    }
}
```

**Cuándo usar Mocks vs SQLite en memoria:**
- **SQLite en memoria**: cuando el test necesita validar el flujo real de inserción, consulta o actualización con relaciones reales. Es el enfoque preferido.
- **NSubstitute sobre DbContext**: solo cuando se necesita simular errores o comportamientos que no son reproducibles con SQLite (timeouts, excepciones específicas).

---

## 8. Clase de Test

```csharp
public class {Feature}Tests : IClassFixture<CustomWebApplicationFactory<Program>>
{
    private readonly HttpClient _httpClient;
    private const string BaseUrl = "/api/{feature}";

    public {Feature}Tests(CustomWebApplicationFactory<Program> factory)
    {
        _httpClient = factory.CreateClient();
    }

    [Theory]
    [ClassData(typeof({Feature}TheoryData))]
    public async Task Insertar_Cuando_{condicion}_Retorna_{statusCode}(
        {FeatureDtoInsertar} dto, int expectedStatus)
    {
        // ACT
        var response = await _httpClient.PostAsJsonAsync(BaseUrl, dto);

        // ASSERT
        ((int)response.StatusCode).Should().Be(expectedStatus);
    }

    [Fact]
    public async Task Obtener_Cuando_ExistenRegistros_Retorna200()
    {
        // ACT
        var response = await _httpClient.GetAsync(BaseUrl);

        // ASSERT
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }
}
```

**Reglas:**
- `IClassFixture<CustomWebApplicationFactory<Program>>` comparte la factory (y la DB en memoria) entre todos los tests de la clase.
- El `HttpClient` se obtiene de `factory.CreateClient()`, nunca se instancia manualmente.
- Usar `[Theory] + [ClassData]` para múltiples escenarios del mismo endpoint.
- Usar `[Fact]` para escenarios únicos o flujos que no se parametrizan bien.
- Siempre retornar `async Task`.

---

## 9. SignalR Hubs (solo si la API usa SignalR)

Si la API expone SignalR Hubs, agregar un helper de conexión al proyecto de tests:

```csharp
internal static class HubConnectionTestHelper
{
    internal static HubConnection CreateConnection(TestServer server, string hubPath)
    {
        return new HubConnectionBuilder()
            .WithUrl(new Uri(server.BaseAddress, hubPath), options =>
            {
                options.Transports = HttpTransportType.LongPolling;
                options.HttpMessageHandlerFactory = _ => server.CreateHandler();
            })
            .Build();
    }

    internal static async Task<TResult> InvokeAndWaitForCallback<TResult>(
        HubConnection connection,
        string callbackEvent,
        string invokeMethod,
        TimeSpan? timeout = null)
    {
        var tcs = new TaskCompletionSource<TResult>(
            TaskCreationOptions.RunContinuationsAsynchronously);

        connection.On<TResult>(callbackEvent, result => tcs.TrySetResult(result));

        await connection.StartAsync();
        await connection.InvokeAsync(invokeMethod);

        var delay = Task.Delay(timeout ?? TimeSpan.FromSeconds(5));
        var completed = await Task.WhenAny(tcs.Task, delay);
        completed.Should().Be(tcs.Task,
            $"se esperaba el callback '{callbackEvent}' antes del timeout");

        return await tcs.Task;
    }
}
```

- Usar `LongPolling` como transporte (único compatible con `TestServer`).
- Siempre aplicar timeout con `Task.WhenAny` + `Task.Delay`.
- Usar `await using` para disponer la conexión al finalizar el test.

---

## 10. Nomenclatura

- Clases de test: `{Feature}Tests`
- Métodos: `{Acción}_Cuando_{condición}_Retorna_{resultado}` (consistente en todo el proyecto)
- NO mezclar idiomas dentro de la misma suite de tests.