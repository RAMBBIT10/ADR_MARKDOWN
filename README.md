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

---
# Rechazar una salida cuando el stock disponible es insuficiente

- **Título:** Rechazar una salida cuando el stock disponible es insuficiente
- **Estado:** Propuesto
- **Escenario:** ESC-CAL-INTG-0008 — Salida rechazada por stock insuficiente

## Contexto

Una salida no debe reducir el inventario por debajo de cero ni quedar registrada si alguna de sus líneas solicita una cantidad superior al stock disponible. Además, varios empleados pueden registrar movimientos al mismo tiempo.

## Alternativas posibles

### A. Validar stock solo en la interfaz

La aplicación móvil/web consulta el stock y decide si permite continuar.

**Evaluación:** Rápido de implementar, pero dos usuarios podrían pasar la validación al mismo tiempo y consumir el mismo stock.

### B. Validar stock en backend sin control de concurrencia

El servidor consulta el stock antes de actualizarlo.

**Evaluación:** Mejora la seguridad lógica, pero aún existe una condición de carrera entre consultas y actualizaciones simultáneas.

### C. Validar y bloquear la fila de stock dentro de la transacción

El backend inicia la transacción, bloquea el registro del producto, valida disponibilidad y actualiza.

**Evaluación:** Evita que dos transacciones consuman simultáneamente las mismas unidades.

## Decisión

Validar el stock exclusivamente en el backend dentro de la misma transacción de la salida y aplicar bloqueo pesimista de la fila del producto durante la validación y actualización. Si la cantidad solicitada supera el stock disponible, el backend rechazará la operación y hará rollback sin modificar salida, líneas ni inventario.




## Alternativa tecnológica elegida

### Enfoque genérico

Control de concurrencia transaccional sobre el recurso crítico antes de modificar el stock.

### Tecnología seleccionada

PostgreSQL con bloqueo pesimista de fila (`SELECT ... FOR UPDATE`) dentro de la transacción; Express para aplicar la validación; Socket.IO y Angular/RxJS para propagar el resultado confirmado; Docker para despliegue.

### ¿Por qué se escogió?

La regla de stock insuficiente necesita consistencia fuerte. PostgreSQL puede bloquear el registro del producto durante la validación y actualización. Express mantiene la regla en backend y no en la interfaz.

### Relación con la reactividad del sistema

El rechazo o confirmación de la salida se resuelve en backend. Solo los cambios confirmados se publican por Socket.IO hacia la PWA.

## Consecuencias

- Positiva: impide stock negativo por operaciones concurrentes.
- Positiva: la regla no depende de la interfaz del usuario.
- Positiva: el error se resuelve antes de confirmar cualquier dato.
- Negativa: dos salidas concurrentes del mismo producto pueden esperar por el bloqueo.
- Negativa: se requiere controlar cuidadosamente el orden de bloqueo cuando una salida contiene varios productos para reducir riesgo de deadlocks.
- Alternativas descartadas: validación solo en frontend y validación backend sin bloqueo, por riesgo de condiciones de carrera.

---
# Mantener el registro de salidas por debajo de 1.5 segundos

- **Título:** Mantener el registro de salidas por debajo de 1.5 segundos
- **Estado:** Propuesto
- **Escenario:** ESC-CAL-REN-0005 — Registro de salida confirmado en menos de 1.5 segundos

## Contexto

El registro de una salida debe validar proyecto, productos y stock, persistir la salida y sus líneas, y actualizar inventario. Todas estas operaciones deben completarse dentro del tiempo de respuesta establecido.

## Alternativas posibles

### A. Ejecutar múltiples consultas individuales por cada línea

Consultar y actualizar cada producto por separado sin optimización.

**Evaluación:** Fácil de programar, pero aumenta viajes a la base de datos y degrada el tiempo con muchas líneas.

### B. Usar caché para el stock

Consultar disponibilidad desde caché y luego persistir cambios.

**Evaluación:** Puede ser rápido, pero complica la consistencia del stock y requiere invalidación correcta.

### C. Optimizar la transacción y las consultas SQL

Usar índices, consultas por lote, relaciones bien indexadas y limitar datos recuperados.

**Evaluación:** Mantiene consistencia sin introducir una capa de caché adicional.

## Decisión

Mantener la operación síncrona y transaccional en PostgreSQL, crear índices sobre las claves de consulta frecuentes (producto, proyecto y relaciones de salida), recuperar en una sola consulta por lote los productos involucrados y ejecutar actualizaciones dentro de la misma transacción. No incorporar caché de stock en la primera versión; se evaluará solo mediante Spike si las pruebas no cumplen 1.5 segundos.




## Alternativa tecnológica elegida

### Enfoque genérico

Backend asíncrono orientado a eventos, consultas SQL optimizadas y actualización reactiva del cliente.

### Tecnología seleccionada

Express/Node.js para manejo asíncrono de solicitudes; PostgreSQL con índices y consultas por lote; Socket.IO para eventos en tiempo real; Angular PWA con RxJS; Docker.

### ¿Por qué se escogió?

