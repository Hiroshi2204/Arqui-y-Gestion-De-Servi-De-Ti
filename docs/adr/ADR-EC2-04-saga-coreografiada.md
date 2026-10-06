# ADR-EC2-04 — Patrón SAGA coreografiada

**Proyecto:** PagaEdu — Pasarela de cobranzas recurrentes (Dominio 4: Recaudación y Pagos)
**Curso:** Arquitectura y Gestión de Servicios TI — EC3

## Estado

Aceptada.

## Contexto

Un cobro ya confirmado puede requerir reversa o compensación si un paso posterior falla (por ejemplo, una validación bancaria tardía que rechaza el cargo). El proyecto ya descartó, en el ADR-EC2-01, un ESB u orquestador centralizado por el riesgo de punto único de falla y de acoplamiento organizacional; la compensación debe seguir el mismo principio de coreografía.

## Decisión

Coordinar las compensaciones entre servicios mediante eventos publicados y consumidos de forma autónoma, **sin un orquestador central**: el Servicio de Cobranzas emite el evento `PagoConfirmado`, el Servicio de Conciliación detecta la inconsistencia y publica el evento de compensación correspondiente, y cada servicio interesado reacciona de forma autónoma.

## Alternativas evaluadas

- **a) SAGA orquestada** (un servicio central dirige cada paso de la compensación): se descarta porque introduce un nuevo punto único de falla y un acoplamiento que el proyecto busca evitar precisamente al desacoplar los servicios.
- **b) Transacción distribuida (2PC) entre todos los servicios involucrados:** se descarta por su alto costo de coordinación y su mala tolerancia a fallos parciales en un entorno con proveedores externos.

## Consecuencias

### Positivas

- Evita un punto único de falla técnico y organizacional.
- Cada servicio puede evolucionar y desplegar su propia lógica de compensación de forma autónoma, sin coordinar releases con un orquestador central.

### Negativas (explícitas)

- La lógica de compensación queda distribuida entre varios servicios, lo que dificulta su trazabilidad y depuración frente a una orquestación central.
- Exige aplicar la misma disciplina de idempotencia también en los eventos de compensación, para no reintroducir el riesgo de doble reversa.
