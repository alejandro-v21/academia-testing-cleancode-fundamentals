---
name: test-coverage
description: Estándares y comandos exactos para recolección de cobertura de código con Coverlet y ReportGenerator en .NET.
---

# Reglas Estrictas de Cobertura de Pruebas

> ⚠️ **NOTA PARA EL AGENTE:** Esta skill usa plantillas. **Reemplazá `[AssemblyPrincipal]` y `[NombreProyecto.UnitTest]` con los nombres reales del proyecto** que encontraste en el repositorio. No uses los nombres de ejemplo tal cual.

---

## 0. OBLIGATORIO: Plan Previo a la Ejecución

**ANTES de ejecutar cualquier comando**, el agente DEBE presentar en el chat:

1. **Assembly principal detectado:** El nombre del assembly del proyecto principal (extraído del `.csproj` o del namespace raíz).
2. **Clases y métodos a medir:** Lista exacta de clases `*DomainService`/`*DominioService` y `*DomainRequirement` encontradas con sus métodos públicos.
3. **Clases excluidas:** Qué namespaces serán excluidos (Controllers, DTOs, Infrastructure, Maps).
4. **Contenido exacto del `coverlet.runsettings`** que se va a crear/sobreescribir.
5. **Comandos exactos** adaptados con paths reales.
6. **Solicitar aprobación:** *"¿Estás de acuerdo con que proceda?"* — **no ejecutar nada hasta recibir confirmación.**

---

## 1. Paquetes Requeridos

Solo en el proyecto de **tests unitarios** (`*.UnitTest.csproj` o `*UniTest.csproj`):

```bash
dotnet add [NombreProyecto.UnitTest]/[NombreProyecto.UnitTest].csproj package coverlet.collector
```

Herramientas globales (una sola vez en la máquina):

```bash
dotnet tool install -g dotnet-reportgenerator-globaltool
```

---

## 2. Archivo `coverlet.runsettings`

El agente **DEBE crear o sobreescribir** el archivo `[NombreProyecto.UnitTest]/coverlet.runsettings`.

**Estrategia:** Incluir TODO el assembly principal y excluir lo que no corresponde medir.

> ⚠️ **Los patrones del `<Exclude>` NO son fijos.** El agente debe inspeccionar el proyecto principal y detectar los namespaces reales que contienen Controllers, DTOs, Mapeos e Infrastructure. Los nombres pueden estar en inglés (`Controllers`, `Dtos`, `Maps`) o en español (`Controladores`, `Dto`, `Mapeados`), en singular o plural. **Usá los nombres reales del proyecto, no los del ejemplo.**

```xml
<?xml version="1.0" encoding="utf-8"?>
<RunSettings>
  <DataCollectionRunSettings>
    <DataCollectors>
      <DataCollector friendlyName="XPlat Code Coverage">
        <Configuration>
          <Format>cobertura</Format>
          <Include>[AssemblyPrincipal]*</Include>
          <Exclude>
            [AssemblyPrincipal]*[NombreRealControllers]*
            [AssemblyPrincipal]*[NombreRealDtos]*
            [AssemblyPrincipal]*[NombreRealMaps]*
            [AssemblyPrincipal]*[NombreRealInfrastructure]*
          </Exclude>
          <ExcludeByAttribute>ExcludeFromCodeCoverage</ExcludeByAttribute>
        </Configuration>
      </DataCollector>
    </DataCollectors>
  </DataCollectionRunSettings>
</RunSettings>
```

**Cómo detectar los nombres reales:**
- Buscá carpetas o namespaces que contengan clases heredadas de `ControllerBase` o `Controller` → ese es el namespace a excluir
- Buscá clases que terminen en `Dto`, `DTO`, `Request`, `Response`, `ViewModel` → ese namespace va en Exclude
- Buscá carpetas `Maps`, `Mapeos`, `Mappings`, `Profiles` (AutoMapper) → ese namespace va en Exclude
- Buscá carpetas `Infrastructure`, `Infraestructura`, `Data`, `Persistence`, `DataBase` → ese namespace va en Exclude

Si alguna de estas categorías no existe en el proyecto, simplemente no la incluyas en el `<Exclude>`.

---

## 3. Exclusión por Atributo

Para excluir clases específicas de forma explícita:

```csharp
[ExcludeFromCodeCoverage]
public class MiClaseAExcluir { }
```

---

## 4. Comando de Ejecución — Solo Tests Unitarios

Los tests de integración **NO** se ejecutan para cobertura.

Ejecutar desde la raíz de la solución:

```bash
dotnet test ./[NombreProyecto.UnitTest] `
  --collect:"XPlat Code Coverage" `
  --settings ./[NombreProyecto.UnitTest]/coverlet.runsettings `
  --results-directory ./[NombreProyecto.UnitTest]/TestResults
```

El archivo `coverage.cobertura.xml` se genera en `[NombreProyecto.UnitTest]/TestResults/[guid]/coverage.cobertura.xml`.

---

## 5. Generación del Reporte Visual con ReportGenerator

Usar glob `**/coverage.cobertura.xml` para capturar el XML dentro de la subcarpeta con guid:

```bash
reportgenerator `
  "-reports:./[NombreProyecto.UnitTest]/TestResults/**/coverage.cobertura.xml" `
  "-targetdir:./[NombreProyecto.UnitTest]/CoverageReport" `
  "-reporttypes:Html"
```

El reporte queda en `[NombreProyecto.UnitTest]/CoverageReport/`. Abrir `index.html`.

---

## 6. Preguntas Frecuentes

**¿Cómo obtengo el nombre del assembly principal?**
Revisá el `.csproj` del proyecto principal. Si tiene `<AssemblyName>MiAssembly</AssemblyName>`, ese es el nombre. Si no, el nombre del assembly es igual al nombre del archivo `.csproj` sin extensión.

**¿Por qué el reporte sale vacío o muestra "N/A"?**
El `<Include>` en el runsettings no matchea el assembly real. El nombre debe coincidir exactamente con el assembly del proyecto principal (case-sensitive). Verificar con: `dotnet build` y buscar el nombre del `.dll` generado en `bin/`.

**¿Por qué no aparecen todos los DomainService?**
El modo `--collect:"XPlat Code Coverage"` solo instrumenta clases que se **cargan en memoria** durante la ejecución. Si un `DomainService` no tiene ningún test que lo instancie, no aparecerá en el reporte. Esto es comportamiento esperado — significa que esa clase no tiene cobertura.

**¿Los tests de integración se incluyen?**
No. Solo se corre el proyecto `*.UnitTest`. Los tests de integración quedan fuera por diseño.

**¿Dónde quedan los archivos generados?**
Todos dentro del proyecto de tests unitarios:
- `coverlet.runsettings` → configuración del filtro
- `TestResults/[guid]/` → XML de cobertura raw
- `CoverageReport/` → reporte HTML visual