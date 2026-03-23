---
name: cc-naming
description: Aplica convenciones de nombres, notaciones y estilos de formato Allman para código en C#.
---

# Reglas de Nombres y Formato (Clean Code)

## 1. Convenciones de Nombres
- **PascalCase:** Clases, Interfaces (con prefijo I), Structs, Delegates, Enums, Métodos, Propiedades públicas, Eventos, Namespaces, Constantes, Parámetros de Record Types.
- **camelCase:** Variables locales, Parámetros de métodos, Parámetros de constructores primarios.
- **Campos privados:** camelCase con prefijo `_` (ej. `_workerQueue`).
- **Campos estáticos privados:** Prefijo `s_` (ej. `s_workerQueue`).
- **Campos thread-static:** Prefijo `t_` (ej. `t_timeSpan`).

## 2. Reglas Estrictas de Nomenclatura
- **PROHIBIDO el uso de notación húngara** (ej. `int iCounter` o `string strName`). Usa nombres limpios (`counter`, `name`).
- **Nombres Claros:** Los nombres deben ser descriptivos. Evita variables de una sola letra como `x`, `w`, `z` a menos que sean expresiones lambda cortas. (ej. En lugar de `public void x(string z)`, usar `public void GenerarSaludoUsuario(string nombre)`).

## 3. Formato y Estilo
- **Llaves (Braces):** Utiliza SIEMPRE el estilo Allman. Las llaves de apertura y cierre deben ir en una nueva línea, nunca en la misma línea que la declaración.