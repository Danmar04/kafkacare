# 🎯 Queuepital — Objetivos del proyecto

## 1. Objetivo general

**Queuepital** es un proyecto personal cuyo objetivo es construir, desde cero, un sistema distribuido de gestión de pacientes en urgencias utilizando tecnologías y patrones habituales en arquitecturas backend modernas.

El proyecto tendrá una doble finalidad:

1. **Construir una aplicación funcional** con un problema suficientemente complejo como para justificar programación reactiva, mensajería asíncrona, concurrencia y procesamiento basado en eventos.
2. **Servir como proyecto de aprendizaje y portfolio**, permitiendo practicar y demostrar conocimientos técnicos que puedan ser defendidos posteriormente en entrevistas técnicas.

La prioridad del proyecto será **comprender y justificar las decisiones técnicas**, evitando introducir complejidad arquitectónica que no aporte valor.

---

# 2. Objetivos tecnológicos

El proyecto deberá permitir trabajar con las siguientes tecnologías:

| Tecnología | Objetivo |
|---|---|
| Java 17 | Utilizar características modernas de Java |
| Spring Boot | Construcción de los servicios |
| Spring WebFlux | APIs HTTP no bloqueantes |
| Project Reactor | Programación reactiva |
| Apache Kafka | Comunicación asíncrona basada en eventos |
| PostgreSQL | Persistencia relacional |
| R2DBC | Acceso reactivo a PostgreSQL |
| Docker | Ejecución de infraestructura local |
| JUnit 5 | Testing unitario |
| StepVerifier | Testing de flujos reactivos |
| Testcontainers | Tests de integración |
| Maven | Gestión y construcción del proyecto |

---

# 3. Objetivos de aprendizaje

## 3.1. Programación reactiva

Profundizar en el modelo de programación reactiva utilizando **Project Reactor**.

Practicar especialmente:

```text
Mono
Flux
map
flatMap
filter
switchIfEmpty
merge
zip
timeout
retryWhen
onErrorResume
```

El objetivo no será utilizar Reactor por obligación, sino aprender a identificar situaciones donde la programación reactiva aporta ventajas reales.

Especialmente:

- operaciones I/O;
- operaciones concurrentes;
- composición de operaciones asíncronas;
- gestión de errores;
- timeouts;
- reintentos;
- procesamiento de streams.

---

# 4. Spring WebFlux

Construir las APIs REST utilizando **Spring WebFlux**.

Se pretende comprender:

- diferencias entre Spring MVC y WebFlux;
- modelo de ejecución reactivo;
- uso de `Mono` y `Flux` en endpoints;
- manejo de errores;
- validación;
- programación no bloqueante;
- consecuencias de introducir operaciones bloqueantes en un flujo reactivo.

El proyecto deberá servir para poder explicar claramente:

> ¿Cuándo utilizaría WebFlux y cuándo no?

---

# 5. Kafka y arquitectura orientada a eventos

Utilizar **Apache Kafka** como mecanismo de comunicación entre los diferentes componentes del sistema.

Se pretende comprender de forma práctica:

- topics;
- partitions;
- producers;
- consumers;
- consumer groups;
- offsets;
- orden de los mensajes;
- distribución de mensajes;
- procesamiento asíncrono;
- eventos de dominio;
- reintentos;
- Dead Letter Topics.

El objetivo es poder responder con conocimiento práctico a preguntas como:

> ¿Por qué utilizar Kafka en lugar de una llamada REST?

> ¿Qué ocurre si un mensaje se procesa dos veces?

> ¿Qué garantiza Kafka respecto al orden?

> ¿Qué ocurre si un consumidor falla?

---

# 6. Arquitectura orientada a eventos

El sistema deberá evolucionar progresivamente desde una aplicación sencilla hasta una arquitectura distribuida.

Evolución prevista:

```text
Fase inicial

REST
 ↓
Service
 ↓
Database
```

Posteriormente:

```text
REST
 ↓
Service
 ↓
Kafka
 ↓
Consumer
```

Y finalmente:

```text
Reception
    │
    ▼
  Kafka
    │
    ▼
 Triage
    │
    ▼
  Kafka
    │
    ▼
  Queue
    │
    ▼
  Kafka
    │
    ▼
Assignment
```

El objetivo es comprender las ventajas y problemas que introduce una arquitectura orientada a eventos.

