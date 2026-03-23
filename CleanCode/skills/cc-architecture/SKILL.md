---
name: cc-architecture
description: Reglas de arquitectura para Web APIs, manejo de parámetros y eliminación de números mágicos.
---

# Arquitectura y Diseño de Parámetros

## 1. Web APIs y DTOs
- **PROHIBIDO devolver Entidades de dominio directamente en los controladores.**
- Siempre se deben mapear las entidades y devolver DTOs (Data Transfer Objects).

## 2. Parámetros de Método
- **Límite:** Máximo 3 a 4 parámetros por método.
- Si un método requiere más de 4 parámetros, se debe refactorizar encapsulando los parámetros en una clase o record tipo *Request* u *Object Parameter*.

## 3. Evitar Números Mágicos
- Prohibido dejar valores literales ("quemados") en validaciones lógicas (ej. `if (user.Age > 18)`).
- Extraer estos valores mágicos a constantes, enums, o diccionarios con nombres descriptivos (ej. `private const int LEGAL_AGE = 18;`).