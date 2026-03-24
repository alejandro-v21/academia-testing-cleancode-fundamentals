# Academia — Testing & Clean Code Fundamentals

Framework de conocimiento para aplicar **Clean Code**, **principios SOLID** y **Testing** en proyectos C# / .NET. No contiene código ejecutable; provee estándares documentados y agentes automatizados que los aplican sobre proyectos reales.

---

## Estructura

```
├── CleanCode/
│   ├── agentes/          # Agente auditor de Clean Code
│   └── skills/           # Reglas de Clean Code
│       ├── cc-architecture/
│       ├── cc-complexity/
│       ├── cc-naming/
│       └── cc-solid/
│
├── CoberturaTests/
│   ├── agentes/          # Agente generador de reportes de cobertura
│   └── skills/
│       └── test-coverage/
│
├── DomainRequirement/
│   └── agentes/          # Agente generador de DomainRequirements
│
└── Tests/
    ├── agentes/          # Agente generador de suites de pruebas
    └── skills/           # Reglas de Testing
        ├── test-first-principles/
        ├── test-integration/
        ├── test-mocks/
        └── test-unit/
```

> Los agentes asumen que las skills están en `.github/skills/` en la raíz del proyecto donde se usen.

---

## Skills de Clean Code

| Skill | Qué define |
|---|---|
| `cc-architecture` | DTOs obligatorios en controllers, límite de parámetros, sin números mágicos |
| `cc-complexity` | Límites de complejidad ciclomática/cognitiva y estrategias de reducción |
| `cc-naming` | Convenciones PascalCase/camelCase/`_prefix`, Allman braces, sin notación húngara |
| `cc-solid` | Los 5 principios SOLID con detección de violaciones y cómo corregirlas |

---

## Skills de Testing

| Skill | Qué define |
|---|---|
| `test-unit` | Stack (xUnit + FluentAssertions + NSubstitute), patrón TheoryData, nomenclatura BDD en español |
| `test-first-principles` | Principios F.I.R.S.T. con mapeo al patrón arquitectónico del proyecto |
| `test-integration` | WebApplicationFactory, SQLite en memoria, IClassFixture, estructura de carpetas |
| `test-mocks` | NSubstitute como única librería permitida, Fake vs Mock, prohibiciones absolutas |
| `test-coverage` | Coverlet + ReportGenerator: instalación, comandos, MergeWith para múltiples suites, reporte HTML |

---

## Agentes Automatizados

### `academia-clean-code-expert`
Audita y refactoriza un archivo C# y todas sus dependencias aplicando las skills de Clean Code.

**Comando:** `analizar feature [NombreDelArchivo.cs]`

Fases: rastreo de dependencias → lectura de skills → reporte + plan → refactorización con aprobación previa.

---

### `domain-requirements-builder`
Genera clases `DomainRequirements` (POCOs con booleans) y los métodos privados de consulta en el service, siguiendo el patrón de Clean Architecture del proyecto.

Fases: análisis del service → lectura de entidades → plan de validaciones por operación (Insertar / Actualizar / Eliminar) → implementación con aprobación previa.

---

### `academia-test-expert`
Genera suites completas de pruebas unitarias o de integración respetando las skills correspondientes.

**Comandos:** `unitarios` o `integracion`

Fases: recopilación de contexto → verificación del entorno → plan de casos → generación autónoma.

---

### `coverage-expert`
Descubre los proyectos de prueba existentes, instala Coverlet si falta, ejecuta la cobertura y genera el reporte HTML con ReportGenerator.

**Comando:** `haz la cobertura`

Fases: descubrimiento de tests + SUT → validación de paquetes → plan con comandos exactos + aprobación → ejecución autónoma.

---

## Patrón Arquitectónico Central

Los agentes y las skills de testing giran en torno a este patrón:

```
Controller → ApplicationService → DomainService → DomainRequirements (POCO)
                                        ↑
                              Lógica pura, sin IO
                              Testeable con new + booleans
```

- `DomainRequirements`: POCO con propiedades `bool` que encapsula el resultado de consultas a DB.
- `DomainService`: valida usando `DomainRequirements`, sin tocar infraestructura → 100% testeable.

---

## Uso en un Proyecto

### 1. Instalar GitHub CLI y la extensión de Copilot

```bash
# 1. Instalar GitHub CLI (si no lo tenés)
# Windows (winget)
winget install --id GitHub.cli

# macOS
brew install gh

# 2. Autenticarse
gh auth login

# 3. Instalar la extensión de Copilot
gh extension install github/gh-copilot
```

> Verificá que funcione con: `gh copilot --version`

---

### 2. Copiar skills y agentes al proyecto destino

Desde la raíz del **proyecto donde vas a usar los agentes**, ejecutá:

```bash
# Crear las carpetas necesarias
mkdir -p .github/skills
mkdir -p .github/agents

# Copiar skills de Clean Code
cp -r /ruta/a/este/repo/CleanCode/skills/* .github/skills/

# Copiar skills de Testing
cp -r /ruta/a/este/repo/Tests/skills/* .github/skills/

# Copiar skill de Cobertura
cp -r /ruta/a/este/repo/CoberturaTests/skills/* .github/skills/

# Copiar los agentes
cp /ruta/a/este/repo/CleanCode/agentes/academia-clean-code-expert.md .github/agents/
cp /ruta/a/este/repo/DomainRequirement/agentes/academia-domain-requirement-builder.md .github/agents/
cp /ruta/a/este/repo/Tests/agentes/academia-test-expert.md .github/agents/
cp /ruta/a/este/repo/CoberturaTests/agentes/academia-coverage-expert.md .github/agents/
```

La estructura resultante en tu proyecto debe quedar así:

```
tu-proyecto/
└── .github/
    ├── agents/
    │   ├── academia-clean-code-expert.md
    │   ├── academia-domain-requirement-builder.md
    │   ├── academia-test-expert.md
    │   └── academia-coverage-expert.md
    └── skills/
        ├── cc-architecture/SKILL.md
        ├── cc-complexity/SKILL.md
        ├── cc-naming/SKILL.md
        ├── cc-solid/SKILL.md
        ├── test-first-principles/SKILL.md
        ├── test-integration/SKILL.md
        ├── test-mocks/SKILL.md
        ├── test-unit/SKILL.md
        └── test-coverage/SKILL.md
```

---

### 3. Invocar los agentes en Copilot Chat (VS Code)

Los agentes aparecen automáticamente en GitHub Copilot Chat una vez que los archivos `.md` están en `.github/agents/`. Para invocarlos:

1. Abrí Copilot Chat en VS Code (`Ctrl+Alt+I` / `Cmd+Alt+I`)
2. Seleccioná el agente desde el selector de modo (ícono de agente / `@`)
3. Enviá el comando correspondiente:

| Agente | Comando de activación |
|---|---|
| `academia-clean-code-expert` | `analizar feature NombreDelArchivo.cs` |
| `domain-requirements-builder` | *(se activa automáticamente al seleccionarlo)* |
| `academia-test-expert` | `unitarios` o `integracion` |
| `coverage-expert` | `haz la cobertura` |

> **Requisito:** Tener habilitado **GitHub Copilot** con acceso a **Agent Mode** en VS Code. Versión mínima recomendada de la extensión: `v0.22+`.
