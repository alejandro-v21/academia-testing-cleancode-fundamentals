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

Todo lo demás (controllers, infrastructure, DTOs, mappers, etc.) queda **excluido**.

### Filtro vía archivo `coverlet.runsettings` (método obligatorio)

> ⚠️ El parámetro `/p:Include` con comas puede fallar silenciosamente en entornos Windows/PowerShell. El método confiable es un archivo `coverlet.runsettings`.

El agente **DEBE crear o sobreescribir** el archivo `[NombreProyecto.UnitTest]/coverlet.runsettings` con este contenido exacto:

```xml
<?xml version="1.0" encoding="utf-8"?>
<RunSettings>
  <DataCollectionRunSettings>
    <DataCollectors>
      <DataCollector friendlyName="XPlat Code Coverage">
        <Configuration>
          <Format>cobertura</Format>
          <Include>[*]*DomainService,[*]*DomainRequirement*</Include>
          <ExcludeByAttribute>ExcludeFromCodeCoverage</ExcludeByAttribute>
          <Exclude>[*Test*]*</Exclude>
        </Configuration>
      </DataCollector>
    </DataCollectors>
  </DataCollectionRunSettings>
</RunSettings>
```

---

## 3. Exclusión por Atributo

Para excluir clases adicionales de forma explícita:

```csharp
[ExcludeFromCodeCoverage]
public class MiClaseAExcluir { }
```

---

## 4. Comando de Ejecución — Solo Tests Unitarios

La cobertura se ejecuta **ÚNICAMENTE** sobre el proyecto de tests unitarios. Los tests de integración **NO** se ejecutan para cobertura.

> ⚠️ **Por qué `coverlet.msbuild` y no `--collect`:**
> El modo `--collect:"XPlat Code Coverage"` solo instrumenta clases que se **cargan en memoria** durante la ejecución. Si un `DomainService` no tiene tests, puede no aparecer en el reporte.
> El modo `coverlet.msbuild` instrumenta **en tiempo de compilación**, por lo que TODAS las clases `*DomainService` y `*DomainRequirement` referenciadas en el proyecto aparecen en el reporte (con 0% si no tienen tests).

> ⚠️ **Por qué usar `[*]*Domain*` y no `[*]*DomainService,[*]*DomainRequirement*`:**
> En PowerShell/MSBuild la coma es un separador de parámetros y `%2c` (URL-encoded) tampoco es aceptado por coverlet — el filtro no matchea nada y el reporte sale vacío ("N/A"). El wildcard `[*]*Domain*` captura tanto `*DomainService` como `*DomainRequirement` con un solo patrón sin comas.

Ejecutar desde la raíz de la solución:

```bash
dotnet test ./[NombreProyecto.UnitTest] `
  /p:CollectCoverage=true `
  /p:CoverletOutput="./[NombreProyecto.UnitTest]/coverage.cobertura.xml" `
  /p:CoverletOutputFormat=cobertura `
  /p:Include="[*]*Domain*" `
  /p:ExcludeByAttribute="ExcludeFromCodeCoverage" `
  "/p:Exclude=[*Test*]*"
```

---

## 5. Generación del Reporte Visual con ReportGenerator

```bash
reportgenerator `
  "-reports:./[NombreProyecto.UnitTest]/coverage.cobertura.xml" `
  "-targetdir:./[NombreProyecto.UnitTest]/CoverageReport" `
  "-reporttypes:Html"
```

El reporte queda en `[NombreProyecto.UnitTest]/CoverageReport/`. Abrir `index.html` para visualizarlo.

---

## 6. Preguntas Frecuentes

**¿Por qué el reporte sale vacío o muestra "N/A"?**
El filtro `/p:Include` no está matcheando nada. Causas comunes:
- Usar comas en el Include (`[*]*DomainService,[*]*DomainRequirement`) — PowerShell las interpreta como separadores
- Usar `%2c` — coverlet lo toma literal, no como coma
- Patrón equivocado que no coincide con los nombres reales del proyecto

Usá siempre el wildcard simple: `/p:Include="[*]*Domain*"`

**¿Por qué la cobertura sale en 0%?**
El `<Include>` apunta a un patrón que no coincide con los nombres reales del proyecto. Verificar que las clases realmente terminan en `DomainService` o `DomainRequirement`. Ajustar el patrón si usan otra convención de nombres.

**¿Puedo medir otras clases además de DomainService y DomainRequirement?**
Sí. Agregar otro filtro en `<Include>`: `[*]*DomainService,[*]*DomainRequirement*,[*]*OtraClase`.

**¿Los tests de integración se incluyen en la cobertura?**
No. Solo se corre el proyecto `*.UnitTest`. Los tests de integración quedan fuera por diseño.

**¿Dónde quedan los archivos generados?**
Todos dentro del proyecto de tests unitarios:
- `coverlet.runsettings` → configuración del filtro
- `TestResults/` → XML de cobertura raw
- `CoverageReport/` → reporte HTML visual