Node.js/Express trabaja bien con operaciones I/O concurrentes. PostgreSQL resuelve la persistencia crítica. RxJS forma parte del ecosistema Angular y permite manejar flujos de datos reactivos en la interfaz.

### Relación con la reactividad del sistema

Express evita bloquear el hilo con I/O y Socket.IO empuja cambios a Angular. RxJS permite que componentes suscritos actualicen su estado automáticamente.

## Consecuencias

- Positiva: reduce viajes a la base de datos.
- Positiva: mantiene la consistencia fuerte del stock.
- Positiva: evita la complejidad de invalidar caché desde el inicio.
- Negativa: el rendimiento dependerá del diseño de índices y del volumen real.
- Negativa: una salida con muchas líneas puede acercarse al límite temporal.
- Alternativa condicionada: caché se mantiene como opción posterior únicamente si un Spike de rendimiento demuestra que las consultas optimizadas no cumplen el objetivo.

---

# Garantizar disponibilidad operativa durante el horario laboral

- **Título:** Garantizar disponibilidad operativa durante el horario laboral
- **Estado:** Propuesto
- **Escenario:** ESC-CAL-DIS-0001 — Disponibilidad del sistema en horario laboral

## Contexto

Servistock debe mantenerse operativo durante el horario laboral. Una caída del backend o del proceso de aplicación no debe requerir intervención manual prolongada, y los datos persistidos no deben perderse.

## Alternativas posibles

### A. Despliegue en un único proceso sin recuperación automática

Si el proceso cae, un integrante del equipo lo reinicia manualmente.

**Evaluación:** Menor complejidad, pero aumenta el tiempo de indisponibilidad.

### B. Reinicio automático del servicio

El entorno de despliegue supervisa el proceso y lo reinicia cuando falla.

**Evaluación:** Reduce indisponibilidad por fallos de proceso con complejidad moderada.

### C. Alta disponibilidad con múltiples réplicas activas

Varias instancias detrás de balanceador y base de datos redundante.

**Evaluación:** Mayor tolerancia a fallos, pero mayor complejidad y costo operativo.

## Decisión

Desplegar el backend como servicio administrado en la nube con health check y política de reinicio automático ante fallo del proceso. Mantener los datos en PostgreSQL persistente con copias de seguridad programadas. Antes de adoptar múltiples réplicas, ejecutar un Spike de disponibilidad para comprobar si el esquema básico alcanza el objetivo definido.




## Alternativa tecnológica elegida

### Enfoque genérico

Despliegue contenerizado con supervisión, reinicio automático y persistencia durable.

### Tecnología seleccionada

Docker para empaquetar Angular PWA y Express; PostgreSQL persistente; health checks de Docker/infraestructura; despliegue en un servicio de nube compatible con contenedores.

### ¿Por qué se escogió?

Docker permite tener el mismo entorno en desarrollo y despliegue. Los health checks facilitan detectar fallos y reiniciar servicios. PostgreSQL mantiene los datos fuera del ciclo de vida del contenedor de aplicación.

### Relación con la reactividad del sistema

La reactividad mejora la experiencia durante operación normal, pero la disponibilidad se cubre con health checks, reinicio y persistencia. Socket.IO se reconecta cuando el servicio vuelve.

## Consecuencias

- Positiva: recupera automáticamente fallos del proceso de aplicación.
- Positiva: reduce dependencia de intervención manual.
- Positiva: separa recuperación de aplicación y persistencia de datos.
- Negativa: no elimina todos los puntos únicos de fallo.
- Negativa: alcanzar 99.999 % puede requerir posteriormente redundancia de infraestructura y base de datos.
- Alternativa condicionada: múltiples réplicas y balanceador solo se incorporarán si el Spike demuestra que el despliegue básico no puede cumplir el objetivo.

---
# Ofrecer el mismo backend para los canales web y móvil

- **Título:** Ofrecer el mismo backend para los canales web y móvil
- **Estado:** Propuesto
- **Escenario:** ESC-CAL-DIS-0002 — Acceso al sistema desde web y móvil

## Contexto

El sistema debe ser accesible desde navegador web y desde una aplicación móvil. Ambos canales utilizan las mismas reglas de negocio de inventario, entradas, salidas, proyectos y seguridad.

## Alternativas posibles

### A. Backend independiente para web y móvil

Cada cliente tendría su propia lógica y endpoints.

**Evaluación:** Permite evolución separada, pero duplica lógica y aumenta mantenimiento.

### B. Backend único con API compartida

Web y móvil consumen los mismos servicios de aplicación.

**Evaluación:** Centraliza reglas y reduce duplicación.

### C. Lógica principal implementada en cada cliente

Cada aplicación resuelve validaciones localmente.

**Evaluación:** Reduce trabajo del servidor, pero aumenta inconsistencias y duplicación.

## Decisión

Implementar un único backend modular con API REST compartida por la aplicación web y la aplicación móvil. Todas las reglas de negocio y validaciones críticas se ejecutarán en el backend; los clientes se limitarán a interacción, presentación y capacidades propias del dispositivo, como la cámara.




