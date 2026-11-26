# Gestión de componentes descontinuados en BOM

**Matrícula:** 181914 \
**Issue:** [#239](https://github.com/jesusrevilla/ERP-simulator/issues/239) · **Rama:** `feature/181914` \
**Proyecto:** ERP inteligente para la gestión, planeación y fabricación de equipos de cómputo \
**Proceso ERP:** identificación, evaluación y sustitución de componentes descontinuados en el módulo BOM, integrado con MRP, Inventarios, Compras y Producción.

---

## Resumen

Cuando un fabricante descontinúa un componente, por ejemplo una tarjeta madre, que sigue en BOM activas, el MRP continúa recomendando comprarlo. El problema se descubre tarde, de forma manual y sin trazabilidad. Esta propuesta define un proceso **TO-BE** con estas características:

1. el ERP registra el **ciclo de vida** del componente y detecta **en el momento** todas las BOM afectadas, incluidas las multinivel;
2. el MRP **deja de recomendar** compras de descontinuados y consume el stock autorizado;
3. un **agente de IA** (a cargo del equipo de IA) propone sustitutos y el ERP valida que sean técnicamente compatibles;
4. el **responsable técnico** autoriza, y el ERP aplica el cambio con una **nueva revisión** de la BOM y conserva el historial.

**Alcance:** este trabajo cubre el proceso ERP (BOM y MRP). Para la base de datos y el agente de IA solo se especifica qué deben hacer sus equipos ([07](07_Requerimientos_para_BD_e_IA.md)); su diseño e implementación no forman parte de este proyecto.

**Resultado del escenario hipotético:**

| KPI | AS-IS | TO-BE |
|---|---|---|
| Tiempo de detección | ≈ 72 h | 0.25 h |
| Tiempo de sustitución | ≈ 120 h | 24 h |
| Recomendaciones de compra inválidas | 25 % | 0 % |
| Órdenes afectadas | 50 % | 0 % |
| Tasa de sustitución exitosa | 0 % | 100 % |

## Contenido

| # | Documento | Contenido |
|---|---|---|
| 1 | [01_Analisis_del_Proceso.md](01_Analisis_del_Proceso.md) | Problema, objetivo, actores, flujo AS-IS, problemas identificados, investigación (ciclo de vida, ERP, integración y compatibilidad) y propuesta |
| 2 | [02_Requisitos_y_Reglas_de_Negocio.md](02_Requisitos_y_Reglas_de_Negocio.md) | 13 requisitos funcionales, 6 no funcionales, información necesaria, **10 reglas de negocio** y decisiones apoyadas por el agente |
| 3 | [03_Diagramas_BPMN.md](03_Diagramas_BPMN.md) | BPMN **AS-IS** y **TO-BE** por carriles, y comparación |
| 4 | [04_Diagramas_de_Flujo.md](04_Diagramas_de_Flujo.md) | Máquina de estados, detección, validación de propuestas, autorización y aplicación, y regla del MRP |
| 5 | [05_Casos_de_Uso.md](05_Casos_de_Uso.md) | Diagrama, 8 casos de uso detallados y diagrama de secuencia |
| 6 | [06_KPIs.md](06_KPIs.md) | 6 KPIs con descripción, unidad, **método de cálculo**, meta y resultados |
| 7 | [07_Requerimientos_para_BD_e_IA.md](07_Requerimientos_para_BD_e_IA.md) | Qué deben hacer los equipos de Bases de Datos e IA para que el proceso funcione, y eventos de integración |
| 8 | [08_Escenarios_de_Validacion_y_Evidencias.md](08_Escenarios_de_Validacion_y_Evidencias.md) | 18 escenarios de validación con su evidencia |

### Artefactos ejecutables

| Archivo | Descripción |
|---|---|
| [gestor_descontinuados.py](gestor_descontinuados.py) | Simulación del proceso ERP (solo biblioteca estándar): ciclo de vida, where-used, alertas, validación de propuestas, autorización, sustitución, MRP y KPIs |
| [../../tests/test_descontinuados_181914.py](../../tests/test_descontinuados_181914.py) | 17 pruebas unitarias |
| [evidencias/](evidencias/) | Salidas de la simulación y de pytest |

## Cómo ejecutar

```bash
# Simulación del escenario (Python 3.10+)
python BOM/Descontinuados_181914/gestor_descontinuados.py

# Pruebas
pytest tests/test_descontinuados_181914.py
```

## Trazabilidad con los criterios de aceptación del issue

| Criterio | Dónde se cumple |
|---|---|
| **Documentación completa:** problema, actores, proceso actual y propuesta de mejora | [01](01_Analisis_del_Proceso.md) §1.1, §1.3, §1.4 y §1.7; detalle en [02](02_Requisitos_y_Reglas_de_Negocio.md) y [07](07_Requerimientos_para_BD_e_IA.md) |
| **Proceso modelado** con los diagramas correspondientes | BPMN AS-IS y TO-BE en [03](03_Diagramas_BPMN.md); flujos de detección, evaluación, autorización y sustitución en [04](04_Diagramas_de_Flujo.md); casos de uso y secuencia en [05](05_Casos_de_Uso.md) |
| **KPIs definidos** con descripción, unidad y método de cálculo | [06](06_KPIs.md) §6.1 (definición y método de cálculo) y §6.3 (resultados) |
| **Reglas de negocio documentadas** para la detección, evaluación y sustitución | [02](02_Requisitos_y_Reglas_de_Negocio.md) §2.4 (RN-01 a RN-10, con el lugar donde se aplica cada una) |
| **Pull Request generado** desde `feature/181914` | PR hacia `jesusrevilla/ERP-simulator:development` |

### Entregables del issue

| Entregable | Ubicación |
|---|---|
| Diagramas BPMN | [03_Diagramas_BPMN.md](03_Diagramas_BPMN.md) |
| Diagramas de flujo | [04_Diagramas_de_Flujo.md](04_Diagramas_de_Flujo.md) |
| Casos de uso | [05_Casos_de_Uso.md](05_Casos_de_Uso.md) |
| Reglas de negocio | [02_Requisitos_y_Reglas_de_Negocio.md](02_Requisitos_y_Reglas_de_Negocio.md#24-reglas-de-negocio) |
| KPIs | [06_KPIs.md](06_KPIs.md) |
| Evidencias | [08_Escenarios_de_Validacion_y_Evidencias.md](08_Escenarios_de_Validacion_y_Evidencias.md), [evidencias/](evidencias/) |

### Plan de actividades: estado

| Fase | Actividad | Estado |
|---|---|---|
| Investigación | Ciclo de vida de componentes electrónicos; sustitución en ERP; integración BOM–MRP–Inventarios–Compras–Producción; criterios de compatibilidad | ✅ [01 §1.6](01_Analisis_del_Proceso.md#16-investigación) |
| Requisitos | Actores, requisitos funcionales, información necesaria, reglas de negocio y decisiones que podrá apoyar el agente de IA | ✅ [01 §1.3](01_Analisis_del_Proceso.md#13-actores-involucrados), [02](02_Requisitos_y_Reglas_de_Negocio.md) |
| Modelado | BPMN AS-IS y TO-BE, diagramas de flujo y casos de uso | ✅ [03](03_Diagramas_BPMN.md), [04](04_Diagramas_de_Flujo.md), [05](05_Casos_de_Uso.md) |
| Diseño de mejoras | Mecanismo de detección, identificación de BOM afectadas, flujo de propuesta y autorización de sustitutos, requerimientos para los equipos de IA y BD, KPIs y escenarios de validación | ✅ [04](04_Diagramas_de_Flujo.md), [06](06_KPIs.md), [07](07_Requerimientos_para_BD_e_IA.md), [08](08_Escenarios_de_Validacion_y_Evidencias.md) |
| Documentación | Análisis, requisitos, diagramas, requerimientos para IA y BD, evidencias y PR | ✅ Esta carpeta |
