# ADR_MARKDOWN

# Registrar una salida válida asociada a un proyecto de forma transaccional

- **Título:** Registrar una salida válida asociada a un proyecto de forma transaccional
- **Estado:** Propuesto
- **Escenario:** ESC-CAL-INTG-0007 — Salida válida asociada a proyecto

## Contexto

Servistock debe permitir registrar una salida asociada a un proyecto y descontar correctamente del inventario las cantidades utilizadas. La operación afecta varias entidades relacionadas: salida, líneas de salida, proyecto y stock. Si una parte se guarda y otra falla, el inventario puede quedar inconsistente.

## Alternativas posibles

### A. Guardar cada cambio de forma independiente

Crear primero la salida, después sus líneas y finalmente actualizar el stock con operaciones separadas.

**Evaluación:** Más simple de implementar, pero puede dejar información parcial si una operación intermedia falla.

### B. Procesar la salida de manera asíncrona

Registrar la solicitud y actualizar posteriormente el stock mediante un proceso en segundo plano.

**Evaluación:** Desacopla procesamiento, pero introduce consistencia eventual y no garantiza que el stock quede actualizado en la misma operación.

### C. Transacción ACID en base de datos relacional

Registrar salida, líneas y actualización del stock dentro de una única transacción.

**Evaluación:** Mantiene atomicidad: o se confirma todo o se revierte todo.

## Decisión

Usar PostgreSQL y ejecutar el registro de la salida, sus líneas y la disminución del stock dentro de una única transacción ACID. Antes de confirmar la transacción, el backend validará que el proyecto exista, que los productos existan y que las cantidades sean válidas. Si falla cualquier validación o persistencia, el sistema ejecutará rollback completo.




## Alternativa tecnológica elegida

### Enfoque genérico

Persistencia relacional con transacciones ACID y propagación de eventos después de confirmar la operación.

### Tecnología seleccionada

PostgreSQL para la persistencia y las transacciones; Express sobre Node.js para coordinar el caso de uso; Socket.IO para publicar el cambio confirmado; Angular PWA con RxJS para actualizar la interfaz; Docker para contenerizar frontend, backend y base de datos.

### ¿Por qué se escogió?

PostgreSQL garantiza atomicidad e integridad en salida, líneas, proyecto y stock. Express centraliza la regla de negocio. Socket.IO permite propagar el movimiento solo después del commit. Angular con RxJS facilita actualizar la PWA sin recargar. Docker hace reproducible el despliegue.

### Relación con la reactividad del sistema

La salida se confirma primero en PostgreSQL. Después Express emite un evento por Socket.IO y Angular lo consume mediante RxJS para refrescar el stock automáticamente.

## Consecuencias

- Positiva: evita salidas parcialmente registradas.
- Positiva: mantiene sincronizados salida, proyecto y stock.
- Positiva: facilita auditoría porque la operación queda confirmada como una unidad.
- Negativa: la transacción mantiene bloqueados temporalmente los registros que modifica.
- Negativa: una operación de salida con muchas líneas puede aumentar el tiempo de transacción.
- Alternativas descartadas: guardado independiente y procesamiento asíncrono para la actualización primaria del stock, porque permiten estados intermedios inconsistentes.