## Alternativa tecnológica elegida

### Enfoque genérico

Cliente web instalable único, backend compartido y canal bidireccional de actualización en tiempo real.

### Tecnología seleccionada

Angular configurado como PWA; Express como API backend; Socket.IO para comunicación en tiempo real; RxJS en Angular; PostgreSQL; Docker.

### ¿Por qué se escogió?

Angular PWA permite una sola base de código para escritorio y móvil. Express centraliza reglas. Socket.IO evita refrescos manuales y RxJS integra naturalmente los eventos en la aplicación.

### Relación con la reactividad del sistema

Los clientes Angular se suscriben mediante Socket.IO/RxJS a cambios de stock, estados y notificaciones enviados por Express.

## Consecuencias

- Positiva: una misma regla de negocio se implementa una sola vez.
- Positiva: web y móvil mantienen comportamiento consistente.
- Positiva: facilita seguridad y auditoría centralizadas.
- Negativa: ambos clientes dependen de la disponibilidad de la misma API.
- Negativa: cambios incompatibles en la API requieren versionado o coordinación con los clientes.
- Alternativas descartadas: backends separados y lógica duplicada en clientes por mayor costo de mantenimiento e inconsistencia.

---

# Mantener el stock actualizado después de cada movimiento

- **Título:** Mantener el stock actualizado después de cada movimiento
- **Estado:** Propuesto
- **Escenario:** ESC-CAL-INTG-0003 — Consulta de stock actualizado

## Contexto

Después de una entrada o salida, las consultas posteriores deben reflejar el nuevo stock. El equipo también definió como restricción técnica que los cambios se propaguen automáticamente y que varios empleados puedan operar simultáneamente.

## Alternativas posibles

### A. Recalcular stock sumando todas las entradas y salidas en cada consulta

No se almacena stock actual; se calcula bajo demanda.

**Evaluación:** Evita duplicar el dato, pero puede degradar rendimiento al crecer el historial.

### B. Mantener una columna de stock y actualizarla transaccionalmente

El stock actual se persiste y cambia con cada movimiento confirmado.

**Evaluación:** Consulta rápida y consistencia fuerte si se actualiza en la misma transacción.

### C. Actualizar stock mediante eventos asíncronos

La salida/entrada se registra y un consumidor actualiza stock después.

**Evaluación:** Escalable, pero introduce una ventana de consistencia eventual.

## Decisión

Persistir el stock actual por producto y actualizarlo dentro de la misma transacción que confirma una entrada o salida. Las consultas de stock leerán directamente ese valor persistido. Las operaciones concurrentes usarán control de concurrencia sobre el registro del producto.




## Alternativa tecnológica elegida

### Enfoque genérico

Stock persistido como fuente autoritativa y propagación de cambios basada en eventos post-commit.

### Tecnología seleccionada

PostgreSQL para almacenar stock actual; Express para publicar eventos después de confirmar la transacción; Socket.IO para distribución; Angular PWA + RxJS para consumo; Docker.

### ¿Por qué se escogió?

Persistir el stock hace rápidas las consultas y mantiene una fuente única. La emisión posterior al commit evita mostrar estados que luego sean revertidos.

### Relación con la reactividad del sistema

Cada entrada o salida confirmada produce un evento de inventario. Angular recibe el evento y actualiza vistas suscritas sin recarga.

## Consecuencias

- Positiva: las consultas de stock son directas y rápidas.
- Positiva: el valor cambia en la misma operación que el movimiento.
- Positiva: soporta el escenario de consulta en tiempo cercano al real.
- Negativa: cualquier proceso que modifique movimientos debe respetar la misma regla transaccional.
- Negativa: requiere evitar actualizaciones directas del stock por fuera de los servicios autorizados.
- Alternativas descartadas: recálculo completo por costo de consulta y actualización asíncrona por introducir consistencia eventual.

---

# Responder consultas de stock en menos de 1 segundo

- **Título:** Responder consultas de stock en menos de 1 segundo
- **Estado:** Propuesto
- **Escenario:** ESC-CAL-REN-0001 — Consulta de stock en menos de 1 segundo

## Contexto

La consulta del stock de un producto es una de las operaciones más frecuentes y debe responder en menos de un segundo.

## Alternativas posibles

### A. Consultar el historial completo y calcular stock

Obtener todos los movimientos y calcular disponibilidad.

**Evaluación:** No requiere stock persistido, pero es costoso.

### B. Consultar stock persistido con índice

Buscar el producto por identificador/código indexado y devolver su stock actual.

**Evaluación:** Simple, consistente y rápido.

### C. Mantener stock en caché en memoria

Leer desde Redis u otra caché.

**Evaluación:** Muy rápido, pero agrega infraestructura e invalidación.

## Decisión

Consultar directamente el stock persistido en PostgreSQL mediante búsquedas por clave primaria, código o QR indexados. La consulta devolverá únicamente los campos necesarios para el detalle del producto. No usar caché distribuida inicialmente; un Spike de carga determinará si es necesaria.




