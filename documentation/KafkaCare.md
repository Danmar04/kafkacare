# 🏥 Hospital Reactive

## 1. Descripción

**Hospital Reactive** es una plataforma distribuida para la gestión de pacientes en un servicio de urgencias hospitalarias.

El sistema debe ser capaz de recibir pacientes de forma continua, realizar su clasificación inicial mediante un proceso de triaje y gestionar su asignación a los recursos médicos disponibles.

La comunicación entre los diferentes componentes del sistema se realizará mediante una arquitectura **orientada a eventos**, utilizando **Apache Kafka** como sistema de mensajería.

El procesamiento de las operaciones deberá implementarse utilizando **programación reactiva**, basada en **Project Reactor y Spring WebFlux**.

El objetivo principal del proyecto es construir un sistema que permita demostrar el uso práctico de:

- Java 17
- Spring Boot
- Spring WebFlux
- Project Reactor
- Apache Kafka
- Arquitectura orientada a eventos
- Procesamiento asíncrono
- Concurrencia
- Priorización de mensajes
- Idempotencia
- Resiliencia
- R2DBC
- PostgreSQL
- Docker
- Testing reactivo
- Testcontainers

---

# 2. Objetivo

El sistema debe gestionar el ciclo de vida de un paciente desde su llegada al hospital hasta su asignación a un médico.

El flujo principal será:

```text
Paciente llega
     │
     ▼
Recepción
     │
     ▼
Triaje
     │
     ▼
Clasificación de prioridad
     │
     ▼
Cola de espera
     │
     ▼
Disponibilidad de médico
     │
     ▼
Motor de asignación
     │
     ▼
Paciente asignado
     │
     ▼
Atención médica
```

El sistema debe garantizar que los pacientes sean atendidos respetando su prioridad clínica, pero evitando que pacientes con una prioridad inferior puedan permanecer indefinidamente en espera.

---

# 3. Arquitectura conceptual

El sistema estará compuesto inicialmente por varios servicios independientes.

```text
                         ┌──────────────────┐
                         │  Reception API   │
                         │    WebFlux       │
                         └────────┬─────────┘
                                  │
                                  ▼
                         patient.arrived
                                  │
                                  ▼
                         ┌──────────────────┐
                         │  Triage Service  │
                         └────────┬─────────┘
                                  │
                                  ▼
                         patient.triaged
                                  │
                                  ▼
                       ┌─────────────────────┐
                       │    Queue Service    │
                       │                     │
                       │   Priority Queue    │
                       └──────────┬──────────┘
                                  │
                                  │ doctor.available
                                  ▼
                       ┌─────────────────────┐
                       │ Assignment Service  │
                       └──────────┬──────────┘
                                  │
                                  ▼
                           patient.assigned
```

Apache Kafka actuará como **bus de eventos** entre los diferentes componentes.

Kafka no será utilizado directamente como una cola de prioridad. La responsabilidad de determinar qué paciente debe ser atendido corresponderá al servicio encargado de gestionar las colas.

---

# 4. Recepción de pacientes

El sistema deberá proporcionar una API REST que permita registrar la llegada de nuevos pacientes.

### Endpoint

```http
POST /patients
```

### Ejemplo de petición

```json
{
  "name": "John Doe",
  "age": 42,
  "symptoms": [
    "chest pain",
    "shortness of breath"
  ],
  "requiredSpecialty": "CARDIOLOGY"
}
```

La API deberá:

1. Validar la información recibida.
2. Generar un identificador único para el paciente.
3. Registrar la llegada.
4. Crear un evento `patient.arrived`.
5. Publicar el evento en Kafka.
6. Responder al cliente sin esperar a que finalice el procesamiento posterior.

La respuesta deberá ser asíncrona.

Ejemplo:

```json
{
  "patientId": "8e8f2c91",
  "status": "RECEIVED"
}
```

El sistema no deberá esperar a que el paciente sea clasificado ni asignado a un médico para responder a la petición HTTP.

