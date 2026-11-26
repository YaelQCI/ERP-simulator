# 2. Levantamiento de requisitos y reglas de negocio

## 2.1 Requisitos funcionales

| ID | Requisito | Criterio de aceptación | Actor |
|---|---|---|---|
| RF-01 | Registrar y consultar el estado de ciclo de vida de cada componente (`ACTIVO`, `PROXIMO_A_DESCONTINUAR`, `DESCONTINUADO`) con fecha, fuente (PCN), fecha de última compra y usuario. | El cambio queda en el historial; las transiciones inválidas se rechazan (RN-01). | Comprador |
| RF-02 | Mantener las especificaciones técnicas estructuradas por componente y los criterios de compatibilidad por categoría. | Cada atributo tiene un valor de texto o uno numérico; cada criterio tiene una regla `IGUAL`, `MINIMO` o `MAXIMO`. | Responsable técnico |
| RF-03 | Detectar automáticamente las BOM afectadas cuando un componente cambia a `PROXIMO_A_DESCONTINUAR` o `DESCONTINUADO`, incluidas las estructuras multinivel. | La alerta lista cada BOM activa con su nivel y su ruta (por ejemplo, `PC-OFI-100 > ENS-BASE-AM4 > MB-B550-PLUS`). | ERP |
| RF-04 | Generar una alerta con existencias, órdenes abiertas afectadas, demanda abierta, cobertura y prioridad, y notificar a los responsables. | La alerta se crea en la misma transacción del cambio de estado o en el siguiente ciclo del proceso de detección (≤ 15 min). | ERP |
| RF-05 | Excluir los componentes descontinuados de las recomendaciones ordinarias de compra del MRP. | Ninguna recomendación ordinaria se emite para un componente `DESCONTINUADO`; el faltante se reporta como bloqueado. | MRP |
| RF-06 | Permitir consumir las existencias del componente descontinuado cuando el uso esté autorizado. | El MRP asigna primero el stock autorizado y reporta el consumo. | MRP / Almacén |
| RF-07 | Recibir las propuestas de sustitutos del agente de IA y validar su compatibilidad técnica (RN-05). | Las propuestas incompatibles se rechazan e indican los atributos que no cumplen; las válidas conservan el orden que dio el agente. | ERP (las propuestas las genera el equipo de IA) |
| RF-08 | Registrar la decisión del responsable técnico (aprobar o rechazar) con comentario. | Solo el rol `RESPONSABLE_TECNICO` puede decidir; la decisión es inmutable. | Responsable técnico |
| RF-09 | Evaluar las órdenes de fabricación afectadas antes de aplicar la sustitución. | Cada orden abierta recibe una decisión: `CONSERVAR` (liberada o en proceso) o `REPLANIFICAR` (planificada). | Producción |
| RF-10 | Aplicar la sustitución creando una nueva revisión de cada BOM con uso directo y cerrando la vigencia de la anterior. | La versión anterior queda `OBSOLETO` sin modificar sus líneas; la nueva queda `ACTIVO`; las BOM superiores toman el subensamble vigente. | Responsable de BOM |
| RF-11 | Conservar el historial completo: ciclo de vida, alerta, recomendaciones, autorización, sustitución y evaluación de órdenes. | Es posible reconstruir quién, cuándo, por qué y qué cambió; los registros históricos no se pueden editar ni borrar. | ERP |
| RF-12 | Recalcular el MRP después de la sustitución con la política "agotar existencias". | Se consume el stock del original antes de comprar el sustituto. | MRP |
| RF-13 | Calcular y consultar los KPIs del proceso. | Los seis KPIs se calculan con el método descrito en 06_KPIs.md. | Gerencia |

## 2.2 Requisitos no funcionales

