# Agente de Trazabilidad de Gastos — PAM Hotels
### Resumen Ejecutivo para el Equipo de Finanzas
*Junio 2026*

---

## ¿Qué problema resuelve?

El área de Finanzas necesita responder tres preguntas para cada línea de gasto del libro mayor:

| Pregunta | Fuente de la respuesta |
|---|---|
| ¿Qué proveedor emitió la factura y cuál es su RFC? | Archivo Ecommerce (CFDIs recibidos) |
| ¿Cuál es el UUID fiscal del CFDI? | Ecommerce + Libro Mayor |
| ¿Cuándo y con qué cuenta bancaria se pagó? | Partidas de proveedor FBL1N |

Hoy ese proceso se hace **de forma manual**, cruzando tres o cuatro archivos de SAP en Excel. El agente automatiza ese cruce por completo y produce un reporte consolidado listo para revisión, con un semáforo de confianza en cada línea.

---

## Archivos que utiliza el agente

```
AuxGastos.xlsx       →  Libro Mayor (FBL3N)         ~2,400 registros de gasto / mes
files/Pagos/*.xlsx   →  Partidas de Proveedor (FBL1N)  ~176,000 registros (11 sociedades)
Acreedor.xlsx        →  Maestro de Proveedores          ~24,600 registros
ecommerce.xlsx       →  CFDIs Recibidos (Ecommerce)    ~108,600 facturas vigentes
```

> **Nota:** El agente es multi-sociedad. Procesa en un solo paso las 11 sociedades del Grupo KTRC (3400–3415).

---

## ¿Cómo funciona? — El flujo en 3 etapas

```
┌─────────────────┐     ETAPA 1      ┌─────────────────────┐
│  Libro Mayor    │ ──────────────► │  Ecommerce (CFDIs)  │
│  (AuxGastos)   │   UUID / RFC     │  RFC, IVA, subtotal │
└─────────────────┘                 └──────────┬──────────┘
                                               │ UUID confirmado
                                    ETAPA 2    ▼
                                   ┌─────────────────────┐
                                   │  FBL1N (Pagos)      │
                                   │  Fecha pago, banco  │
                                   └──────────┬──────────┘
                                              │
                                   ETAPA 3    ▼
                                   ┌─────────────────────┐
                                   │  ReporteFinal.xlsx  │
                                   │  ✅ Verde / 🟡 Amarillo │
                                   │  🟠 Naranja / 🔴 Rojo  │
                                   └─────────────────────┘
```

### Etapa 1 — Carga y preparación
El agente carga los cuatro archivos, corrige problemas de codificación en los encabezados de SAP, normaliza formatos de fecha y valida que los UUIDs estén en formato correcto (mayúsculas). Solo procesa cuentas de gasto (cuentas 7XXXXXXX) y excluye documentos anulados.

### Etapa 2 — Enriquecimiento con datos fiscales (Ecommerce)
Para cada línea de gasto, el agente busca la factura CFDI correspondiente en el archivo Ecommerce para obtener:
- RFC y nombre del proveedor emisor
- Subtotal e IVA del CFDI
- Forma y método de pago (PUE / PPD)
- Descripción del concepto de la factura

### Etapa 3 — Búsqueda del pago en FBL1N
Con el UUID confirmado en el paso anterior, el agente localiza el pago en el archivo de partidas de proveedor (FBL1N) para obtener:
- Fecha real en que se realizó el pago
- Cuenta bancaria SAP utilizada
- Documento contable del pago

---

## Estrategias de búsqueda — del más confiable al menos confiable

El agente aplica hasta **6 estrategias** en orden de prioridad, deteniéndose en la primera que tenga éxito:

### 🟢 Búsquedas exactas — Confianza EXACTO

