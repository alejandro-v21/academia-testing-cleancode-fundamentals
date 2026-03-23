---
name: test-unit
description: Reglas arquitectónicas para crear Pruebas Unitarias usando xUnit, FluentAssertions, TheoryData y NSubstitute para infraestructura.
---

# Reglas Estrictas de Pruebas Unitarias

> ⚠️ **NOTA PARA EL AGENTE:** Los bloques de código en esta skill son **SOLO ILUSTRATIVOS**. Sirven para explicar el concepto. **NUNCA copies los nombres de clases, métodos o variables de estos ejemplos al proyecto real.** Siempre adaptá el patrón al código concreto que estás analizando o generando.

## 1. Stack y Convenciones Generales
- **Frameworks:** Usar `xunit`, `FluentAssertions` y, cuando sea necesario, `NSubstitute`.
- **Estructura:** Cada feature debe tener su subcarpeta en `DataTest/{Feature}/`.
- **Nomenclatura de Tests:** Formato BDD en español: `Dado_{condición}_cuando_{acción}_entonces_{resultado_esperado}`.

## 2. Manejo de Dependencias y Mocks (Excepciones de Infraestructura)
- **Para Pruebas de Dominio (DomainService):** Está **PROHIBIDO** el uso de frameworks de mocking. El `DomainService` se instancia directamente con `new`. Las dependencias lógicas se simulan inyectando los objetos planos `DomainRequirements` (POCOs con propiedades booleanas).
- **Para Pruebas de Infraestructura/Aplicación:** Si el código a probar depende de interfaces externas (ej. accesos a datos, APIs de terceros, logs), **SÍ se permite** aislar la infraestructura usando mocks.
- **Librería Obligatoria:** Cuando se requiera el uso de mocks, DEBE usarse **EXCLUSIVAMENTE `NSubstitute`**. Queda terminantemente **PROHIBIDO usar `Moq`**.
- **Verificación de Llamadas:** En la fase de Assert, usa las funcionalidades de NSubstitute como `.Received()` o `.DidNotReceive()` para confirmar que el componente interactuó correctamente con su dependencia (ej. `_logger.Received(1).LogInformation(...)`).

## 3. Patrón TheoryData (Obligatorio para el Dominio)
- **NUNCA** usar `[InlineData]`.
- En las pruebas de dominio, los casos se alimentan desde una clase que hereda de `TheoryData<TEntidad, TDomainRequirements, bool, string>`.
- La clase `TheoryData` **DEBE** estructurarse en 6 secciones comentadas:
  1. Base Entity Factory (`EntidadBase()`)
  2. Entity Mutation Helpers (`ModificarEntidad()`)
  3. Invalid Entity Test Cases
  4. Valid Entity Factory
  5. Entity Cloner
  6. Domain Requirements (Factories, Helpers, Invalid Cases, Cloner).

## 4. Patrón AAA y Asincronismo en la Clase de Test
- Los métodos llevan `[Theory]` y `[ClassData(typeof(XxxTest))]` (o `[Fact]` si no requieren parámetros).
- **Asincronismo:** Si el método a probar es asíncrono, el test debe retornar `async Task` (NUNCA `async void`). Para validar excepciones en métodos asíncronos, usar `await act.Should().ThrowAsync<Excepcion>()`.
- El bloque `// ARRANGE` configura los mocks de infraestructura (si aplica). En pruebas de dominio va vacío.
- El bloque `// ACT` ejecuta el método del sistema bajo prueba.
- El bloque `// ASSERT` normaliza nulls (`resultado.Mensaje ??= "";`) y usa EXCLUSIVAMENTE aserciones legibles de FluentAssertions (ej. `resultado.Should().BeEquivalentTo(...)`).