---
name: test-mocks
description: Reglas para el uso de mocks en pruebas. NSubstitute es la única librería permitida. Moq está terminantemente prohibido.
---

# Reglas de Uso de Mocks

## 1. Librería Permitida
- **ÚNICA librería autorizada: `NSubstitute`.**
- **`Moq` está TERMINANTEMENTE PROHIBIDO.** Sin excepciones.
- Si se encuentra código usando `Mock<T>`, `It.IsAny<T>()` o `.Setup(...)` → refactorizar a NSubstitute.

---

## 2. Cuándo Usar Mocks

| Escenario | ¿Usar mock? | Herramienta |
|---|---|---|
| Test unitario de `DomainService` | **NO** | Instanciar con `new`, inyectar POCOs planos |
| Test unitario de infraestructura/aplicación con dependencias externas | **SÍ** | NSubstitute |
| Test de integración con comportamiento fijo | **PREFERIR Fake** | Clase sellada manual |
| Test de integración con comportamiento variable por test | **SÍ** | NSubstitute |

**Regla general: si podés evitar el mock, evitalo. Un Fake o un POCO plano siempre es más claro.**

---

## 3. Sintaxis NSubstitute — Referencia Rápida

### Crear un sustituto
```csharp
var repo = Substitute.For<IUserRepository>();
var logger = Substitute.For<ILogger<UserService>>();
```

### Configurar retorno (Returns)
```csharp
// Valor directo
repo.GetByIdAsync(1).Returns(new User { Id = 1, Name = "Juan" });

// Null
repo.GetByIdAsync(99).Returns((User?)null);

// Tarea async
repo.SaveAsync(Arg.Any<User>()).Returns(Task.CompletedTask);

// Lanzar excepción
repo.GetByIdAsync(Arg.Any<int>()).Throws(new DatabaseException("Error"));
```

### Argumentos (Arg)
```csharp
Arg.Any<int>()              // cualquier int
Arg.Is<string>(s => s.Length > 0) // condición específica
```

### Verificar llamadas (Assert)
```csharp
// Recibió exactamente 1 llamada
await repo.Received(1).SaveAsync(Arg.Any<User>());

// No recibió ninguna llamada
await repo.DidNotReceive().DeleteAsync(Arg.Any<int>());

// Recibió al menos 1 llamada
await repo.Received().GetByIdAsync(1);
```

---

## 4. Estructura en Test Unitario con Mock

```csharp
[Theory]
[ClassData(typeof(UserServiceTest))]
public async Task Dado_UsuarioNoExiste_Cuando_Registrar_Entonces_RetornaError(
    User usuario,
    bool existeEnRepo,
    bool resultadoEsperado,
    string mensajeEsperado)
{
    // ARRANGE
    var repo = Substitute.For<IUserRepository>();
    repo.ExistsAsync(usuario.Email).Returns(existeEnRepo);

    var service = new UserService(repo);

    // ACT
    var resultado = await service.RegistrarAsync(usuario);

    // ASSERT
    resultado.Ok.Should().Be(resultadoEsperado);
    resultado.Mensaje.Should().Be(mensajeEsperado);
    await repo.Received(existeEnRepo ? 0 : 1).SaveAsync(Arg.Any<User>());
}
```

---

## 5. Fakes vs Mocks — Cuándo Usar Cada Uno

### Usar Fake cuando:
- El comportamiento es siempre el mismo (hardcodeado).
- Se usa en múltiples tests y clases.
- Es para tests de integración.

```csharp
// Fake: comportamiento fijo, reutilizable
internal sealed class UserRepositoryFake : IUserRepository
{
    public Task<bool> ExistsAsync(string email) => Task.FromResult(false);
    public Task SaveAsync(User user) => Task.CompletedTask;
}
```

### Usar Mock (NSubstitute) cuando:
- El comportamiento varía según el test case.
- Necesitás verificar que se llamó (o no) a un método específico.

```csharp
// Mock: comportamiento configurable por test
var repo = Substitute.For<IUserRepository>();
repo.ExistsAsync("test@test.com").Returns(true); // varía por caso
```

---

## 6. Prohibiciones

- **NUNCA** mockear clases concretas (solo interfaces).
- **NUNCA** mockear el `DomainService` en tests unitarios de dominio.
- **NUNCA** usar `Moq`, `FakeItEasy` u otras librerías de mocking.
- **NUNCA** verificar llamadas internas de implementación (solo el contrato público).
