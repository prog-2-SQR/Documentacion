# Casos de uso: servicio de turnos

Responsabilidades del servicio según el enunciado §4.2: gestionar los usuarios finales, construir la disponibilidad, llevar adelante el proceso de reserva con la cátedra y conservar el estado local de cada proceso asociado a su usuario.

Regla transversal: **cada proceso y cada reserva pertenecen al usuario que los inició**. El usuario se toma siempre del JWT, nunca de un dato enviado por el cliente (enunciado §9).

---

## CU-TUR-01: Registrar usuario

- **Actor:** usuario final.
- **Flujo principal:**
  1. Envía sus datos a `POST /api/register` (modelo y validaciones de usuario de JHipster).
  2. El servicio valida, guarda la contraseña con hash y crea la cuenta.
- **Alternativas y errores:** login o email ya usados, datos inválidos: `400` con `application/problem+json`.
- **Referencias:** enunciado §5.1, §9 · anexo (modelo de usuario compatible con JHipster).

## CU-TUR-02: Iniciar sesión y obtener JWT

- **Actor:** usuario final.
- **Flujo principal:**
  1. Envía credenciales a `POST /api/authenticate`.
  2. Recibe `{ "id_token": ... }`, que la app usa en las llamadas a ambos servicios.
- **Alternativas y errores:** credenciales incorrectas: `401`, sin revelar si falló el usuario o la contraseña.
- **Referencias:** enunciado §5.1, §9.

## CU-TUR-03: Consultar disponibilidad de un profesional

- **Actor:** usuario final.
- **Flujo principal:**
  1. Elige profesional y fecha.
  2. Turnos obtiene de Catálogo los horarios semanales vigentes (CU-CAT-05).
  3. Consulta las ocupaciones (`CONFIRMED` y `HELD`) en la cátedra.
  4. Devuelve los slots libres.
- **Alternativas y errores:** Catálogo no disponible: no se puede calcular la disponibilidad y se informa el error.
- **Referencias:** enunciado §5.8, §7, §8 · anexo §8.

## CU-TUR-04: Iniciar reserva (crear hold)

- **Actor:** usuario final.
- **Flujo principal:**
  1. Elige un slot disponible.
  2. Turnos valida con Catálogo la información vigente del profesional.
  3. Crea el hold en la cátedra y guarda el proceso local (`holdId`, `reservationProcessId`, `expiresAt`) asociado al usuario.
- **Alternativas y errores:** slot ya ocupado o datos inválidos: error de la cátedra traducido a `problem+json`.
- **Referencias:** enunciado §5.9, §7 · anexo §9.

## CU-TUR-05: Confirmar inicialmente el hold

- **Actor:** usuario final.
- **Flujo principal:**
  1. Turnos llama a `POST /api/appointment-holds/{holdId}/confirm` con los datos del paciente.
  2. La cátedra responde `202` y el proceso queda esperando el teléfono.
- **Alternativas y errores:** hold vencido (`HOLD_EXPIRED`): el proceso se cierra como vencido.
- **Referencias:** enunciado §5.10 · anexo §10, §18.3.

## CU-TUR-06: Enviar el teléfono solicitado

- **Actor:** servicio de la cátedra (pide) y usuario final (responde).
- **Disparador:** evento `AdditionalInformationRequested` por Kafka.
- **Flujo principal:**
  1. Turnos registra la solicitud (guarda su `eventId`) y la app le pide el teléfono al usuario.
  2. El usuario informa el teléfono.
  3. Turnos publica `AdditionalInformationSubmitted` con `requestEventId` igual al `eventId` recibido, usando **outbox**: se guarda en la base y se publica después; los reintentos conservan el mismo `eventId`.
- **Alternativas y errores:** teléfono rechazado (`AdditionalInformationRejected`): el usuario puede reintentar hasta `expiresAt`.
- **Referencias:** enunciado §5.11 · anexo §15.4, §15.5, §15.7.

## CU-TUR-07: Procesar el resultado final de la reserva

- **Actor:** servicio de la cátedra (vía Kafka).
- **Disparador:** `AppointmentConfirmed`, `AppointmentProcessExpired` o `AppointmentProcessInvalid`.
- **Flujo principal:** actualiza el estado local del proceso y lo informa a la app.
- **Alternativas y errores:**
  - Evento duplicado (mismo `eventId`): se ignora sin repetir efectos.
  - Evento tardío: un estado final nunca retrocede.
- **Referencias:** enunciado §5.12, §7, §8 · anexo §15.6, §15.8, §15.9, §18.1.

## CU-TUR-08: Consultar mis reservas

- **Actor:** usuario final.
- **Flujo principal:** devuelve, paginadas, solo las reservas y procesos del usuario autenticado.
- **Alternativas y errores:** si la cátedra no responde, se muestran los datos locales.
- **Referencias:** enunciado §5.13, §9 · anexo §11.

## CU-TUR-09: Cancelar una reserva propia

- **Actor:** usuario final.
- **Flujo principal:**
  1. Pide cancelar una reserva confirmada propia.
  2. Turnos verifica que le pertenezca y llama a `POST /api/appointments/{reservationProcessId}/cancel`.
  3. Refleja la cancelación localmente (también llega `AppointmentCancelled` por Kafka).
- **Alternativas y errores:** reserva de otro usuario: `404` o `403`, sin revelar que existe.
- **Referencias:** enunciado §5.14, §9 · anexo §12, §15.10.

## CU-TUR-10: Recuperar procesos pendientes

- **Actor:** el propio servicio.
- **Disparador:** reinicio del servicio, timeout o respuesta perdida.
- **Flujo principal:**
  1. Busca procesos locales que no llegaron a un estado final.
  2. Reconcilia su estado con la cátedra antes de repetir cualquier acción con efectos.
  3. Cierra los vencidos y reintenta, de forma limitada, lo que quedó pendiente (por ejemplo, el outbox).
- **Referencias:** enunciado §8 · anexo §18.3, §18.4.