| # | Nombre | Condición | Qué hace |
|---|---|---|---|
| **1** | UUID directo (Ecommerce) | El libro mayor tiene UUID | Cruza directamente el UUID con el archivo de CFDIs |
| **2** | UUID directo (FBL1N) | UUID confirmado en paso 1 | Cruza el UUID con el archivo de pagos para obtener fecha y banco |
| **3** | Número de documento | Sin UUID, pero mismo número de doc SAP | Cruza el BELNR (número interno SAP) entre el libro mayor y FBL1N |

> El UUID es el identificador único del SAT para cada factura electrónica. Cuando está disponible en ambos archivos, el cruce es 100% confiable.

---

### 🟡 Búsquedas inteligentes — Confianza DIFUSO

Cuando el UUID no está disponible o no coincide, el agente aplica búsqueda difusa usando combinaciones de criterios:

| # | Nombre | Criterios usados | Confianza |
|---|---|---|---|
| **4** | Difuso con proveedor | Mismo RFC proveedor + fecha ± 5 días + similitud de descripción | Alta / Media |
| **5** | Difuso con número de cuenta | Acreedor SAP + fecha ± 5 días + texto | Alta / Media |
| **6** | Difuso solo por fecha/importe | Fecha ± 3 días + importe ± 1% | Baja — **requiere revisión** |

La similitud de descripción compara el texto de la cuenta de gasto contra el concepto del CFDI o el texto del pago en FBL1N, y asigna una puntuación entre 0 y 1.

---

### 🔴 Sin resolución automática

Las líneas que no cumplen ningún criterio quedan marcadas como **NO_RESUELTO** en una hoja separada del reporte para revisión manual del equipo de Finanzas.

---

## Niveles de confianza en el reporte final

| Color | Etiqueta | Significado | Acción recomendada |
|---|---|---|---|
| 🟢 Verde | `EXACTO` | Cruce por UUID o número de documento | Ninguna — datos verificados |
| 🟡 Amarillo | `DIFUSO_ALTO` | Score de similitud ≥ 70% | Revisión mínima recomendada |
| 🟠 Naranja | `DIFUSO_MEDIO` | Score de similitud ≥ 40% | Revisar antes de usar |
| 🔴 Rosa | `DIFUSO_BAJO` | Score < 40% | Requiere validación manual |
| 🔴 Rojo | `NO_RESUELTO` | Sin match automático | Hoja "Pendientes" — revisión manual |
| ⚪ Gris | `ANULADO` | Documento anulado en SAP | Excluido del proceso |

---

## Resultados actuales — Febrero 2020, Sociedad 3400

> Archivo de prueba: 2,404 líneas de gasto del libro mayor

| Estrategia | Líneas resueltas | % del total |
|---|---|---|
| UUID directo | 378 | 18.8% |
| Número de documento | 100 | 5.0% |
| Difuso con proveedor (RFC) | 23 | 1.1% |
| Difuso sin proveedor | 643 | 31.9% |
| **Total resuelto automáticamente** | **1,144** | **56.8%** |
| Pendientes (revisión manual) | 872 | 43.2% |
| Anulados (excluidos) | 388 | — |

### ¿Por qué el 43% queda pendiente?

La mayoría de los registros sin resolución son documentos tipo **WA** (movimientos de almacén) y **WE** (entradas de mercancía), que son documentos del módulo de compras de SAP. Estos **no tienen una contraparte directa en el archivo de pagos a proveedores** porque el flujo de pago pasa por una cadena adicional (orden de compra → entrada de mercancía → factura de proveedor). Para resolverlos completamente se requeriría acceso a los datos del módulo MM (Gestión de Materiales) de SAP.

Las líneas de tipo **RE** (facturas de proveedor directas) y **SA** (contabilizaciones generales) sí se resuelven correctamente.

---

## Datos que el agente agrega automáticamente al reporte

Para cada línea resuelta, el reporte final incluye campos que no existen en el libro mayor original:

| Campo agregado | Fuente | ¿Para qué sirve? |
|---|---|---|
| RFC Emisor | Ecommerce / Maestro | Verificación fiscal, declaraciones |
| Nombre del proveedor normalizado | Ecommerce / Maestro | Consistencia en reportes |
| Subtotal CFDI | Ecommerce | Desglose para IVA acreditable |
| IVA CFDI | Ecommerce | Conciliación con declaraciones |
| Concepto de la factura | Ecommerce | Auditoría y clasificación |
| Forma de pago SAT | Ecommerce | Cumplimiento fiscal |
| Fecha real de pago | FBL1N | Conciliación bancaria |
| Cuenta bancaria de pago | FBL1N | Trazabilidad de tesorería |
| Documento contable del pago | FBL1N | Auditoría interna |

---

## Archivos que produce el agente

```
output/ReporteFinal_3400_202002.xlsx
│
├── Hoja "Resultado"   →  Todas las líneas de gasto enriquecidas (con semáforo de colores)
├── Hoja "Pendientes"  →  Solo las líneas sin resolución automática (para revisión)
└── Hoja "Métricas"    →  Estadísticas del proceso (totales por estrategia, % resolución)
```

---

## Parámetros que Finanzas puede ajustar

El agente tiene una sección de configuración al inicio del notebook donde se pueden modificar las tolerancias sin necesidad de cambiar código:

| Parámetro | Valor actual | ¿Qué controla? |
|---|---|---|
| `S3_FECHA_TOLERANCIA_DIAS` | 5 días | Ventana de fecha para búsqueda difusa con proveedor |
| `S3_IMPORTE_TOLERANCIA_PCT` | 50% | Tolerancia de importe para búsqueda con proveedor* |
| `S4_FECHA_TOLERANCIA_DIAS` | 3 días | Ventana de fecha para búsqueda difusa sin proveedor |
| `S4_IMPORTE_TOLERANCIA_PCT` | 1% | Tolerancia de importe sin proveedor |
| `TEXT_SCORE_ALTO` | 0.70 | Umbral mínimo para confianza Alta |
| `TEXT_SCORE_MEDIO` | 0.40 | Umbral mínimo para confianza Media |

> *La tolerancia del 50% en importe con proveedor se debe a que el libro mayor registra líneas parciales por centro de costo, mientras que la factura del proveedor tiene el importe total de todos los conceptos.

---

## Próximos pasos sugeridos

| Prioridad | Acción | Impacto esperado |
|---|---|---|
| Alta | Validar manualmente una muestra de matches `DIFUSO_ALTO` para confirmar precisión | Aumentar confianza en el proceso |
| Alta | Proporcionar archivos FBL1N con histórico mayor (actualmente solo 2020) | Resolver facturas de periodos anteriores |
| Media | Integrar datos de órdenes de compra (MM) para resolver documentos WA/WE | Pasar del 57% al ~80% de resolución |
| Media | Ejecutar el proceso mensualmente con los archivos del cierre | Automatizar la conciliación rutinaria |
| Baja | Migración a Google Cloud / BigQuery para procesamiento en tiempo real | Escalabilidad a toda la operación del grupo |

---

## Glosario rápido

| Término | Significado |
|---|---|
| **UUID** | Identificador único del SAT para cada factura electrónica (CFDI) |
| **CFDI** | Comprobante Fiscal Digital por Internet — la factura electrónica mexicana |
| **FBL3N / AuxGastos** | Reporte de libro mayor en SAP — lista las cuentas de gasto |
| **FBL1N** | Reporte de partidas de proveedor en SAP — lista facturas y pagos |
| **BELNR** | Número interno de documento contable en SAP |
| **RFC** | Registro Federal de Contribuyentes — identificador fiscal del proveedor |
| **WA / WE** | Clases de documento SAP para movimientos de almacén y entradas de mercancía |
| **RE** | Clase de documento SAP para facturas de proveedor directas |
| **Matching difuso** | Técnica que encuentra coincidencias aproximadas usando múltiples criterios combinados |

---

*Documento preparado por el equipo de Tecnología — PAM Hotels / Grupo KTRC*
*Para más información técnica consultar: `PAM_UUID_Agent_Architecture.md` y `PAM_UUID_Agent_CHANGES_v2.md`*