---

# 5. Triaje

El `Triage Service` será responsable de determinar la prioridad clínica del paciente.

El triaje recibirá eventos:

```text
patient.arrived
```

y generará:

```text
patient.triaged
```

## Niveles de prioridad

Se utilizarán inicialmente cinco niveles:

| Nivel | Descripción |
|---|---|
| 1 | Crítico |
| 2 | Muy urgente |
| 3 | Urgente |
| 4 | Poco urgente |
| 5 | No urgente |

El nivel `1` tendrá la máxima prioridad.

El resultado del triaje deberá incluir:

```json
{
  "patientId": "8e8f2c91",
  "priority": 1,
  "requiredSpecialty": "CARDIOLOGY",
  "triagedAt": "2026-10-05T15:30:00Z"
}
```

---

# 6. Gestión de colas

Los pacientes clasificados deberán incorporarse a una cola de espera.

La cola deberá tener en cuenta:

- prioridad clínica;
- tiempo de espera;
- especialidad requerida;
- disponibilidad de recursos.

No existirá necesariamente una única cola global.

El sistema podrá mantener diferentes colas según la especialidad:

```text
                ┌── Cardiology Queue
                │
                ├── Trauma Queue
Patients ───────┼── Pediatrics Queue
                │
                └── General Queue
```

La implementación concreta de las colas queda abierta, pero deberá permitir seleccionar de forma eficiente el siguiente paciente que debe ser atendido.

---

# 7. Motor de priorización

La selección del siguiente paciente será responsabilidad de un **motor de priorización**.

La prioridad no deberá depender exclusivamente del nivel de triaje.

Un paciente con prioridad baja que lleve mucho tiempo esperando deberá poder incrementar progresivamente su prioridad efectiva.

## Prioridad efectiva

Inicialmente se propone una fórmula similar a:

```text
score =
    clinicalPriorityWeight
    + waitingTimeWeight
    + specialtyWeight
```

Por ejemplo:

```text
score =
    gravedad * 1000
    + minutosEsperando * 2
    + factorEspecialidad
```

La fórmula concreta deberá quedar desacoplada del resto del sistema.

---

# 8. Estrategias de priorización

El sistema deberá utilizar un diseño extensible que permita implementar diferentes estrategias de cálculo.

Se propone una abstracción similar a:

```java
public interface PatientPriorityStrategy {

    int calculatePriority(Patient patient);
}
```

Podrán existir diferentes estrategias:

```text
ClinicalPriorityStrategy
WaitingTimePriorityStrategy
EmergencyPriorityStrategy
PediatricPriorityStrategy
```

También deberá ser posible combinar diferentes estrategias mediante una estrategia compuesta:

```text
CompositePriorityStrategy
```

El objetivo es poder modificar las reglas de priorización sin modificar el motor de asignación.

---

# 9. Disponibilidad de médicos

Los médicos y otros recursos hospitalarios deberán poder registrarse como disponibles.

Ejemplo:

```http
POST /doctors/{doctorId}/availability
```

La disponibilidad deberá generar un evento:

```text
doctor.available
```

El evento deberá contener información suficiente para determinar qué pacientes puede atender el recurso.

Ejemplo:

```json
{
  "doctorId": "doctor-123",
  "specialties": [
    "CARDIOLOGY"
  ],
  "availableAt": "2026-10-05T15:45:00Z"
}
```

---

# 10. Asignación de pacientes

Cuando un médico quede disponible, el sistema deberá buscar el paciente elegible con mayor prioridad.

Ejemplo:

```text
             doctor.available
                    │
                    ▼
          ┌───────────────────┐
          │ Assignment Service │
          └─────────┬─────────┘
                    │
                    ▼
             Buscar pacientes
                    │
                    ▼
          Filtrar por especialidad
                    │
                    ▼
            Calcular prioridad
                    │
                    ▼
             Seleccionar uno
                    │
                    ▼
             Asignar paciente
```