## Alternativa tecnológica elegida

### Enfoque genérico

Consulta directa indexada sobre el stock persistido y actualización reactiva posterior.

### Tecnología seleccionada

PostgreSQL con índices sobre id, código y QR; Express para el endpoint; Angular PWA con RxJS; Socket.IO para posteriores cambios; Docker.

### ¿Por qué se escogió?

La búsqueda indexada evita recalcular movimientos y reduce tiempo de respuesta. Express mantiene la consulta simple y Angular/RxJS permite mantener la pantalla actualizada.

### Relación con la reactividad del sistema

La consulta inicial llega por API; cambios posteriores del mismo producto pueden recibirse por Socket.IO.

## Consecuencias

- Positiva: reduce la cantidad de datos procesados.
- Positiva: evita infraestructura adicional en la primera versión.
- Positiva: mantiene una única fuente autoritativa de stock.
- Negativa: un diseño deficiente de índices puede incumplir el objetivo.
- Alternativa condicionada: Redis u otra caché se evaluará solo si el Spike demuestra que PostgreSQL optimizado no cumple el tiempo.

---

# Responder consultas de stock en menos de 1 segundo

- **Título:** Responder consultas de stock en menos de 1 segundo
- **Estado:** Propuesto
- **Escenario:** ESC-CAL-REN-0001 — Consulta de stock en menos de 1 segundo

## Contexto

La consulta del stock de un producto es una de las operaciones más frecuentes y debe responder en menos de un segundo.

## Alternativas posibles

### A. Consultar el historial completo y calcular stock

Obtener todos los movimientos y calcular disponibilidad.

**Evaluación:** No requiere stock persistido, pero es costoso.

### B. Consultar stock persistido con índice

Buscar el producto por identificador/código indexado y devolver su stock actual.

**Evaluación:** Simple, consistente y rápido.

### C. Mantener stock en caché en memoria

Leer desde Redis u otra caché.

**Evaluación:** Muy rápido, pero agrega infraestructura e invalidación.

## Decisión

Consultar directamente el stock persistido en PostgreSQL mediante búsquedas por clave primaria, código o QR indexados. La consulta devolverá únicamente los campos necesarios para el detalle del producto. No usar caché distribuida inicialmente; un Spike de carga determinará si es necesaria.




## Alternativa tecnológica elegida

### Enfoque genérico

Consulta directa indexada sobre el stock persistido y actualización reactiva posterior.

### Tecnología seleccionada

PostgreSQL con índices sobre id, código y QR; Express para el endpoint; Angular PWA con RxJS; Socket.IO para posteriores cambios; Docker.

### ¿Por qué se escogió?

La búsqueda indexada evita recalcular movimientos y reduce tiempo de respuesta. Express mantiene la consulta simple y Angular/RxJS permite mantener la pantalla actualizada.

### Relación con la reactividad del sistema

La consulta inicial llega por API; cambios posteriores del mismo producto pueden recibirse por Socket.IO.

## Consecuencias

- Positiva: reduce la cantidad de datos procesados.
- Positiva: evita infraestructura adicional en la primera versión.
- Positiva: mantiene una única fuente autoritativa de stock.
- Negativa: un diseño deficiente de índices puede incumplir el objetivo.
- Alternativa condicionada: Redis u otra caché se evaluará solo si el Spike demuestra que PostgreSQL optimizado no cumple el tiempo.

---

# Generar exportaciones de historial de pedidos dentro del tiempo objetivo

- **Título:** Generar exportaciones de historial de pedidos dentro del tiempo objetivo
- **Estado:** Propuesto
- **Escenario:** ESC-CAL-REN-0007 — Exportación de historial de pedidos con datos

## Contexto

El usuario debe poder exportar el historial de pedidos a un archivo Excel. La operación fue clasificada bajo Rendimiento, por lo que debe medirse el tiempo de generación.

## Alternativas posibles

### A. Generar Excel completamente en memoria

Cargar todos los registros, construir el workbook y luego enviarlo.

**Evaluación:** Simple, pero puede consumir mucha memoria con grandes volúmenes.

### B. Generar Excel por streaming

Leer registros en lotes y escribir el archivo progresivamente.

**Evaluación:** Reduce memoria y mantiene operación síncrona.

### C. Generar archivo de forma asíncrona

Crear un trabajo en segundo plano y notificar cuando esté listo.

**Evaluación:** Escala mejor para grandes volúmenes, pero cambia la experiencia inmediata.

## Decisión

Generar la exportación de forma síncrona desde el backend utilizando lectura paginada/por lotes y escritura streaming del archivo XLSX. Ejecutar un Spike con volúmenes representativos; si el tiempo supera el objetivo, cambiar esta exportación a generación asíncrona.




## Alternativa tecnológica elegida

### Enfoque genérico

Generación de XLSX por streaming y aislamiento de tareas pesadas del flujo principal de solicitudes.

### Tecnología seleccionada

Express para orquestar la exportación; ExcelJS en modo streaming para generar XLSX; PostgreSQL con consultas paginadas; Docker. Si un Spike lo exige, usar worker thread/proceso separado en Node.js.