| ID | Requisito |
|---|---|
| RNF-01 | **Integridad.** Las reglas críticas (RN-01, RN-02, RN-06, RN-08) también deben garantizarse en la base de datos, para que ningún canal de escritura pueda saltarlas. Lo implementa el equipo de BD (ver [07 §7.1.3](07_Requerimientos_para_BD_e_IA.md#713-reglas-que-la-base-de-datos-debe-garantizar)). |
| RNF-02 | **Atomicidad.** La aplicación de una sustitución (nueva revisión, cierre de la anterior, historial y reasignación de órdenes) ocurre en una sola transacción. |
| RNF-03 | **Explicabilidad.** Toda propuesta del agente de IA debe incluir su justificación y la versión del modelo (requerimiento para el equipo de IA, ver [07 §7.2.5](07_Requerimientos_para_BD_e_IA.md#725-restricciones-para-el-agente)). |
| RNF-04 | **Supervisión humana.** El agente no escribe en BOM ni en órdenes; solo propone. |
| RNF-05 | **Rendimiento.** La consulta where-used multinivel debe responder en menos de 2 s con 10 000 BOM activas (requerimiento para el equipo de BD). |
| RNF-06 | **Auditoría.** Las fechas se registran en UTC, con hora, para medir los KPIs de tiempo. |

## 2.3 Información necesaria

| Información | Origen | Uso en el proceso | Requerimiento para BD (ver 07) |
|---|---|---|---|
| Estado de ciclo de vida, fecha de EOL, LTB y fuente | Proveedor → Compras | Detección y bloqueo de compras | BD-01 |
| Categoría, especificaciones técnicas y criterios de compatibilidad | Ingeniería | Validación de sustitutos (RN-05) | BD-02 |
| Estructura de las BOM activas | BOM | Where-used multinivel | Ya existe (`BOM/BD/schema.sql`); consulta CQ-01 |
| Existencias disponibles y reservadas | Inventarios | Cobertura y consumo autorizado | BD-04 |
| Costo, tiempo de entrega y disponibilidad por proveedor | Compras | Entrada para el agente de IA | Catálogo de compras |
| Órdenes de fabricación abiertas | Producción | Prioridad e impacto | Ya existe; BD-10 para KPIs |
| Alertas y BOM afectadas | ERP | Seguimiento y KPI | BD-05, BD-08 |
| Propuestas del agente de IA | IA | Decisión técnica | BD-03 |
| Autorizaciones | Responsable técnico | Control | BD-07 |
| Historial de sustituciones y evaluación de órdenes | ERP | Trazabilidad | BD-06 |
| Recomendaciones de compra del MRP | MRP | KPI de compras inválidas | BD-09 |

## 2.4 Reglas de negocio

Las reglas preliminares del issue se numeran y precisan a continuación. La columna **Aplicación** indica dónde se hace cumplir cada regla.

| ID | Regla | Regla preliminar del issue | Aplicación |
|---|---|---|---|
| RN-01 | **Ciclo de vida.** Las transiciones válidas son `ACTIVO ↔ PROXIMO_A_DESCONTINUAR` y `ACTIVO/PROXIMO_A_DESCONTINUAR → DESCONTINUADO`. `DESCONTINUADO` es un estado final; si el proveedor reactiva la pieza, se registra como un artículo nuevo. Todo cambio guarda usuario, fecha y fuente. | — | `ERP.cambiar_estado`; también en BD (7.1.3) |
| RN-02 | **Sin compras ordinarias.** Un componente `DESCONTINUADO` no genera nuevas recomendaciones ordinarias de compra. Solo se admite una compra de tipo `ULTIMA_COMPRA` antes de la fecha LTB y con autorización de Compras. | "Un componente descontinuado no deberá generar nuevas recomendaciones ordinarias de compra." | `ERP.mrp`; también en BD (7.1.3) |
| RN-03 | **Consumo de existencias.** Un descontinuado se puede seguir usando mientras haya unidades disponibles y `uso_stock_autorizado = TRUE`. Tras la sustitución, el MRP consume primero ese stock (política `AGOTAR_EXISTENCIAS`), salvo que se elija `INMEDIATA` por razones de calidad. | "Podrá seguir utilizándose mientras existan unidades disponibles y su uso esté autorizado." | `ERP.mrp` |
| RN-04 | **Detección multinivel.** El ERP identifica todas las BOM activas que contienen el componente en cualquier nivel, sin repetir y sin entrar en ciclos. | "El ERP deberá identificar todas las listas de materiales afectadas, incluyendo aquellas con estructuras multinivel." | `ERP.boms_afectadas`; consulta CQ-01 (7.1.2) |
| RN-05 | **Compatibilidad técnica.** Un sustituto debe pertenecer a la misma categoría, no estar descontinuado y cumplir **todos** los criterios de compatibilidad de su categoría: atributos que deben ser iguales al original y atributos que deben igualarlo o superarlo. Se puede proponer un componente `PROXIMO_A_DESCONTINUAR`, pero el responsable técnico debe considerar ese riesgo. | "Los componentes sustitutos deberán cumplir con los requisitos técnicos del producto." | `ERP.es_compatible`, al recibir propuestas y al autorizar |
| RN-06 | **Autorización obligatoria.** Ninguna sustitución se aplica sin una autorización `APROBADA` del rol `RESPONSABLE_TECNICO` que coincida con el par original–sustituto recomendado. | "Ninguna sustitución podrá aplicarse sin autorización del responsable correspondiente." | `ERP.autorizar`, `ERP.aplicar_sustitucion`; también en BD (7.1.3) |
| RN-07 | **Evaluación de órdenes.** Antes de aplicar la sustitución se evalúan las órdenes abiertas afectadas (en cualquier nivel): las `LIBERADA`/`EN_PROCESO` **conservan** su versión de BOM (snapshot) y las `PLANIFICADA` se **replanifican** con la nueva revisión. Las `TERMINADA`/`CANCELADA` no se tocan. | "Las órdenes de fabricación existentes deberán evaluarse antes de aplicar modificaciones." | `ERP.aplicar_sustitucion` |
| RN-08 | **Historial inmutable.** Toda sustitución crea una **nueva revisión** de la BOM; la anterior pasa a `OBSOLETO` sin modificar sus líneas. Las autorizaciones, las sustituciones y el historial del ciclo de vida no se pueden actualizar ni borrar. | "Toda sustitución deberá conservar un registro histórico." | `ERP.aplicar_sustitucion`; también en BD (7.1.3) |
| RN-09 | **Supervisión humana del agente.** El agente de IA solo propone. No cambia estados, BOM ni órdenes. | — | Restricción para el equipo de IA (7.2.5) |
| RN-10 | **Prioridad de alertas.** `CRITICA`: descontinuado, con órdenes abiertas y existencias insuficientes. `ALTA`: con órdenes abiertas, o descontinuado usado en 3 o más BOM. `MEDIA`: usado en BOM, sin órdenes. `BAJA`: sin uso en BOM activas. | — | `ERP.priorizar`; el agente de IA puede sugerir un ajuste |

## 2.5 Decisiones y quién las apoya

El issue pide identificar las decisiones que podrá apoyar un agente inteligente. El agente lo diseña e implementa el equipo de IA; este proceso solo define qué se espera de él (ver [07 §7.2](07_Requerimientos_para_BD_e_IA.md#72-requerimientos-para-el-equipo-de-inteligencia-artificial)).

| Decisión | Quién decide | Lo resuelve el ERP (este módulo) | Apoyo esperado del agente de IA |
|---|---|---|---|
| ¿Qué BOM y órdenes están afectadas? | ERP | Where-used multinivel y cruce con órdenes (RN-04) | Resumir el impacto para los responsables |
| ¿Qué tan urgente es la alerta? | Responsable de BOM | Prioridad base con RN-10 | Sugerir un ajuste según el impacto en producción |
| ¿Existe riesgo de desabasto? | Comprador / Producción | Cobertura = existencias / demanda abierta | Advertir el riesgo considerando la fecha LTB |
| ¿Cuál sustituto conviene más? | Responsable técnico | — | Proponer sustitutos ordenados y justificados |
| ¿El sustituto es compatible? | Responsable técnico | Valida RN-05 sobre cada propuesta | No proponer incompatibles |
| ¿Se aplica la sustitución? | Responsable técnico | Exige la autorización (RN-06) | — (**decisión humana obligatoria**) |
| ¿Se conserva o se replanifica cada orden? | Producción | Propone la decisión según el estado de la orden (RN-07) | — |
