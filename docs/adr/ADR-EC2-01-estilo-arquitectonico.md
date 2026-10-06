# ADR-EC2-01 — Estilo arquitectónico

**Proyecto:** PagaEdu — Pasarela de cobranzas recurrentes (Dominio 4: Recaudación y Pagos)
**Curso:** Arquitectura y Gestión de Servicios TI — EC3

## Estado

Aceptada (formaliza y refina la propuesta preliminar de la EC1).

## Contexto

El monolito actual comparte una única base de datos entre los módulos de matrícula, facturación, notificaciones, conciliación y reportes, y se despliega como una sola unidad. Durante los picos de matrícula, la carga generada por el job batch de cobranza degrada el desempeño de módulos no relacionados, y un incidente en conciliación puede bloquear también el flujo de cobro. El negocio necesita que el servicio crítico de cobranzas escale y falle de forma aislada, sin arrastrar al resto del sistema, y sin exponerse a un escenario de picos de hasta 3× la carga promedio.

## Decisión

Migrar incrementalmente, mediante el patrón **Strangler Fig**, desde el monolito en capas hacia una arquitectura de **microservicios orientada a eventos**, extrayendo primero el Servicio de Cobranzas Recurrentes y comunicándolo con el resto de módulos mediante un bus de eventos, en lugar de acceso directo a la base de datos compartida.

## Alternativas evaluadas

- **a) Monolito modular** (mejorar la modularidad interna sin separar los despliegues): se descarta porque conserva una única unidad de despliegue y recursos compartidos, sin resolver el aislamiento de fallos ni el escalamiento independiente que exige el escenario de picos 3×.
- **b) Escalar verticalmente el monolito** (más CPU/RAM): se descarta porque no resuelve el acoplamiento de datos ni el riesgo de fallo en cascada entre módulos; solo pospone el problema y tiene un techo de costo-beneficio decreciente.
- **c) SOA clásica con ESB centralizado:** se descarta porque el bus terminaría concentrando el acoplamiento y convirtiéndose en un punto único de falla y en un cuello de botella organizacional para publicar cambios, exactamente lo opuesto al objetivo de desempeño y autonomía que se busca.
- **d) Reescritura completa ("big bang"):** se descarta por el riesgo de interrumpir la recaudación mensual real de 9,000 estudiantes durante una migración de alto riesgo y sin marcha atrás sencilla.

## Consecuencias

### Positivas

- El servicio de cobranzas puede escalar y desplegarse de forma independiente durante los picos de matrícula.
- Un incidente en conciliación, notificaciones o reportes ya no bloquea la capacidad de cobrar.
- Se reduce el acoplamiento de datos al eliminar la base de datos compartida para el flujo crítico.

### Negativas (explícitas)

- Mayor complejidad operativa (más servicios, bus de eventos, observabilidad distribuida) que el equipo actual no tiene desarrollada.
- Consistencia eventual visible en los módulos que no forman parte del camino crítico de cobro, lo que obliga a capacitar a atención al cliente y a las áreas administrativas.
- Costo adicional de infraestructura y licenciamiento (bus de mensajería, observabilidad, posible orquestación de contenedores) que no existía en el monolito.
