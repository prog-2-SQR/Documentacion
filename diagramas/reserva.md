# Reserva de un turno

Proceso completo desde que el usuario elige un horario hasta el resultado final. Cubre CU-TUR-03 a CU-TUR-07.

Identidades: la app habla con Backend-Turnos usando el **JWT del usuario**; Turnos habla con la cátedra usando el **JWT técnico**, que nunca llega a la app. Turnos asocia cada proceso al usuario que lo inició, porque la cátedra no conoce a los usuarios finales (anexo §2).

## Secuencia

```mermaid
sequenceDiagram
    actor U as Usuario
    participant APP as App KMP
    participant T as Backend-Turnos
    participant DBT as PostgreSQL turnos
    participant C as Backend-Catalogo
    participant API as API REST cátedra
    participant KA as Kafka acciones
    participant KP as Kafka teléfono

    Note over U,API: Disponibilidad (CU-TUR-03)
    U->>APP: elige profesional y fecha
    APP->>T: GET disponibilidad (JWT usuario)
    T->>C: GET profesional y agenda vigente (JWT)
    C-->>T: horarios semanales
    T->>API: GET /api/appointment-occupancies
    API-->>T: horarios CONFIRMED y HELD
    T-->>APP: slots libres

    Note over U,API: Hold y confirmación inicial (CU-TUR-04 y 05)
    U->>APP: elige un horario
    APP->>T: POST reserva (JWT usuario)
    T->>C: validar profesional vigente
    T->>API: POST /api/appointment-holds
    API-->>T: holdId, reservationProcessId, expiresAt
    T->>DBT: guardar proceso asociado al usuario
    T->>API: POST /api/appointment-holds/{holdId}/confirm
    API-->>T: 202 WAITING_FOR_PHONE
    T->>DBT: proceso esperando teléfono

    Note over U,KP: Teléfono (CU-TUR-06)
    KA-->>T: AdditionalInformationRequested (eventId)
    T->>DBT: guardar eventId de la solicitud
    T-->>APP: pedir teléfono (SSE)
    U->>APP: ingresa teléfono
    APP->>T: enviar teléfono
    T->>DBT: guardar evento en el outbox (misma transacción)
    T->>KP: publicar AdditionalInformationSubmitted
    Note over T,KP: si falla, se reintenta con el mismo eventId

    Note over U,KA: Resultado (CU-TUR-07)
    alt teléfono y proceso válidos
        KA-->>T: AppointmentConfirmed
        T->>DBT: CONFIRMED (estado final)
    else teléfono rechazado
        KA-->>T: AdditionalInformationRejected
        T->>DBT: vuelve a esperar teléfono hasta expiresAt
    else vencido o inválido
        KA-->>T: AppointmentProcessExpired o AppointmentProcessInvalid
        T->>DBT: EXPIRED o INVALID (estado final)
    end
    T-->>APP: resultado (SSE)
```

## Reglas

| Situación | Qué hace el servicio | Referencia |
|---|---|---|
| Mensaje Kafka repetido (mismo `eventId`) | Lo reconoce como procesado, no repite efectos y confirma el offset | Anexo §18.1 |
| Evento que llega tarde sobre un proceso ya finalizado | Lo ignora: un estado final nunca retrocede | Enunciado §7 |
| Kafka no confirma la publicación del teléfono | El outbox reintenta con el **mismo** `eventId`; la cátedra ignora duplicados | Decisión confirmada por el profesor en Slack |
| Kafka confirma la recepción | No significa reserva confirmada: eso solo lo dice `AppointmentConfirmed` | Decisión confirmada por el profesor en Slack |
| Timeout en una llamada REST con efectos | Antes de repetirla, consulta el estado con los IDs guardados | Anexo §18.4 |
| Hold vencido (`HOLD_EXPIRED` o `AppointmentProcessExpired`) | Cierra el proceso como vencido y no vuelve a intentar confirmarlo | Anexo §18.3 |
| Catálogo no disponible | No inicia la reserva: no puede validar la información vigente | Enunciado §8 |

Los nombres de los estados locales son tentativos: la máquina de estados se define en M6.