Una vez realizada la asignación se publicará:

```text
patient.assigned
```

Ejemplo:

```json
{
  "patientId": "8e8f2c91",
  "doctorId": "doctor-123",
  "specialty": "CARDIOLOGY",
  "assignedAt": "2026-10-05T15:46:00Z"
}
```

---

# 11. Concurrencia

El sistema deberá ser capaz de gestionar situaciones en las que varios recursos estén disponibles simultáneamente.

Por ejemplo:

```text
doctor.available
        │
        ├──────────────► Assignment Service
        │
doctor.available
        │
        └──────────────► Assignment Service
```

Ambos procesos podrían intentar asignar al mismo paciente.

El sistema deberá garantizar que un paciente **no pueda ser asignado simultáneamente a dos médicos**.

Para ello deberá utilizar mecanismos de:

- control de concurrencia;
- operaciones atómicas;
- idempotencia;
- control del estado del paciente.

Este comportamiento será una de las partes principales del proyecto.

---

# 12. Idempotencia

Los eventos de Kafka podrán ser entregados más de una vez.

Por tanto, los consumidores deberán ser capaces de procesar un mismo evento de forma segura varias veces.

Por ejemplo:

```text
patient.triaged
      │
      ├── procesamiento
      │
      └── mensaje duplicado
```

El segundo procesamiento no deberá provocar:

- una segunda asignación;
- una segunda reserva;
- una transición de estado inválida.

Los eventos deberán disponer de un identificador único que permita detectar duplicados.

---

# 13. Estados del paciente

El paciente tendrá un ciclo de vida controlado.

Inicialmente se propone:

```text
RECEIVED
   │
   ▼
TRIAGED
   │
   ▼
WAITING
   │
   ▼
ASSIGNED
   │
   ▼
IN_TREATMENT
   │
   ▼
COMPLETED
```

También podrán existir estados de error o cancelación:

```text
CANCELLED
ERROR
```

Las transiciones deberán estar controladas para evitar estados inconsistentes.

---

# 14. Eventos Kafka

Inicialmente se utilizarán los siguientes eventos:

| Evento | Productor | Consumidores |
|---|---|---|
| `patient.arrived` | Reception Service | Triage Service |
| `patient.triaged` | Triage Service | Queue Service |
| `doctor.available` | Doctor Service | Assignment Service |
| `patient.assigned` | Assignment Service | Patient Service |
| `patient.completed` | Treatment Service | Patient Service |

Los nombres y número de eventos podrán evolucionar durante el desarrollo.

---

# 15. Programación reactiva

El sistema deberá utilizar **Spring WebFlux y Project Reactor**.

Los endpoints REST utilizarán:

```java
Mono<T>
```

y:

```java
Flux<T>
```

Las operaciones de entrada/salida deberán evitar llamadas bloqueantes.

El acceso a la base de datos deberá realizarse mediante **R2DBC**.

El objetivo es mantener el flujo:

```text
HTTP
 ↓
WebFlux
 ↓
Reactor
 ↓
R2DBC
 ↓
PostgreSQL
```

sin introducir operaciones bloqueantes dentro del flujo reactivo.

---

# 16. Procesamiento reactivo de recursos

Para demostrar el uso de programación reactiva, algunos servicios podrán consultar diferentes recursos simultáneamente.

Por ejemplo, el hospital podría disponer de diferentes almacenes de material médico:

```text
                 ┌── Pharmacy A
                 │
Request ─────────┼── Pharmacy B
                 │
                 └── Pharmacy C
```

Las consultas deberán realizarse de forma concurrente mediante Reactor:

```java
Flux.merge(
    pharmacyA.checkAvailability(),
    pharmacyB.checkAvailability(),
    pharmacyC.checkAvailability()
);
```

Este escenario permitirá demostrar el uso de:

- `Mono`
- `Flux`
- `merge`
- `flatMap`
- `filter`
- `map`
- `timeout`
- `retryWhen`
- manejo reactivo de errores

---

# 17. Resiliencia

El sistema deberá gestionar errores producidos durante el procesamiento.

Se contemplarán inicialmente:

- errores de comunicación;
- timeouts;
- errores temporales;
- mensajes duplicados;
- errores de procesamiento;
- servicios no disponibles.

Se deberán implementar mecanismos de:

```text
Retry
Timeout
Error handling
Dead Letter Topics
```

Los mensajes que no puedan procesarse después de un número determinado de intentos deberán enviarse a una **Dead Letter Topic (DLT)**.

---

# 18. Persistencia

Se utilizará:

- PostgreSQL
- R2DBC

La base de datos almacenará, como mínimo:

### Pacientes

```text
Patient
- id
- name
- age
- status
- priority
- requiredSpecialty
- createdAt
- updatedAt
```

### Médicos

```text
Doctor
- id
- name
- specialties
- status
```

### Eventos procesados

```text
ProcessedEvent
- eventId
- eventType
- processedAt
```

Esta última entidad podrá utilizarse para implementar mecanismos de idempotencia.

---

# 19. Testing

El proyecto deberá disponer de diferentes niveles de pruebas.

## Tests unitarios

Se probarán de forma aislada:

- reglas de priorización;
- estrategias;
- cambios de estado;
- validaciones;
- lógica de asignación.

## Tests reactivos

Se utilizará `StepVerifier` para comprobar los flujos de Reactor.

Ejemplo conceptual:

```java
StepVerifier
    .create(service.findNextPatient())
    .expectNext(expectedPatient)
    .verifyComplete();
```

## Tests de integración

Se utilizará **Testcontainers** para levantar:

- Kafka;
- PostgreSQL.

Los tests deberán comprobar flujos completos entre los diferentes componentes.

---

# 20. Docker

El proyecto deberá poder iniciarse mediante:

```bash
docker compose up
```

Como mínimo se levantarán:

```text
Kafka
PostgreSQL
```

Opcionalmente:

```text
Kafka UI
```

para poder inspeccionar:

- topics;
- particiones;
- mensajes;
- consumidores;
- consumer groups.

---

# 21. Requisitos no funcionales

El sistema deberá:

- utilizar Java 17;
- utilizar Spring Boot;
- utilizar programación reactiva;
- evitar operaciones bloqueantes;
- utilizar Kafka para la comunicación entre servicios;
- soportar mensajes duplicados;
- gestionar errores y reintentos;
- soportar procesamiento concurrente;
- garantizar que un paciente no sea asignado dos veces;
- permitir modificar las reglas de priorización sin modificar el motor de asignación;
- disponer de tests automatizados;
- poder ejecutarse localmente mediante Docker Compose.

---

# 22. Objetivo técnico del proyecto

El objetivo de **Hospital Reactive** no es construir una aplicación hospitalaria real, sino disponer de un proyecto demostrativo que permita explorar y mostrar conceptos de arquitectura y desarrollo backend moderno.

Especialmente:

```text
Java 17
   │
   ├── Spring Boot
   │
   ├── Spring WebFlux
   │       │
   │       └── Project Reactor
   │
   ├── Apache Kafka
   │       │
   │       └── Event-Driven Architecture
   │
   ├── R2DBC
   │       │
   │       └── PostgreSQL
   │
   ├── Concurrency
   │
   ├── Idempotency
   │
   ├── Resilience
   │
   └── Testing
           ├── JUnit 5
           ├── StepVerifier
           └── Testcontainers
```

El proyecto deberá priorizar **decisiones técnicas justificadas** frente a la complejidad innecesaria.

No se busca crear el mayor número posible de microservicios, clases o abstracciones, sino construir un sistema pequeño pero suficientemente complejo como para demostrar cómo resolver problemas reales de **procesamiento asíncrono, concurrencia, priorización y consistencia distribuida**.