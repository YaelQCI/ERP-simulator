# 4. Diagramas de flujo

Estos diagramas describen la **lógica** de cada etapa: detección, evaluación, autorización y sustitución. Cada uno corresponde a una función de `gestor_descontinuados.py`.

---

## 4.1 Máquina de estados del ciclo de vida (RN-01)

```mermaid
stateDiagram-v2
    [*] --> ACTIVO: alta del componente
    ACTIVO --> PROXIMO_A_DESCONTINUAR: PCN / EOL anunciado
    PROXIMO_A_DESCONTINUAR --> ACTIVO: proveedor revoca el aviso
    ACTIVO --> DESCONTINUADO: discontinuación inmediata
    PROXIMO_A_DESCONTINUAR --> DESCONTINUADO: llega la fecha de fin
    DESCONTINUADO --> [*]

    note right of PROXIMO_A_DESCONTINUAR
        Alerta preventiva.
        Compras permitidas hasta la fecha LTB.
        Riesgo si se propone como sustituto.
    end note
    note right of DESCONTINUADO
        Estado final.
        Sin compras ordinarias (RN-02).
        Stock utilizable si está autorizado (RN-03).
    end note
```

---

## 4.2 Detección y generación de la alerta

Funciones: `ERP.cambiar_estado`, `ERP.boms_afectadas`, `ERP.generar_alerta`, `ERP.priorizar`.

```mermaid
flowchart TD
    A([Cambio de estado solicitado]) --> B{¿Estado válido<br/>y transición permitida?}
    B -- No --> B1[Rechazar con error RN-01] --> Z0([Fin])
    B -- Sí --> C[Actualizar estado y<br/>guardar historial]
    C --> D{¿Nuevo estado es<br/>DESCONTINUADO o<br/>PROXIMO_A_DESCONTINUAR?}
    D -- No --> Z1([Fin])
    D -- Sí --> E[Inicializar cola con el SKU, nivel 1]
    E --> F{¿Cola vacía?}
    F -- No --> G[Tomar SKU buscado]
    G --> H[Buscar BOM ACTIVAS que lo<br/>contienen en sus líneas]
    H --> I{¿Padre ya está<br/>en la ruta?}
    I -- Sí: ciclo --> F
    I -- No --> J[Registrar BOM, nivel y ruta]
    J --> K[Encolar padre con nivel + 1]
    K --> F
    F -- Sí --> L[Buscar órdenes abiertas cuyo producto<br/>requiere el SKU en cualquier nivel]
    L --> M[Demanda abierta = Σ cantidad orden ×<br/>requerimiento por unidad]
    M --> N[Cobertura = min de existencias / demanda, 1]
    N --> P{Prioridad RN-10}
    P -- sin BOM --> P1[BAJA]
    P -- descontinuado + órdenes<br/>+ cobertura menor a 100 % --> P2[CRITICA]
    P -- órdenes, o descontinuado<br/>en ≥ 3 BOM --> P3[ALTA]
    P -- otro caso --> P4[MEDIA]
    P1 & P2 & P3 & P4 --> Q[Crear alerta + BOM afectadas<br/>Emitir evento BOM.DiscontinuedComponentAlert]
    Q --> R[Notificar a Responsable de BOM,<br/>Compras, Producción y Responsable técnico]
    R --> Z2([Fin])
```

**Ejemplo del escenario:** `MB-B550-PLUS` → 4 BOM (2 de nivel 1 y 2 de nivel 2), 3 órdenes abiertas, demanda de 45, existencias de 12, cobertura de 27 % → prioridad **CRITICA**.

---

## 4.3 Recepción y validación de propuestas de sustitutos

Funciones: `ERP.recibir_propuestas`, `ERP.es_compatible`.

