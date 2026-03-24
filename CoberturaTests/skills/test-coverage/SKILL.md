---
name: test-coverage
description: Estándares y comandos exactos para recolección de cobertura de código con Coverlet y ReportGenerator en .NET.
---

# Reglas Estrictas de Cobertura de Pruebas

> ⚠️ **NOTA PARA EL AGENTE:** Los comandos en esta skill son plantillas reales. **Adaptá los paths y nombres de proyecto al proyecto concreto** donde los ejecutes. No uses nombres de ejemplo como `MiProyecto.Test` — usá el nombre real del proyecto que encontraste en el repositorio.

---

## 0. OBLIGATORIO: Plan Previo a la Ejecución

**ANTES de ejecutar cualquier comando**, el agente DEBE presentar en el chat:

1. **Archivos y métodos a cubrir:** Lista exacta de clases `*DomainService` y `*DomainRequirement` encontradas en el proyecto, con sus métodos públicos que serán medidos.
2. **Comandos exactos:** Los comandos completos ya adaptados con los paths y nombres reales del proyecto.
3. **Solicitar aprobación:** Preguntar al usuario *"¿Estás de acuerdo con que proceda?"* y **no ejecutar nada hasta recibir confirmación.**

---

## 1. Paquetes Requeridos

Solo aplica al proyecto de **tests unitarios** (`*.UnitTest.csproj`):

```bash
dotnet add [NombreProyecto.UnitTest]/[NombreProyecto.UnitTest].csproj package coverlet.collector
dotnet add [NombreProyecto.UnitTest]/[NombreProyecto.UnitTest].csproj package coverlet.msbuild
```

Herramientas globales (instalar una sola vez en la máquina):

```bash
dotnet tool install --global coverlet.console
dotnet tool install -g dotnet-reportgenerator-globaltool
```

---

## 2. Clases a Medir — SOLO DomainService y DomainRequirement

La cobertura aplica **ÚNICAMENTE** a métodos de:
- Clases que terminen en `DomainService`
- Clases que terminen en `DomainRequirement`

Usar el parámetro `/p:Include` para filtrar:

```
/p:Include="[*]*DomainService,[*]*DomainRequirement*"
```

Todo lo demás (controllers, infrastructure, DTOs, mappers, etc.) queda **excluido automáticamente**.

---

## 3. Exclusión por Atributo

Para excluir clases adicionales de forma explícita:

```csharp
[ExcludeFromCodeCoverage]
public class MiClaseAExcluir { }
```

Siempre incluir `/p:ExcludeByAttribute="ExcludeFromCodeCoverage"` en el comando.

---

## 4. Comando de Ejecución — Solo Tests Unitarios

La cobertura se ejecuta **ÚNICAMENTE** sobre el proyecto de tests unitarios. Los tests de integración **NO** se ejecutan para cobertura.

Ejecutar desde la raíz de la solución:

```bash
dotnet test ./[NombreProyecto.UnitTest] `
  /p:CollectCoverage=true `
  /p:CoverletOutput="./[NombreProyecto.UnitTest]/coverage.cobertura.xml" `
  /p:CoverletOutputFormat=cobertura `
  /p:Include="[*]*DomainService,[*]*DomainRequirement*" `
  /p:ExcludeByAttribute="ExcludeFromCodeCoverage" `
  /p:Exclude="[*Test*]*"
```

> ⚠️ `CoverletOutput` apunta a la **raíz del proyecto de tests unitarios**. Todos los archivos de cobertura se generan ahí.

---

## 5. Generación del Reporte Visual con ReportGenerator

Una vez generado el `.xml`, crear el reporte HTML dentro del proyecto de tests unitarios:

```bash
reportgenerator `
  "-reports:./[NombreProyecto.UnitTest]/coverage.cobertura.xml" `
  "-targetdir:./[NombreProyecto.UnitTest]/CoverageReport" `
  "-reporttypes:Html"
```

El reporte queda en `[NombreProyecto.UnitTest]/CoverageReport/`. Abrir `index.html` para visualizarlo.