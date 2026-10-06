# ADR-EC2-02 — Patrón de Idempotencia

**Proyecto:** PagaEdu — Pasarela de cobranzas recurrentes (Dominio 4: Recaudación y Pagos)
**Curso:** Arquitectura y Gestión de Servicios TI — EC3

## Estado

Aceptada.

## Contexto

El módulo de Facturación reintenta cargos ante timeouts o errores transitorios de red con el agregador de pagos. Al no existir un identificador único de intento lógico de cobro, un mismo reintento puede generar dos cargos reales contra la tarjeta o cuenta del apoderado, especialmente en los picos de fin/inicio de mes cuando la pasarela externa está más congestionada. El negocio exige que cada cobro se procese exactamente una vez, sin sacrificar la capacidad de reintentar ante fallos legítimos.

## Decisión

Usar una **Idempotency-Key** determinística por intento lógico de cobro (hash de institución + alumno + periodo + concepto + monto), exigida en toda petición de cobro hacia el Servicio de Cobranzas, con una restricción **UNIQUE** en base de datos que rechace duplicados.

## Alternativas evaluadas

- **a) Confiar en los reintentos y la deduplicación propios del agregador de pagos:** descartada porque no cubre duplicados originados en el propio sistema (p. ej. dos hilos del job batch procesando el mismo registro) y mantiene una dependencia total de un comportamiento no garantizado por un tercero.
- **b) Bloqueo (lock) distribuido durante toda la operación de cobro:** se descarta por el costo en desempeño y el riesgo de deadlocks bajo la carga de picos de matrícula.
- **c) Deduplicación solo en el cliente (portal/app):** se descarta porque no cubre los reintentos que puede generar el propio proveedor externo o un job batch programado.

## Consecuencias

### Positivas

- Elimina los cobros duplicados incluso ante reintentos automáticos o manuales.
- Habilita reintentos seguros como estrategia general de resiliencia.
- Deja trazabilidad explícita de qué intento generó qué resultado, útil para auditoría.

### Negativas (explícitas)

- Pequeña latencia adicional en cada cobro por la escritura y validación de la clave.
- Una tabla adicional que el equipo debe mantener y depurar.
- Dependencia de disciplina en todos los canales (portal web, app móvil, backoffice y job batch), que deben generar y propagar correctamente la Idempotency-Key: un error de implementación en un solo canal reintroduce el riesgo que se buscaba eliminar.