El agente de IA es un sistema externo: recibe la alerta y devuelve sustitutos ordenados y justificados. Cómo los elige lo define el equipo de IA (ver [07 §7.2](07_Requerimientos_para_BD_e_IA.md#72-requerimientos-para-el-equipo-de-inteligencia-artificial)). Este flujo muestra lo que hace el ERP con esa respuesta.

```mermaid
flowchart TD
    A([Alerta abierta]) --> B[[Solicitar propuestas<br/>al agente de IA]]
    B --> C[Recibir lista de sustitutos<br/>en orden de preferencia]
    C --> D{¿Quedan propuestas<br/>por validar?}
    D -- Sí --> E[Tomar propuesta]
    E --> F{¿Misma categoría<br/>y no DESCONTINUADO?}
    F -- No --> X[Rechazar con motivo] --> D
    F -- Sí --> G[Evaluar cada criterio de la categoría:<br/>igual → valor idéntico<br/>mínimo → valor ≥ original]
    G --> H{¿Cumple todos?}
    H -- No --> X
    H -- Sí --> K[Aceptar propuesta<br/>conservando su orden] --> D
    D -- No --> L{¿Hay propuestas<br/>válidas?}
    L -- No --> M[Escalar al responsable técnico:<br/>rediseño o compra LTB] --> Z([Fin])
    L -- Sí --> N[Presentar propuestas válidas<br/>al responsable técnico]
    N --> Z
```

**Resultado en el escenario** (la respuesta del agente está simulada):

| Orden del agente | Propuesta | Validación del ERP (RN-05) |
|---|---|---|
| 1 | `MB-B550-PRO` | Compatible |
| 2 | `MB-X570-ELITE` | Compatible |
| 3 | `MB-B650-AURA` | **Rechazada:** socket AM5 ≠ AM4, memoria DDR5 ≠ DDR4 |

---

## 4.4 Autorización y aplicación de la sustitución

Funciones: `ERP.autorizar`, `ERP.aplicar_sustitucion`.

```mermaid
flowchart TD
    A([Propuesta seleccionada]) --> B{¿Rol del usuario es<br/>RESPONSABLE_TECNICO?}
    B -- No --> B1[Rechazar: RN-06] --> Z0([Fin])
    B -- Sí --> C{¿Decisión?}
    C -- Rechazar --> C1[Registrar RECHAZADA<br/>con comentario] --> Z1([Solicitar nuevas propuestas])
    C -- Aprobar --> D{¿Revalidación de<br/>compatibilidad OK?}
    D -- No --> D1[Rechazar: RN-05] --> Z0
    D -- Sí --> E[Registrar autorización APROBADA<br/>inmutable]
    E --> F[Evaluar órdenes abiertas afectadas<br/>en todos los niveles]
    F --> G{Estado de la orden}
    G -- LIBERADA / EN_PROCESO --> G1[CONSERVAR versión<br/>snapshot]
    G -- PLANIFICADA --> G2[REPLANIFICAR]
    G1 & G2 --> H[Por cada BOM con uso directo]
    H --> I[Crear revisión vN+1 copiando líneas<br/>y reemplazando original → sustituto]
    I --> J[Marcar vN como OBSOLETO y<br/>cerrar su vigencia]
    J --> K[Reasignar órdenes PLANIFICADA<br/>de vN a vN+1]
    K --> L[Registrar sustitución en el historial<br/>con su autorización]
    L --> M{¿Más BOM directas?}
    M -- Sí --> H
    M -- No --> N[Cerrar alerta: RESUELTA<br/>Emitir BOM.SubstitutionApplied]
    N --> O[Disparar recálculo de MRP] --> Z2([Fin])
```

> Las BOM de nivel superior (por ejemplo `PC-OFI-100`) **no se versionan**. Referencian al subensamble `ENS-BASE-AM4` y, al explotarse, toman su revisión vigente (`v2`). Así se evita multiplicar revisiones innecesarias.

Todo el bloque de E a N se ejecuta en **una transacción** (RNF-02): si falla un paso, no se aplica ninguno.

---

## 4.5 Regla del MRP para componentes descontinuados

Función: `ERP.mrp`.

```mermaid
flowchart TD
    A([Corrida MRP]) --> B[Explosión multinivel de la demanda<br/>→ necesidades brutas por SKU]
    B --> C{¿Quedan SKU?}
    C -- No --> Z([Publicar recomendaciones])
    C -- Sí --> D[Tomar SKU y requerido]
    D --> E{¿SKU es sustituto de un<br/>descontinuado con stock<br/>autorizado?}
    E -- Sí --> E1[Consumir primero el stock<br/>del original<br/>política AGOTAR_EXISTENCIAS] --> F
    E -- No --> F[Neto = requerido − existencias]
    F --> G{¿SKU DESCONTINUADO?}
    G -- No --> H{¿Neto > 0?}
    H -- Sí --> H1[Recomendación ORDINARIA] --> C
    H -- No --> C
    G -- Sí --> I{¿Uso de stock<br/>autorizado?}
    I -- Sí --> I1[Consumir existencias disponibles]
    I -- No --> I2[No consumir]
    I1 & I2 --> J{¿Queda faltante?}
    J -- Sí --> J1[Faltante BLOQUEADO:<br/>requiere sustitución<br/>sin recomendación de compra] --> C
    J -- No --> C
```

**Escenario (pedido de 30 `PC-OFI-100` + 10 `PC-WS-300`):**

| Momento | `MB-B550-PLUS` | `MB-B550-PRO` | Otras compras |
|---|---|---|---|
| AS-IS | **Compra 28** (inválida) | — | RAM 20, SSD 30, PSU 25 |
| TO-BE, sustitución pendiente | Consume 12; **28 bloqueadas** | — | RAM 20, SSD 30, PSU 25 |
| TO-BE, sustitución aplicada | Consume 12 (agotar existencias) | 28 desde stock (40) → sin compra | RAM 20, SSD 30, PSU 25 |