---

# 7. R2DBC y persistencia reactiva

Utilizar **R2DBC** para acceder a PostgreSQL.

El objetivo es entender por qué utilizar una tecnología de persistencia reactiva cuando el resto de la aplicación utiliza WebFlux.

Se deberá poder comparar conceptualmente:

```text
WebFlux
   ↓
JPA / JDBC
   ↓
Operación bloqueante
```

frente a:

```text
WebFlux
   ↓
R2DBC
   ↓
PostgreSQL
```

Se estudiarán especialmente:

- transacciones;
- consultas reactivas;
- `Mono` / `Flux`;
- limitaciones de R2DBC;
- diferencias respecto a JPA/Hibernate.

---

# 8. Concurrencia

Uno de los objetivos principales de Queuepital será estudiar problemas reales de concurrencia.

El escenario principal será:

```text
        Pacientes
            │
            ▼
      Priority Queue
            │
      ┌─────┴─────┐
      ▼           ▼
   Doctor A    Doctor B
```

Dos médicos podrían intentar obtener simultáneamente el mismo paciente.

El sistema deberá garantizar:

```text
Patient X
   │
   └──► Doctor A
```

y nunca:

```text
Patient X
   ├──► Doctor A
   └──► Doctor B
```

Se estudiarán:

- race conditions;
- operaciones atómicas;
- locking;
- optimistic locking;
- estados;
- concurrencia distribuida;
- idempotencia.

---

# 9. Sistema de prioridades

Queuepital tendrá un sistema de priorización suficientemente complejo como para estudiar diferentes estrategias.

La prioridad deberá tener en cuenta factores como:

```text
Gravedad clínica
      +
Tiempo de espera
      +
Especialidad
```

El sistema deberá evitar tanto:

### Problema 1 — ignorar prioridades

```text
Paciente crítico
       ↓
esperando indefinidamente
```

como:

### Problema 2 — starvation

```text
Paciente prioridad 5
       ↓
nunca es atendido porque continuamente
llegan pacientes de prioridad superior
```

El proyecto permitirá estudiar cómo diseñar una política de prioridad que equilibre ambos problemas.

---

# 10. Strategy Pattern y diseño extensible

Las reglas de priorización deberán estar desacopladas del motor de asignación.

Se utilizará una abstracción similar a:

```java
public interface PatientPriorityStrategy {

    int calculate(Patient patient);
}
```

Esto permitirá implementar diferentes estrategias:

```text
ClinicalPriorityStrategy
WaitingTimePriorityStrategy
SpecialtyPriorityStrategy
CompositePriorityStrategy
```

El objetivo será practicar:

- Strategy Pattern;
- composición;
- SOLID;
- inversión de dependencias;
- separación de responsabilidades.

No se pretende utilizar patrones simplemente por utilizarlos. Cada abstracción deberá justificar su existencia.

---

# 11. Idempotencia

El sistema deberá asumir que un mismo evento puede recibirse más de una vez.

Por ejemplo:

```text
patient.assigned
       │
       ├── mensaje recibido
       │
       └── mensaje duplicado
```

El resultado final deberá ser el mismo que si el evento solamente se hubiese procesado una vez.

Se estudiarán:

- identificadores de eventos;
- eventos procesados;
- estados;
- operaciones idempotentes;
- duplicados de Kafka;
- consistencia.

---

# 12. Resiliencia

El sistema deberá ser capaz de recuperarse de errores.

Se trabajará con:

```text
Timeout
Retry
Dead Letter Topic
Error handling
```

Se estudiará especialmente la diferencia entre:

```text
Error temporal
```

y:

```text
Error permanente
```

No todos los errores deberán provocar un `retry`.

---

# 13. Testing

El proyecto tendrá una estrategia de testing completa.

### Unit testing

Probar de forma aislada:

- reglas de negocio;
- estrategias;
- prioridades;
- transiciones de estado;
- validaciones.

### Reactive testing

Utilizar:

```java
StepVerifier
```

para probar los flujos reactivos.

### Integration testing

Utilizar **Testcontainers** para ejecutar:

```text
Kafka
PostgreSQL
```

durante las pruebas.

El objetivo es poder probar escenarios completos y no depender únicamente de mocks.

---

# 14. Docker

Toda la infraestructura necesaria para ejecutar el proyecto localmente deberá poder levantarse mediante Docker.

