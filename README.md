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

1. Copiar las carpetas `CleanCode/skills/` y `Tests/skills/` a `.github/skills/` en el proyecto destino.
2. Copiar los agentes a `.github/agents/` (o donde configure el cliente de IA).
3. Invocar los agentes con los comandos definidos arriba.