### ¿Por qué se escogió?

ExcelJS es compatible con Node.js y permite generar archivos XLSX sin cargar todo el libro en memoria. La lectura por lotes reduce consumo de memoria y evita bloquear otras solicitudes.

### Relación con la reactividad del sistema

Las exportaciones no deben bloquear las operaciones en tiempo real. Si crecen en volumen, se ejecutan en un worker y el progreso puede notificarse por Socket.IO.

## Consecuencias

- Positiva: controla el uso de memoria.
- Positiva: mantiene una experiencia de descarga inmediata mientras el volumen lo permita.
- Negativa: la generación ocupa recursos del backend durante la solicitud.
- Negativa: volúmenes muy grandes pueden superar el límite de tiempo.
- Alternativa condicionada: procesamiento asíncrono se reserva para el caso en que el Spike evidencie incumplimiento.

---

# Generar exportaciones de proyectos de manera eficiente

- **Título:** Generar exportaciones de proyectos de manera eficiente
- **Estado:** Propuesto
- **Escenario:** ESC-CAL-REN-0013 — Exportación de proyectos con datos

## Contexto

La exportación de proyectos puede incluir información relacionada con productos, cantidades y movimientos, por lo que puede producir consultas más complejas que una exportación plana.

## Alternativas posibles

### A. Ejecutar una consulta por proyecto y luego una por cada relación

Implementación directa con múltiples consultas.

**Evaluación:** Produce problema N+1 y aumenta el tiempo.

### B. Construir una consulta optimizada con joins/proyección

Recuperar únicamente las columnas necesarias en una consulta o conjunto reducido de consultas.

**Evaluación:** Reduce viajes a la base de datos.

### C. Mantener reportes precalculados

Generar periódicamente una vista/materialización para exportación.

**Evaluación:** Muy rápido al descargar, pero agrega sincronización y mantenimiento.

## Decisión

Implementar una consulta específica de exportación con joins/proyecciones sobre los datos requeridos y escribir el XLSX mediante streaming. Evitar cargar entidades completas del dominio cuando solo se necesitan campos de reporte. Validar con Spike el volumen esperado antes de considerar vistas materializadas o procesamiento asíncrono.




## Alternativa tecnológica elegida

### Enfoque genérico

Consulta de reporte especializada, evitando N+1, y generación XLSX por streaming.

### Tecnología seleccionada

PostgreSQL con joins/proyecciones; Express; ExcelJS streaming; Docker.

### ¿Por qué se escogió?

Las proyecciones reducen datos innecesarios y evitan múltiples consultas por proyecto. ExcelJS permite construir el archivo progresivamente.

### Relación con la reactividad del sistema

La exportación queda separada del flujo de eventos de inventario; si se procesa en segundo plano, Express puede notificar disponibilidad del archivo por Socket.IO.

## Consecuencias

- Positiva: evita consultas N+1.
- Positiva: reduce memoria y transferencia desde la base de datos.
- Negativa: las consultas de exportación quedan especializadas para el formato requerido.
- Negativa: cambios en las columnas del reporte requieren ajustar la proyección.
- Alternativas descartadas inicialmente: consultas N+1 por bajo rendimiento y reportes precalculados por complejidad prematura.

---

# Exportar el inventario actual sin recalcular movimientos históricos

- **Título:** Exportar el inventario actual sin recalcular movimientos históricos
- **Estado:** Propuesto
- **Escenario:** ESC-CAL-REN-0015 — Exportación de inventario actual con datos

## Contexto

La exportación del inventario debe reflejar el stock actual de los productos y generarse con buen rendimiento.

## Alternativas posibles

### A. Calcular stock desde movimientos al exportar

Sumar entradas y restar salidas para cada producto.

**Evaluación:** Garantiza derivación histórica, pero aumenta costo.

### B. Exportar el stock actual persistido

Leer productos y su stock actual.

**Evaluación:** Rápido y coherente con la estrategia transaccional de stock.

### C. Exportar desde una caché/reporting database

Mantener una copia optimizada para reportes.

**Evaluación:** Rápido, pero introduce sincronización adicional.

## Decisión

Generar la exportación desde la tabla de productos/stock actual persistido, usando consulta proyectada y escritura streaming a XLSX. La exportación no recalculará el inventario desde el historial de movimientos. El historial se utilizará para auditoría, no como fuente de cálculo en cada exportación.




## Alternativa tecnológica elegida

### Enfoque genérico

Exportación desde el valor de stock persistido, sin recalcular todo el historial.

### Tecnología seleccionada

PostgreSQL como fuente de stock actual; Express; ExcelJS streaming; Docker.

### ¿Por qué se escogió?

Leer el stock persistido es más eficiente que sumar entradas y salidas en cada exportación. ExcelJS controla el uso de memoria.

### Relación con la reactividad del sistema

La generación del archivo se mantiene aislada para no afectar las actualizaciones reactivas del inventario.

## Consecuencias

