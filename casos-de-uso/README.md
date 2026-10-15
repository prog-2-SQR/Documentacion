# Casos de uso

Casos de uso del sistema, agrupados por el servicio responsable. Se derivan del alcance funcional obligatorio (enunciado §5) y de los contratos del anexo de integración. La app KMP es el cliente de ambos servicios: sus pantallas implementan la parte visible de estos casos.

- [Servicio de catálogo](catalogo.md): `CU-CAT-01` a `CU-CAT-06`
- [Servicio de turnos](turnos.md): `CU-TUR-01` a `CU-TUR-10`

## Actores

| Actor | Descripción |
|---|---|
| Usuario final | Persona que usa la app Android para buscar profesionales y reservar turnos. Tiene cuenta propia en Backend-Turnos. |
| Alumno (operador) | Registra una única vez la cuenta técnica del proyecto ante la cátedra (Postman) y configura los secretos. |
| Servicio de la cátedra | Fuente del catálogo (REST, Redis, Kafka) y responsable de holds, confirmaciones y reservas centrales. |
| Backend-Turnos | Además de servicio propio, es consumidor de Backend-Catalogo para obtener la información vigente de un profesional. |

## Casos compartidos (integración técnica)

Los dos backends usan la misma cuenta técnica del proyecto para hablar con la cátedra.

| ID | Caso | Actor | Referencias |
|---|---|---|---|
| CU-INT-01 | Registrar la cuenta técnica del proyecto (una sola vez, fuera de los backends) | Alumno | Enunciado §5.2 · Anexo §5.1 |
| CU-INT-02 | Autenticar la cuenta técnica y recuperar la configuración (JWT, Redis, Kafka, topics) | Backends | Enunciado §5.3 · Anexo §5.2, §5.3 |

El JWT técnico nunca sale de los backends: no se entrega a la app ni aparece en logs o en el repositorio (anexo §5.1).

## Resumen y planificación

| ID | Caso | Servicio | Milestone |
|---|---|---|---|
| CU-CAT-01 | Inicializar o reconstruir el catálogo desde snapshot | Catálogo | M2 |
| CU-CAT-02 | Aplicar cambios incrementales | Catálogo | M3 |
| CU-CAT-03 | Detectar discontinuidad y resincronizar | Catálogo | M3 |
| CU-CAT-04 | Buscar profesionales con filtros locales | Catálogo | M4 |
| CU-CAT-05 | Proveer profesional y agenda vigente a Turnos | Catálogo | M4 |
| CU-CAT-06 | Informar el estado de sincronización | Catálogo | M4 |
| CU-TUR-01 | Registrar usuario | Turnos | M5 |
| CU-TUR-02 | Iniciar sesión y obtener JWT | Turnos | M5 |
| CU-TUR-03 | Consultar disponibilidad de un profesional | Turnos | M6 |
| CU-TUR-04 | Iniciar reserva (crear hold) | Turnos | M6 |
| CU-TUR-05 | Confirmar inicialmente el hold | Turnos | M6 |
| CU-TUR-06 | Enviar el teléfono solicitado | Turnos | M6 |
| CU-TUR-07 | Procesar el resultado final de la reserva | Turnos | M6 |
| CU-TUR-08 | Consultar mis reservas | Turnos | M7 |
| CU-TUR-09 | Cancelar una reserva propia | Turnos | M7 |
| CU-TUR-10 | Recuperar procesos pendientes | Turnos | M7 |
