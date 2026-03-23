---
name: test-first-principles
description: Principios F.I.R.S.T. para evaluar y escribir pruebas unitarias de calidad en C# con xUnit y FluentAssertions.
---

# Principios F.I.R.S.T. en Pruebas Unitarias

Un buen test unitario cumple los cinco principios F.I.R.S.T. Este documento explica cada uno con ejemplos en C# y su relación con el patrón arquitectónico del proyecto.

> ⚠️ **NOTA PARA EL AGENTE:** Los bloques de código en esta skill son **SOLO ILUSTRATIVOS**. Sirven para explicar el concepto. **NUNCA copies los nombres de clases, métodos o variables de estos ejemplos al proyecto real.** Siempre adaptá el patrón al código concreto que estás analizando o generando.

---

## F — Fast (Rápido)

Un test unitario debe ejecutarse en milisegundos. Si tarda, nadie lo corre en cada commit y pierde su valor.

### Antipatrón
```csharp
[Fact]
public void Procesar_AntiPattern()
{
    var procesador = new ProcesadorDeDatos();
    procesador.ProcesarTransaccion(); // Thread.Sleep(5000) adentro — inaceptable
}
```

### Patrón correcto — verificar tiempo máximo con FluentAssertions.Extensions
```csharp
[Fact]
public void Procesar_DebeEjecutarse_EnMenosDe200ms()
{
    // ARRANGE
    var procesador = new ProcesadorDeDatos();

    // ACT
    var accion = procesador.ProcesarTransaccion;

    // ASSERT
    accion.ExecutionTime().Should().BeLessThanOrEqualTo(200.Milliseconds());
}
```
> Requiere: `using FluentAssertions.Extensions;`

### Causas comunes de lentitud
- Llamadas reales a base de datos
- Llamadas HTTP sin mockear
- `Thread.Sleep` en el sistema bajo prueba
- Loops o cálculos pesados sin abstracción

### En el patrón del proyecto
El `DomainService` es lógica pura sin IO. Cumple **Fast** por diseño arquitectónico, no por esfuerzo extra.

---

## I — Isolated / Independent (Aislado / Independiente)

Cada test debe poder ejecutarse solo, en cualquier orden, sin depender del resultado de otro test.

### Antipatrón — estado compartido entre tests
```csharp
public class CalculadoraTests
{
    private int ultimoResultado = 0; // estado mutable compartido
    private readonly Calculadora calculadora = new Calculadora();

    [Fact]
    public void Sumar_Debe_Retornar_Suma()
    {
        ultimoResultado = calculadora.Sumar(1, 2);
        ultimoResultado.Should().Be(3);
    }

    [Fact]
    public void Restar_Debe_Retornar_Resta()
    {
        // Si el orden de ejecución cambia, ultimoResultado puede tener valor de otro test
        ultimoResultado = calculadora.Restar(1, 2);
        ultimoResultado.Should().Be(-1);
    }
}
```

### Patrón correcto — cada test se inicializa solo
```csharp
[Fact]
public void Sumar_DebeRetornar_SumaCorrecta()
{
    // ARRANGE
    var calculadora = new Calculadora();

    // ACT
    var resultado = calculadora.Sumar(1, 2);

    // ASSERT
    resultado.Should().Be(3);
}

[Fact]
public void Restar_DebeRetornar_RestaCorrecta()
{
    // ARRANGE
    var calculadora = new Calculadora();

    // ACT
    var resultado = calculadora.Restar(1, 2);

    // ASSERT
    resultado.Should().Be(-1);
}
```

### Reglas de aislamiento
- No compartir instancias mutables entre tests
- Si necesitás setup común, usá el constructor de la clase de test (xUnit crea una instancia nueva por cada `[Fact]` / `[Theory]`)
- No depender de base de datos, archivos ni variables de estado global

### En el patrón del proyecto
`TheoryData` garantiza aislamiento: cada `Add()` es un caso independiente con sus propios datos.
El `DomainService` se instancia en el constructor de la clase de test — xUnit lo recrea por cada ejecución.

---

## R — Repeatable (Repetible)

El test debe dar el mismo resultado siempre, en cualquier entorno, sin importar el momento ni la máquina.
Un test que pasa en local y falla en CI es un **test mentiroso**.

### Antipatrón — no-determinismo por dependencia externa
```csharp
[Fact]
public void Convertir_Dolares_AntiPattern()
{
    var conversor = new ConversorDeMoneda(); // llama a una API real de tipo de cambio
    var resultado = conversor.Convertir(100, "USD", "HNL");
    resultado.Should().Be(2500); // falla mañana cuando el tipo de cambio varía
}
```

### Patrón correcto — mockear la dependencia no determinística con NSubstitute
```csharp
[Fact]
public void Convertir_Dolares_A_Lempiras_DebeRetornar_ValorEsperado()
{
    // ARRANGE
    var conversor = Substitute.For<ConversorDeMoneda>();
    conversor.Convertir(100, "USD", "HNL").Returns(2500);

    // ACT
    var resultado = conversor.Convertir(100, "USD", "HNL");

    // ASSERT
    resultado.Should().Be(2500);
}
```
> Requiere: `NSubstitute` — `dotnet add package NSubstitute`

### Fuentes de no-determinismo
- Llamadas HTTP a servicios externos
- Consultas a base de datos con datos variables
- `DateTime.Now` sin inyección de dependencia
- Números aleatorios sin seed fijo
- Archivos del sistema operativo