- Positiva: reduce significativamente el costo de exportación.
- Positiva: usa la misma fuente autoritativa de stock que las consultas operativas.
- Negativa: depende de que todas las entradas y salidas actualicen correctamente el stock persistido.
- Alternativas descartadas: recálculo histórico por costo y base de reporting separada por complejidad innecesaria en el alcance actual.

---
# Restringir la creación de empleados al administrador

- **Título:** Restringir la creación de empleados al administrador
- **Estado:** Propuesto
- **Escenario:** ESC-CAL-SEG-0007 — Administrador crea una cuenta de empleado

## Contexto

La creación de cuentas de empleado es una operación privilegiada. El sistema debe impedir que usuarios sin rol administrativo creen nuevas cuentas.

## Alternativas posibles

### A. Ocultar el botón en la interfaz

Solo el administrador ve la opción de creación.

**Evaluación:** Mejora experiencia, pero no evita llamadas directas a la API.

### B. Validar rol dentro de cada controlador

Cada endpoint comprueba manualmente el tipo de empleado.

**Evaluación:** Funciona, pero repite lógica y facilita omisiones.

### C. Autorización centralizada por middleware/política

El backend valida token y rol antes de ejecutar el caso de uso.

**Evaluación:** Centraliza la regla y protege cualquier cliente.

## Decisión

Proteger el endpoint/caso de uso de creación de empleados mediante una política de autorización centralizada que exija un token válido con rol Administrador. La interfaz también ocultará la opción a otros roles, pero la decisión de seguridad se aplicará obligatoriamente en el backend.




## Alternativa tecnológica elegida

### Enfoque genérico

Autenticación basada en token y autorización centralizada por roles en backend.

### Tecnología seleccionada

Express con middleware de autenticación/autorización; JWT firmado con expiración; PostgreSQL para usuarios/roles; Angular guards para experiencia de usuario; Docker.

### ¿Por qué se escogió?

JWT funciona bien con una PWA y una API stateless. El middleware de Express protege el backend independientemente del cliente. Los guards de Angular complementan, pero no sustituyen, la autorización del servidor.

### Relación con la reactividad del sistema

El token se valida antes de establecer o usar canales Socket.IO protegidos y antes de ejecutar endpoints.

## Consecuencias

- Positiva: un usuario no autorizado no puede evadir la restricción llamando directamente a la API.
- Positiva: la regla se aplica igual para web y móvil.
- Negativa: requiere mantener correctamente claims/roles del token.
- Negativa: cambios en los permisos exigen actualizar las políticas de autorización.
- Alternativas descartadas: seguridad solo en interfaz y validaciones repetidas por endpoint.

---

# Impedir la modificación libre del TipoEmpleado

- **Título:** Impedir la modificación libre del TipoEmpleado
- **Estado:** Propuesto
- **Escenario:** ESC-CAL-SEG-0011 — Intento de modificación libre de TipoEmpleado

## Contexto

El tipo de empleado determina privilegios dentro del sistema. Permitir cambios libres podría elevar permisos de un usuario sin autorización.

## Alternativas posibles

### A. Permitir editar TipoEmpleado desde cualquier actualización de perfil

El campo se envía con el resto de datos del empleado.

**Evaluación:** Muy simple, pero permite escalamiento de privilegios.

### B. Ocultar el campo para usuarios normales

La interfaz no muestra la opción.

**Evaluación:** No impide modificaciones mediante API.

### C. Separar el cambio de rol en un caso de uso administrativo

Las actualizaciones de perfil ignoran/rechazan TipoEmpleado y solo una operación administrativa puede cambiarlo.

**Evaluación:** Aísla la operación sensible.

## Decisión

Excluir TipoEmpleado de los DTO de actualización ordinaria del perfil. Implementar, si el alcance requiere modificarlo, un caso de uso separado protegido por autorización de Administrador. Cualquier intento de enviar TipoEmpleado mediante una operación de edición común será rechazado por el backend.




## Alternativa tecnológica elegida

### Enfoque genérico

Separar operaciones administrativas sensibles de las actualizaciones ordinarias de perfil.

### Tecnología seleccionada

Express con rutas/controladores separados y middleware JWT por rol; DTOs/esquemas de validación independientes; PostgreSQL; Angular PWA; Docker.

### ¿Por qué se escogió?

Separar endpoints y payloads evita que TipoEmpleado se modifique accidentalmente desde un formulario común. La autorización en Express asegura que solo el administrador pueda acceder.

### Relación con la reactividad del sistema

Los cambios administrativos confirmados pueden notificarse por Socket.IO si deben reflejarse en sesiones o vistas activas.

## Consecuencias

- Positiva: evita escalamiento de privilegios mediante manipulación de solicitudes.
- Positiva: hace explícito y auditable cualquier cambio de rol.
- Negativa: requiere un flujo administrativo separado si el negocio necesita cambiar roles.
- Alternativas descartadas: edición libre y ocultamiento solo visual por no constituir controles de seguridad suficientes.

---

# Consultar productos mediante QR desde la aplicación móvil

