# ADR_MARKDOWN

| Campo | Enunciado |
|-------|-------|
| **Titulo** | consultar productos mediante QR desde la aplicacion movil  |
| **Escenario** | ESC-CAL-USA-0003 |
| **Fecha** | 2 de septiembre de 2026 |
| **Contexto** | El empleado necesita consultar rápidamente(1.5 seg de respuesta) un producto utilizando la cámara del dispositivo móvil como mecanismo de entrada, evitando escribir manualmente el código o nombre, así ahorrando tiempo y proporcionando facilidad. |
| **Decision** | Lectura QR mediante la camara del dispositivo móvil por lo que el usuario capturara el codigo y podra consulta el producto, usaremos PWA usando librería web para lectura del QR, la libreria implementada fue Media device par ala camara luego la libreria para el QR seria @zxing/ngx-scanner o ngx-scanner-qrcode  |
| **Alternativas** |  |
| **consecuencias** | -Positiva: reduce digitación y pasos para consultar productos. -Negativa: depende de permisos y disponibilidad de la cámara. Negativa: se debe manejar iluminación, enfoque y códigos no legibles.|
| **Participantes** |Empleado del inventario y el empleado de campo. |
| **estatus** | |
