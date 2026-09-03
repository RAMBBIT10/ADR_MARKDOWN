# ADR_MARKDOWN

| Campo | Valor |
|-------|-------|
| **Código** | ESC-CAL-INTG-0007 |
| **Nombre** | Salida válida asociada a proyecto |
| **Atributo de calidad** | Integridad |
| **Categoría** | Reglas de negocio |
| **Característica** | Toda salida debe asociarse a un proyecto y validar el stock antes de registrar el movimiento. |
| **Objetivo** | Registrar una salida solo cuando existe proyecto y stock suficiente. |
| **Criterios de éxito** | La salida se crea y el stock disminuye en la misma operación. |
| **Prerrequisitos** | 1. Proyecto válido. 2. Productos existentes. 3. Cantidades no superiores al stock. |
| **Requisito relacionado** | RF-39 / RN-14 |
| **Tipo de escenario** | Éxito |
