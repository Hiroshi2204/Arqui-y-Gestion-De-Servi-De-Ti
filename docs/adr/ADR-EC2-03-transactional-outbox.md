# ADR-EC2-03 — Patrón Transactional Outbox

**Proyecto:** PagaEdu — Pasarela de cobranzas recurrentes (Dominio 4: Recaudación y Pagos)
**Curso:** Arquitectura y Gestión de Servicios TI — EC3

## Estado

Aceptada.

## Contexto

Una vez resuelta la idempotencia del cargo (ADR-EC2-02), persiste el riesgo de que el cargo quede registrado pero el evento de dominio correspondiente (p. ej. `PagoIniciado`) no se publique, o viceversa, si se escribe en la base de datos y se publica al bus como dos operaciones independientes ("dual write").

## Decisión

Registrar el cargo y su evento de dominio en una **misma transacción local ACID** (tabla Outbox), publicado luego de forma asíncrona por un relay (Outbox Publisher/Relay) hacia el bus de eventos.

## Alternativas evaluadas

- **a) Transacción distribuida (2PC) entre la base de datos y el bus de eventos:** se descarta por la complejidad de coordinación y el acoplamiento fuerte entre infraestructuras heterogéneas; además, los agregadores de pago y brokers habituales no exponen un coordinador XA/2PC confiable.
- **b) Publicar el evento directamente después del commit, sin tabla Outbox ("dual write"):** se descarta porque si el proceso falla entre el commit y la publicación, el evento se pierde de forma silenciosa.

## Consecuencias

### Positivas

- Garantiza atomicidad entre "cobrar" y "avisar que se cobró" sin acoplar infraestructuras heterogéneas.
- Deja trazabilidad de qué eventos se derivaron de cada cobro, útil para auditoría y para reconstruir el estado ante incidentes.

### Negativas (explícitas)

- Existe un pequeño desfase entre el cargo confirmado y la publicación real del evento (latencia del relay).
- Se debe mantener y depurar la tabla Outbox (purga de eventos ya publicados, monitoreo de rezago).