Como mínimo:

```text
Kafka
PostgreSQL
```

Opcionalmente:

```text
Kafka UI
```

El objetivo es que una persona pueda clonar el repositorio y comenzar a trabajar sin tener que instalar manualmente toda la infraestructura.

---

# 15. Evolución incremental

El proyecto se desarrollará de forma incremental.

No se comenzará directamente con todos los microservicios.

## Fase 1 — Proyecto base

```text
Java 17
Spring Boot
Maven
WebFlux
JUnit 5
```

Primer objetivo:

```text
POST /patients
```

---

## Fase 2 — Programación reactiva

Introducir:

```text
Mono
Flux
map
flatMap
filter
error handling
```

y crear los primeros flujos reactivos reales.

---

## Fase 3 — Persistencia

Introducir:

```text
PostgreSQL
R2DBC
```

y persistencia reactiva de pacientes.

---

## Fase 4 — Kafka

Introducir:

```text
Kafka
Producer
Consumer
Topics
Consumer Groups
```

Primer flujo:

```text
Reception
   ↓
patient.arrived
   ↓
Triage
```

---

## Fase 5 — Arquitectura distribuida

Separar progresivamente:

```text
Reception Service
Triage Service
Queue Service
Assignment Service
```

---

## Fase 6 — Prioridades

Implementar:

```text
Priority Queue
Priority Strategies
Waiting Time
Clinical Priority
Specialties
```

---

## Fase 7 — Concurrencia

Resolver:

```text
Doctor A ─┐
          ├──► Patient X
Doctor B ─┘
```

garantizando una única asignación.

---

## Fase 8 — Resiliencia

Introducir:

```text
Retry
Timeout
DLT
Idempotency
```

---

## Fase 9 — Testing avanzado

Introducir:

```text
StepVerifier
Testcontainers
Kafka integration tests
R2DBC integration tests
```

---

## Fase 10 — Docker y documentación

Preparar:

```text
docker-compose.yml
README.md
Architecture documentation
Functional requirements
Technical decisions
```

El objetivo final será que una persona pueda:

```bash
git clone <repository>
docker compose up
mvn test
```

y comenzar a explorar el proyecto.

---

# 16. Objetivo final

Al finalizar el proyecto, Queuepital deberá ser capaz de demostrar de forma práctica conocimientos sobre:

```text
                 Queuepital
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
      Java 17      Spring       Kafka
                     │            │
                  WebFlux      Events
                     │            │
                  Reactor      Async
                     │            │
        ┌────────────┴────────────┘
        ▼
    R2DBC + PostgreSQL
        │
        ▼
   Concurrency
        │
        ▼
   Idempotency
        │
        ▼
    Resilience
        │
        ▼
     Testing
```

El proyecto deberá ser lo suficientemente pequeño como para poder comprenderlo completamente, pero lo suficientemente complejo como para enfrentarse a problemas reales de sistemas distribuidos.

La complejidad deberá aparecer **porque el problema lo necesita**, no porque se hayan añadido tecnologías o patrones artificialmente.

---

# 17. Criterio de éxito

Queuepital se considerará terminado cuando no solo funcione, sino cuando sea posible explicar y defender las principales decisiones técnicas del proyecto.

Especialmente:

- ¿Por qué WebFlux?
- ¿Por qué Reactor?
- ¿Por qué Kafka?
- ¿Por qué R2DBC?
- ¿Por qué arquitectura orientada a eventos?
- ¿Cómo se gestiona la prioridad?
- ¿Cómo se evita el starvation?
- ¿Cómo se evita asignar un paciente dos veces?
- ¿Cómo se gestionan mensajes duplicados?
- ¿Qué ocurre cuando un consumidor falla?
- ¿Cómo funcionan los retries?
- ¿Cuándo se utiliza una DLT?
- ¿Cómo se garantiza la consistencia?
- ¿Dónde existen problemas de concurrencia?
- ¿Cómo se prueban los flujos reactivos?
- ¿Cómo se prueban Kafka y PostgreSQL?
- ¿Qué partes del sistema son realmente reactivas?
- ¿Qué partes podrían simplificarse?

El objetivo final no es únicamente tener un repositorio que "funcione", sino poder utilizar **Queuepital como demostración práctica de conocimientos de backend, programación reactiva, sistemas distribuidos y diseño de software**.