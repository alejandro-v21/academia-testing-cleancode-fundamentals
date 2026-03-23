---
name: cc-complexity
description: Reglas para evaluar y reducir la complejidad ciclomática y cognitiva del código.
---

# Complejidad Ciclomática y Cognitiva

> ⚠️ **NOTA PARA EL AGENTE:** Los bloques de código en esta skill son **SOLO ILUSTRATIVOS**. Sirven para explicar el concepto. **NUNCA copies los nombres de clases, métodos o variables de estos ejemplos al proyecto real.** Siempre adaptá el patrón al código concreto que estás analizando o generando.

## 1. Complejidad Ciclomática
- **Cálculo:** `if`, `else if`, `for`, `while`, `do-while`, `case`, `catch`, `&&`, `||`, `? :` suman +1.
- **Límites:**
  - 1-4: Excelente.
  - 5-7: Aceptable.
  - 8-10: Alta (Refactorizar).
  - 11-15: Muy alta (Refactorizar urgente).
  - 15+: Extrema (Rediseñar).
- **Estrategia de solución:** Usar "Extract Method" para separar validaciones complejas o usar polimorfismo/switch statements.

## 2. Complejidad Cognitiva
- **Cálculo:** Se penaliza (+1) adicional por cada nivel de anidamiento dentro de bucles o condicionales.
- **Límite:** Mantener estrictamente por debajo de 15.
- **Estrategia de solución:** Extraer lógicas anidadas a métodos privados descriptivos, o usar Streams/LINQ (ej. `Where().Select()`) para hacerlo más declarativo.

## 3. Límites de Tamaño (NOTA DE SISTEMA IMPORTANTE)
*Nota estricta para el agente:* Aunque la literatura mencione límites estrictos (como 20-50 líneas por método o 200 líneas por clase, o la Regla de los 30), **ESTOS LÍMITES DEBEN IGNORARSE EN LA PRÁCTICA AUTOMATIZADA**.
- Únicamente se debe marcar como error un método si supera las **600 líneas**.
- Únicamente se debe marcar como error una clase si supera las **1000 líneas**.
- En cualquier otro caso, priorizar tener tests, eliminar duplicación y expresar intención.