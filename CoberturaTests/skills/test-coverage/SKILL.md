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

> 🚨 **FORMATO OBLIGATORIO — `<Exclude>` debe ser una sola línea con comas.** Coverlet NO interpreta saltos de línea como separadores. Si los patrones están en líneas separadas, coverlet trata todo el bloque como un único patrón inválido y **silenciosamente ignora todas las exclusiones**. Todos los patrones van en **una sola línea, separados por coma, sin espacios**.

```xml
<?xml version="1.0" encoding="utf-8"?>
<RunSettings>
  <DataCollectionRunSettings>
    <DataCollectors>
      <DataCollector friendlyName="XPlat Code Coverage">
        <Configuration>
          <Format>cobertura</Format>
          <Include>[AssemblyPrincipal]*</Include>
          <Exclude>[AssemblyPrincipal]*[NombreRealControllers]*,[AssemblyPrincipal]*[NombreRealDtos]*,[AssemblyPrincipal]*[NombreRealMaps]*,[AssemblyPrincipal]*[NombreRealInfrastructure]*,[AssemblyPrincipal]*[NombreRealApiResponses]*,[AssemblyPrincipal]*.Program,[AssemblyPrincipal]*.*Dto,[AssemblyPrincipal]*.*DTO,[AssemblyPrincipal]*.*Request,[AssemblyPrincipal]*.*Response,[AssemblyPrincipal]*.*ViewModel</Exclude>
          <ExcludeByAttribute>ExcludeFromCodeCoverage</ExcludeByAttribute>
        </Configuration>
      </DataCollector>
    </DataCollectors>
  </DataCollectionRunSettings>
</RunSettings>
```

> ⚠️ **Estrategia de exclusión en capas:**
> 1. **Por namespace/carpeta detectada:** Controllers, DTOs, Maps, Infrastructure, ApiResponses (cobertura amplia por carpeta).
> 2. **Por nombre de clase (fallback):** Cualquier clase cuyo nombre termine en `Dto`, `DTO`, `Request`, `Response`, `ViewModel` — cubre DTOs en namespaces inesperados.
> 3. **Boilerplate fijo:** `*.Program` siempre se excluye — es el entry point de la app, no es lógica de negocio.

**Cómo detectar los nombres reales de cada categoría (SIEMPRE buscar en el proyecto, nunca asumir):**
- **Controllers:** buscá archivos `.cs` que contengan `ControllerBase` o `: Controller` → el nombre de la carpeta/namespace es el que va en el patrón (ej: `Controllers`, `Controladores`)
- **DTOs:** buscá carpetas que contengan clases terminadas en `Dto`, `DTO`, `Request`, `Response`, `ViewModel` → anotá el nombre exacto de esa carpeta (ej: `Dtos`, `Dto`, `DTOs`, `ViewModels`). **Los fallbacks por nombre de clase ya cubren este caso de todas formas.**
- **Maps/Profiles:** buscá carpetas `Maps`, `Mapeos`, `Mappings`, `Profiles` → usá el nombre exacto encontrado
- **Infrastructure/Data:** buscá carpetas `Infrastructure`, `Infraestructura`, `Data`, `Persistence`, `DataBase` → usá el nombre exacto encontrado
- **ApiResponses/Wrappers:** buscá carpetas `_ApiResponses`, `ApiResponses`, `Responses`, `Wrappers` o clases que terminen en `Helper` dentro de carpetas de respuesta → usá el nombre exacto encontrado. Si no existe esta carpeta, omitir el patrón `[AssemblyPrincipal]*[NombreRealApiResponses]*`.

Si alguna de estas categorías no existe en el proyecto, no incluyas ese patrón de namespace en el `<Exclude>`. Los fallbacks por nombre de clase (`*.*Dto`, `*.*Request`, etc.) y `*.Program` **siempre se incluyen** independientemente de lo detectado.

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

> 🚨 **OBLIGATORIO: limpiar TestResults viejos antes de generar el reporte.** El glob `**/coverage.cobertura.xml` captura **todos** los XMLs de ejecuciones anteriores y los fusiona. Si existe un XML viejo (sin las exclusiones correctas), va a contaminar el reporte con clases que deberían estar excluidas. **Siempre eliminá los TestResults previos antes de correr `reportgenerator`.**

```bash
# Paso 1 — Limpiar TestResults anteriores
Remove-Item -Recurse -Force ./[NombreProyecto.UnitTest]/TestResults

# Paso 2 — Ejecutar tests (genera el XML nuevo y limpio)
dotnet test ./[NombreProyecto.UnitTest] `
  --collect:"XPlat Code Coverage" `
  --settings ./[NombreProyecto.UnitTest]/coverlet.runsettings `
  --results-directory ./[NombreProyecto.UnitTest]/TestResults

# Paso 3 — Generar el reporte HTML
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

**¿Por qué el reporte sigue mostrando clases que deberían estar excluidas (DTOs, Controllers, etc.)?**
Hay tres causas posibles, revisá en este orden:
1. **Formato incorrecto del `<Exclude>`:** Los patrones están en líneas separadas en lugar de una sola línea con comas. Coverlet los ignora silenciosamente. → Corregir al formato de una sola línea con comas (ver Sección 2).
2. **XMLs viejos fusionados:** El `TestResults/` tiene carpetas de ejecuciones anteriores (antes del fix). El glob `**/coverage.cobertura.xml` las fusiona todas. → Eliminar la carpeta `TestResults/` completa y volver a ejecutar `dotnet test` + `reportgenerator` (ver Sección 5).
3. **Namespace no detectado:** La carpeta de DTOs tiene un nombre atípico que no coincide con el patrón `*Dtos*`. Los fallbacks por nombre de clase (`*.*Dto`, `*.*Request`, etc.) deberían cubrirlo igual.

**¿Por qué no aparecen todos los DomainService?**
El modo `--collect:"XPlat Code Coverage"` solo instrumenta clases que se **cargan en memoria** durante la ejecución. Si un `DomainService` no tiene ningún test que lo instancie, no aparecerá en el reporte. Esto es comportamiento esperado — significa que esa clase no tiene cobertura.

**¿Los tests de integración se incluyen?**
No. Solo se corre el proyecto `*.UnitTest`. Los tests de integración quedan fuera por diseño.

**¿Dónde quedan los archivos generados?**
Todos dentro del proyecto de tests unitarios:
- `coverlet.runsettings` → configuración del filtro
- `TestResults/[guid]/` → XML de cobertura raw
- `CoverageReport/` → reporte HTML visual