### En el patrón del proyecto
`DomainRequirements` resuelve el problema de **Repeatable** de forma arquitectónica:
en lugar de consultar la DB dentro del `DomainService`, los resultados llegan como booleans ya resueltos.
El test controla 100% los valores sin tocar infraestructura.

```csharp
// El test controla si "el nombre existe en DB" → siempre predecible
Add(ProveedorValido(), RequerimentsNombreDuplicado(), false, Mensaje.NOMBRE_PROVEEDOR_DUPLICADO);
//                     ↑ NombreExiste = true, sin llamada a DB
```

---

## S — Self-Checking (Autovalidable)

El test debe determinar solo si pasó o falló. Sin `Console.WriteLine`, sin inspección manual.

### Antipatrón — test que siempre está en verde (mentira verde)
```csharp
[Fact]
public void Sumar_AntiPattern_SinAssert()
{
    var calculadora = new Calculadora();
    var resultado = calculadora.Sumar(2, 3);
    Console.WriteLine("El resultado es: " + resultado); // requiere que alguien lo lea
    // Sin assert → SIEMPRE pasa, incluso si el resultado está mal
}
```

### Patrón correcto — assert que falla automáticamente
```csharp
[Fact]
public void Sumar_DebeRetornar_SumaCorrecta()
{
    // ARRANGE
    var calculadora = new Calculadora();

    // ACT
    var resultado = calculadora.Sumar(2, 3);

    // ASSERT
    resultado.Should().Be(5);
}
```

### Señales de que un test NO es Self-Checking
- Tiene `Console.WriteLine` o `Debug.WriteLine` sin assert
- El assert siempre pasa (`true.Should().BeTrue()`)
- Necesitás leer el log para saber si funcionó
- El test no tiene sección `// ASSERT`

### En el patrón del proyecto
El patrón es Self-Checking por diseño:
```csharp
// ASSERT
resultado.Mensaje ??= "";
resultado.Should().BeEquivalentTo(new
{
    Ok = resultadoEsperado,
    Mensaje = mensajeEsperado
});
```
Si `Ok` o `Mensaje` cambian, el test falla solo. Sin intervención humana.

---

## T — Thorough / Timely (Exhaustivo / A tiempo)

### Thorough — cubrir los casos que importan

No es 100% de cobertura de líneas. Es cubrir los escenarios que pueden fallar:

```csharp
// Caso normal
[Fact]
public void Sumar_DebeRetornar_SumaPositiva() { ... }

// Caso borde — valores negativos
[Fact]
public void Sumar_ConNegativos_DebeRetornar_ResultadoCorrecto() { ... }

// Caso de error — excepción esperada
[Fact]
public void Dividir_PorCero_DebeLanzar_DivideByZeroException()
{
    // ARRANGE
    var calc = new Calculadora();
    var accion = () => calc.Dividir(10, 0);

    // ASSERT
    accion.Should().Throw<DivideByZeroException>();
}
```

### Categorías obligatorias en el patrón del proyecto

| Categoría | Descripción | Ejemplo |
|---|---|---|
| Input inválido | Datos de la entidad incorrectos | nombre vacío, texto muy largo, valor negativo |
| Requirements fallidos | Estado de DB que invalida la operación | usuario no existe, nombre duplicado |
| Múltiples errores | Varios requirements fallando simultáneamente | mensajes concatenados con `"; "` |
| Caso borde (boundary) | Exactamente en el límite válido | nombre de exactamente 100 caracteres |
| Caso feliz | Todo válido → operación exitosa | `Ok = true, Mensaje = Mensaje.EXITO` |

```csharp
// Múltiples errores — el mensaje concatena con "; "
Add(EntidadValida(), RequerimentsMultiplesErrores(), false,
    Mensaje.ERROR_1 + "; " + Mensaje.ERROR_2);

// Caso borde — exactamente en el límite
Add(EntidadNombreExacto100Caracteres(), RequerimentsValido(), true, Mensaje.EXITO);

// Caso feliz — siempre al final
Add(EntidadValida(), RequerimentsValido(), true, Mensaje.EXITO);
```

### Timely — escribir el test a tiempo

| Momento | Valor |
|---|---|
| Antes de codear (TDD) | Máximo — el test guía el diseño |
| Durante el desarrollo | Ideal — el contexto está fresco |
| Semanas después | Mínimo — ya olvidaste los edge cases |
| Después del bug en producción | Negativo — ya hay daño |

---

## Resumen F.I.R.S.T.

```
F → ¿Corre en milisegundos?                → Si no, mockeá las dependencias lentas
I → ¿Puede correr solo, en cualquier orden? → Si no, hay estado compartido o acoplamiento
R → ¿Da el mismo resultado siempre?         → Si no, hay no-determinismo (DB, HTTP, DateTime.Now)
S → ¿Se valida solo sin leer nada?          → Si no, falta el assert
T → ¿Cubre bordes, errores y el happy path? → Si no, la cobertura es cosmética
```

---

## Cómo el patrón del proyecto cumple F.I.R.S.T.

| Principio | Mecanismo arquitectónico |
|---|---|
| **Fast** | `DomainService` = lógica pura, sin IO ni dependencias externas |
| **Isolated** | `TheoryData` + constructor de test: cada caso y cada test se inicializan solos |
| **Repeatable** | `DomainRequirements` reemplaza llamadas a DB con booleans controlados por el test |
| **Self-Checking** | `BeEquivalentTo(new { Ok, Mensaje })` — falla o pasa automáticamente |
| **Thorough** | Las 5 categorías de casos (input inválido, requirements, múltiples errores, borde, feliz) |
