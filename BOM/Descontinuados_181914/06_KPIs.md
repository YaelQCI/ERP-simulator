# 6. Indicadores de desempeño (KPIs)

Cada KPI tiene descripción, unidad, método de cálculo, fuente de datos, frecuencia y meta. El método de cálculo está implementado en la simulación (`ERP.kpis` en `gestor_descontinuados.py`) y verificado con pruebas.

## 6.1 Definición

| # | Indicador | Descripción | Unidad | Método de cálculo | Datos de origen | Frecuencia | Meta |
|---|---|---|---|---|---|---|---|
| 1 | **Tiempo de detección** | Tiempo entre el registro de la descontinuación y la generación de la alerta. | Horas | Promedio de (fecha de la alerta − fecha de registro de la descontinuación) de las alertas del periodo | Alertas | Mensual | ≤ 0.25 h (un ciclo del proceso de detección) |
| 2 | **Tiempo de sustitución** | Tiempo entre la detección (alerta) y la aprobación del sustituto. | Horas o días | Promedio de (fecha de la primera autorización aprobada − fecha de la alerta), por alerta | Alertas y autorizaciones | Mensual | ≤ 48 h para alertas `CRITICA`; ≤ 5 días para el resto |
| 3 | **BOM afectadas** | Porcentaje de BOM activas que contienen un componente descontinuado en **cualquier nivel**. | % | (BOM activas distintas que contienen, en cualquier nivel, al menos un componente descontinuado ÷ total de BOM activas) × 100 | Ciclo de vida de componentes y BOM activas (where-used multinivel) | Diaria | 0 % al cerrar cada alerta; tendencia a la baja |
| 4 | **Recomendaciones de compra inválidas** | Porcentaje de recomendaciones del MRP que corresponden a componentes descontinuados. | % | (Recomendaciones de la corrida cuyo componente está descontinuado ÷ total de recomendaciones de la corrida) × 100 | Recomendaciones de compra del MRP y ciclo de vida | Por corrida de MRP | 0 % (lo garantiza RN-02) |
| 5 | **Órdenes de fabricación afectadas** | Porcentaje de órdenes retrasadas por componentes descontinuados. | % | (Órdenes retrasadas cuya causa es un componente descontinuado ÷ órdenes no canceladas del periodo) × 100 | Órdenes de fabricación (fecha requerida, fecha de fin real, causa de retraso) | Mensual | < 2 % |
| 6 | **Tasa de sustitución exitosa** | Porcentaje de componentes descontinuados con un sustituto validado, respecto a los que requieren sustitución. | % | (Componentes descontinuados con BOM afectadas que tienen una sustitución aprobada ÷ componentes descontinuados con BOM afectadas) × 100 | Alertas y autorizaciones | Mensual | ≥ 90 % |

**Notas de cálculo**

- **KPI 1.** Si la detección es síncrona (ocurre en el mismo momento en que se registra el cambio de estado), el valor tiende a 0 h. Si las notificaciones se envían por lotes cada 15 min, el máximo es 0.25 h. El escenario usa 0.25 h, el peor caso.
- **KPI 3.** Se cuenta cada BOM una sola vez, aunque contenga varios descontinuados o el mismo componente por varias rutas.
- **KPI 4.** Se evalúa con el estado **vigente** del componente. Una compra emitida antes de conocer la descontinuación también cuenta como inválida, porque representa el riesgo real.
- **KPI 5.** Una orden está retrasada si su fecha de fin real es posterior a la fecha requerida, o si sigue abierta después de la fecha requerida. La causa la registra Producción.
- **KPI 6.** Un componente cuenta en el denominador solo si afecta al menos una BOM activa. Los que no tienen uso no requieren sustitución.

## 6.2 Datos que debe proveer el equipo de Bases de Datos

Para poder calcular los KPIs, la base de datos debe registrar la información BD-01, BD-05, BD-07, BD-08, BD-09 y BD-10 descrita en [07 §7.1.1](07_Requerimientos_para_BD_e_IA.md#711-información-que-se-debe-almacenar). Las consultas concretas las implementa ese equipo.

## 6.3 Resultados en el escenario hipotético

Valores obtenidos al ejecutar `gestor_descontinuados.py` (ver [evidencias](evidencias/)):

| # | KPI | AS-IS | TO-BE al detectar | TO-BE tras la sustitución | Meta | ¿Cumple? |
|---|---|---|---|---|---|---|
| 1 | Tiempo de detección | ≈ 72 h¹ | **0.25 h** | 0.25 h | ≤ 0.25 h | ✅ |
| 2 | Tiempo de sustitución | ≈ 120 h¹ | — (pendiente) | **24 h** | ≤ 48 h (CRITICA) | ✅ |
| 3 | BOM afectadas | 80 % (no medido)² | 80 % | **0 %** | 0 % al cierre | ✅ |
| 4 | Recomendaciones de compra inválidas | **25 %** (1 de 4) | 0 % | **0 %** | 0 % | ✅ |
| 5 | Órdenes de fabricación afectadas | **50 %** (2 de 4)³ | 0 % | **0 %** | < 2 % | ✅ |
| 6 | Tasa de sustitución exitosa | 0 % | 0 % | **100 %** | ≥ 90 % | ✅ |

¹ **Estimación hipotética del AS-IS.** El proveedor descontinúa la pieza el 20/11 a las 09:00. El comprador lo descubre al consultar al proveedor después de la corrida de MRP del 23/11, unas 72 h después. La investigación manual, la validación y la edición de la BOM toman unos 5 días. El AS-IS no genera alertas, así que estos valores no se pueden calcular desde el sistema. Esa imposibilidad es, en sí misma, uno de los problemas detectados.

² El valor existe, pero nadie lo conoce. En el AS-IS no hay consulta where-used multinivel, así que el responsable solo encontraría 2 de las 4 BOM (las de uso directo).

³ Las órdenes OF-1001 y OF-1002 se detienen hasta que se resuelve la sustitución.