- **Título:** Consultar productos mediante QR desde la aplicación móvil
- **Estado:** Propuesto
- **Escenario:** ESC-CAL-USA-0003 — Consulta de producto mediante código válido 

## Contexto

El empleado necesita consultar rápidamente un producto utilizando la cámara del dispositivo móvil como mecanismo de entrada, evitando escribir manualmente el código o nombre, así ahorrando tiempo y proporcionando facilidad.

## Alternativas posibles

### A. Búsqueda manual únicamente

El empleado escribe nombre o código.

**Evaluación:** No requiere cámara, pero aumenta pasos y errores de digitación.

### B. Escaneo QR nativo en la aplicación móvil

La app abre la cámara, obtiene el código y consulta el backend.

**Evaluación:** Reduce pasos y aprovecha hardware disponible.

### C. Lector físico externo

Un dispositivo especializado envía el código.

**Evaluación:** Puede ser preciso, pero agrega hardware y costo.

## Decisión

Integrar lectura QR mediante la cámara del dispositivo dentro de la aplicación móvil. El cliente capturará únicamente el valor del código y lo enviará al endpoint de consulta de producto; la identificación del producto y las reglas de negocio permanecerán en el backend.




## Alternativa tecnológica elegida

### Enfoque genérico

Captura QR desde la PWA y resolución del producto en backend.

### Tecnología seleccionada

Angular PWA usando `getUserMedia`/MediaDevices para cámara y una librería web de lectura QR como `@zxing/browser`; Express para consulta; PostgreSQL; RxJS; Docker.

### ¿Por qué se escogió?

La PWA puede acceder a la cámara desde navegadores compatibles sin crear una app nativa. ZXing permite decodificar QR/códigos de barras en el navegador. La lógica de negocio permanece en Express/PostgreSQL.

### Relación con la reactividad del sistema

La lectura del QR produce un flujo en Angular y RxJS; la consulta va al backend y cambios posteriores del producto pueden recibirse por Socket.IO.

## Consecuencias

- Positiva: reduce digitación y pasos para consultar productos.
- Positiva: mantiene la lógica de identificación en el backend.
- Negativa: depende de permisos y disponibilidad de la cámara.
- Negativa: se debe manejar iluminación, enfoque y códigos no legibles.
- Alternativas descartadas: búsqueda exclusivamente manual por menor operabilidad y lector externo por costo adicional.

---

# Agregar productos a una salida mediante escaneo QR

- **Título:** Agregar productos a una salida mediante escaneo QR
- **Estado:** Propuesto
- **Escenario:** ESC-CAL-USA-0006 — Agregar producto a salida por escaneo

## Contexto

Durante el registro de una salida, el empleado debe poder identificar el producto mediante QR para reducir búsqueda manual. La facilidad de captura no debe omitir las validaciones de stock ni del proyecto.

## Alternativas posibles

### A. El QR agrega directamente la línea y descuenta stock

Al leer el código se ejecuta inmediatamente el movimiento.

**Evaluación:** Muy rápido, pero una lectura accidental podría modificar inventario.

### B. El QR identifica el producto y el usuario confirma cantidad

El escaneo precarga el producto; luego se valida cantidad y se confirma la salida.

**Evaluación:** Equilibra rapidez y control.

### C. El QR solo abre el detalle del producto

El usuario después inicia manualmente el flujo de salida.

**Evaluación:** Seguro, pero añade pasos.

## Decisión

Usar el QR únicamente para identificar y precargar el producto dentro del formulario de salida. El empleado deberá indicar/confirmar la cantidad antes de registrar la línea. El backend validará proyecto y stock; el escaneo nunca modificará inventario por sí solo.




## Alternativa tecnológica elegida

### Enfoque genérico

Escaneo QR para precargar el producto y confirmación explícita antes de modificar inventario.

### Tecnología seleccionada

Angular PWA + `@zxing/browser`; Express; PostgreSQL transaccional; Socket.IO; RxJS; Docker.

### ¿Por qué se escogió?

El escaneo acelera la identificación sin permitir que una lectura accidental modifique stock. Express valida cantidad, proyecto y stock antes del commit.

### Relación con la reactividad del sistema

Tras confirmar la salida, Express publica el nuevo stock por Socket.IO y Angular actualiza las pantallas suscritas.

## Consecuencias

- Positiva: reduce tiempo de búsqueda del producto.
- Positiva: evita movimientos de stock por lecturas accidentales.
- Positiva: conserva las validaciones críticas en el backend.
- Negativa: requiere un paso de confirmación adicional después del escaneo.
- Alternativas descartadas: descuento automático al escanear por riesgo operativo y QR solo informativo por no aprovechar suficientemente la funcionalidad.

---

# Registrar una entrada únicamente contra un pedido válido

- **Título:** Registrar una entrada únicamente contra un pedido válido
- **Estado:** Propuesto
- **Escenario:** ESC-CAL-INTG-0005 — Registro de entrada asociado a pedido válido

## Contexto

Cada entrada debe asociarse obligatoriamente a un pedido existente y registrar sus líneas de producto y cantidades. La entrada incrementa stock y debe conservar la relación con el pedido que la originó.

## Alternativas posibles

### A. Guardar la entrada con un identificador de pedido sin restricción

La aplicación confía en que el ID recibido es válido.

**Evaluación:** Simple, pero puede crear referencias inválidas.

### B. Validar pedido en backend y usar clave foránea

El caso de uso verifica que exista y la base de datos mantiene integridad referencial.

**Evaluación:** Combina validación de negocio y protección estructural.

### C. Copiar los datos del pedido dentro de la entrada

La entrada guarda una réplica de la información relevante.

**Evaluación:** Reduce dependencia para consulta, pero duplica datos y puede divergir.

## Decisión

Modelar Entrada con una clave foránea obligatoria hacia Pedido y validar en el backend que el pedido exista antes de iniciar el registro. Registrar entrada, líneas e incremento de stock dentro de una única transacción. Si el pedido no existe o una línea falla, ejecutar rollback completo.




## Alternativa tecnológica elegida

### Enfoque genérico

Integridad referencial y transacción única para pedido, entrada, líneas y stock, con publicación post-commit.

### Tecnología seleccionada

PostgreSQL con claves foráneas, NOT NULL y transacciones ACID; Express; Socket.IO; Angular PWA con RxJS; Docker.

### ¿Por qué se escogió?

PostgreSQL refuerza la asociación obligatoria con Pedido. Express coordina la transacción. Socket.IO/RxJS propagan el cambio confirmado.

### Relación con la reactividad del sistema

La entrada confirmada genera un evento de inventario después del commit para actualizar el stock en las PWA conectadas.

## Consecuencias

- Positiva: ninguna entrada queda huérfana.
- Positiva: entrada y actualización de stock se confirman juntas.
- Positiva: la base de datos refuerza la regla mediante integridad referencial.
- Negativa: no se podrá eliminar físicamente un pedido que tenga entradas relacionadas sin tratar previamente sus dependencias.
- Alternativas descartadas: referencia sin restricción y duplicación completa del pedido por riesgo de inconsistencia.

---

# Decisión transversal de reactividad de Servistock

## Estado

Propuesto

## Objetivo

Implementar Servistock como una **PWA reactiva**, de manera que los cambios relevantes de inventario, estados y notificaciones puedan propagarse automáticamente a los usuarios conectados sin que deban refrescar manualmente la aplicación.

## Alternativa elegida de forma genérica

Arquitectura web reactiva orientada a eventos:

1. cliente PWA;
2. backend asíncrono;
3. base de datos relacional transaccional;
4. canal de eventos en tiempo real;
5. despliegue contenerizado.

## Tecnología seleccionada

- **Angular** como frontend y PWA.
- **RxJS** para manejo de flujos reactivos en Angular.
- **Express sobre Node.js** como backend.
- **Socket.IO** sobre WebSocket para propagación de eventos en tiempo real.
- **PostgreSQL** como base de datos relacional.
- **Docker** para contenerización y despliegue.
- **JWT** para autenticación stateless.
- **ExcelJS** para generación streaming de archivos XLSX.
- **@zxing/browser** y APIs `MediaDevices/getUserMedia` para lectura de QR/códigos desde la PWA.

## ¿Por qué se escogió este conjunto?

### Angular

Ya es la tecnología definida para el frontend. Permite construir una PWA y RxJS está integrado en su modelo de programación, facilitando suscripciones a cambios de estado y eventos.

### Express / Node.js

Es la tecnología definida para el backend. Node.js utiliza un modelo de I/O asíncrono basado en event loop, adecuado para múltiples conexiones concurrentes y para integrar canales WebSocket.

### PostgreSQL

Las operaciones de inventario requieren consistencia fuerte, transacciones, restricciones e integridad referencial. Por eso la reactividad no reemplaza la transacción de base de datos.

### Socket.IO

Permite establecer comunicación bidireccional entre Express y la PWA y propagar cambios de stock, estados o notificaciones sin refrescar la pantalla.

### Docker

Permite desplegar frontend, backend y servicios con configuraciones reproducibles y facilita health checks y reinicio de componentes.

## Regla arquitectónica principal



Flujo para una entrada o salida:

1. Angular PWA envía la operación a Express.
2. Express valida reglas de negocio.
3. Express inicia una transacción en PostgreSQL.
4. Se actualizan movimiento y stock.
5. PostgreSQL confirma el `COMMIT`.
6. Express emite un evento mediante Socket.IO.
7. Las PWA conectadas reciben el evento.
8. RxJS actualiza los componentes suscritos.

Si la transacción hace `ROLLBACK`, **no se emite el evento de cambio de stock**.

## Observación sobre reactividad

Servistock será reactivo en el sentido de:

- procesamiento asíncrono de I/O en el backend con Node.js;
- propagación de eventos mediante Socket.IO;
- flujos reactivos en Angular mediante RxJS;
- actualización automática de la interfaz.

No se adoptará consistencia eventual para las operaciones críticas de stock: entradas y salidas seguirán utilizando transacciones ACID en PostgreSQL.











