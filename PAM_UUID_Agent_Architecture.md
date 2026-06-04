# PAM Hotels — Agente de Trazabilidad UUID

## Arquitectura Técnica para Claude Code

---

## 1\. Contexto del Negocio

**Cliente:** PAM Hotels (Grupo KTRC / Sociedad 3400\)  
**Objetivo:** Dado un conjunto de cuentas de gasto del libro mayor (FBL3N), determinar automáticamente:

1. Qué **proveedor** y **factura (CFDI)** generaron cada gasto  
2. Con qué **cuenta bancaria** se pagó y en qué **fecha**  
3. Usar el **UUID fiscal** como llave primaria de trazabilidad  
4. Cuando el UUID está ausente, aplicar **matching difuso** por importe \+ fecha \+ concepto

**Ecosistema SAP del cliente:** ECC / en migración a S/4HANA  
**Transacciones fuente:** FBL3N (libro mayor), FBL1N (partidas proveedor), datos maestros acreedor  
**Plataforma destino:** Claude Agents \+ subagentes (PoC) → migración futura a Vertex AI / Google

---

## 2\. Fuentes de Datos — Estructura Real Validada

### 2.1 Archivo A: `AuxGastos` — Libro Mayor (FBL3N)

**Hoja:** `Febrero_2020_KTRC` (u otras del mismo patrón `MES_YYYY_KTRC`)  
**Formato origen:** Excel exportado desde SAP ALV Grid  
**Volumen:** \~2,400 registros por mes por grupo de sociedades  
**Encoding:** UTF-8 con caracteres especiales en headers (ej. `Nº`, `Ej./mes`)

| Índice Col | Nombre SAP | Nombre Normalizado | Tipo | Notas |
| :---- | :---- | :---- | :---- | :---- |
| 0 | `Soc.` | `sociedad` | INT | Ej: 3400, 3401, 3402 |
| 1 | `Fe.contab.` | `fecha_contab` | STR `DD.MM.YYYY` | Fecha de contabilización |
| 2 | `Ejercicio` | `ejercicio` | INT | Año fiscal |
| 3 | `Ej./mes` | `periodo` | DATETIME | Periodo contable |
| 4 | `Pos.` | `posicion` | INT | Posición en documento |
| 5 | `Nº doc.` | `nro_documento` | INT | BELNR — llave de cruce con FBL1N |
| 6 | `Clase` | `clase_doc` | STR | WA, RE, KR, etc. |
| 7 | `ClvCT` | `clave_ct` | INT | Clave de contabilización |
| 8 | `LibrMay` | `cuenta_gl` | INT | Cuenta de mayor (ej: 71101060\) |
| 9 | `Txt.brv.` | `descripcion_cuenta` | STR | Texto breve cuenta — **clave para matching difuso** |
| 10 | `Importe` | `importe` | FLOAT | En moneda documento |
| 11 | `Moneda` | `moneda` | STR | MXN, USD |
| 12 | `Impte.ML` | `importe_ml` | FLOAT | **Importe en moneda local — llave de matching** |
| 13 | `Mon.local` | `moneda_local` | STR | MXN |
| 14 | `Ind.imp.` | `indicador_iva` | STR | Indicador IVA |
| 15 | `Folio Fisc` | `folio_fiscal` | STR | Folio fiscal (campo adicional) |
| 16 | `UUID Pos.` | `uuid` | STR | **UUID CFDI — llave primaria. PUEDE ESTAR VACÍO (84% de registros)** |
| 17 | `Cta.CP` | `cta_contrapartida` | INT | Cuenta contrapartida |
| 18 | `Acreedor` | `acreedor_id` | INT | Número de proveedor SAP |
| 19 | `Nombre` | `nombre_proveedor` | STR | Nombre del proveedor |
| 20 | `SocGLAsoc.` | `soc_gl` | STR | Sociedad GL asociada |
| 21 | `Anulación` | `anulacion` | STR | Indicador anulación |
| 22 | `Registrado` | `fecha_registro` | STR | Fecha registro |
| 23 | `Referencia` | `referencia` | STR | Referencia externa — usar en fallback |
| 24 | `Asign.` | `asignacion` | INT | Asignación (YYYYMMDD numérico) |
| 25 | `Texto` | `texto` | STR | Texto posición — **clave para matching semántico** |
| 26 | `Ce.coste` | `centro_costo` | INT | Centro de costo PAM (ej: 340312890\) |
| 27 | `Usuario` | `usuario` | STR | Usuario SAP |
| 28 | `Número del documento de compensación` | `doc_compensacion` | INT | Doc. compensación — puede estar vacío |

**⚠️ Hallazgo crítico:** Solo el **16% de registros tienen UUID** (378 de 2,405). El 84% requiere matching difuso.

---

### 2.2 Archivo B: `3400` — Partidas de Proveedor / Pagos (FBL1N)

**Hoja:** `3400 2020`  
**Formato origen:** Excel exportado desde SAP ALV Grid  
**Volumen:** \~37,500 registros (histórico multi-año, sociedad 3400\)

| Índice Col | Nombre SAP | Nombre Normalizado | Tipo | Notas |
| :---- | :---- | :---- | :---- | :---- |
| 0 | `Referencia` | `referencia` | STR | Referencia del documento externo |
| 3 | `Cuenta` | `acreedor_id` | INT/STR | Número proveedor o cuenta GL |
| 4 | `Descripción de proveedor` | `nombre_proveedor` | STR | Nombre proveedor |
| 5 | `CT` | `clave_ct` | INT | Clave contabilización |
| 6 | `Nº doc.` | `nro_documento` | INT | **BELNR — llave de cruce con FBL3N** |
| 7 | `SocGLA` | `soc_gl` | STR | Sociedad GL |
| 8 | `Clase` | `clase_doc` | STR | Clase documento (RE, ZP, ZZ) |
| 10 | `Fecha doc.` | `fecha_documento` | STR `DD.MM.YYYY` | Fecha del documento original |
| 11 | `Fe.contab.` | `fecha_contab` | STR `DD.MM.YYYY` | Fecha contabilización |
| 13 | `Fecha pago` | `fecha_pago` | STR `DD.MM.YYYY` | **Fecha real de pago — dato objetivo** |
| 18 | `Impte.MD` | `importe` | FLOAT | Importe (negativo \= pago) |
| 19 | `Mon.` | `moneda` | STR | MXN, USD |
| 20 | `Soc.` | `sociedad` | INT | Sociedad |
| 21 | `Cta.CP` | `cta_banco` | INT | **Cuenta bancaria SAP donde se pagó** |
| 22 | `Texto` | `texto` | STR | Descripción del pago |
| 67 | `II` | `indicador_ii` | STR |  |
| 68 | `VP` | `via_pago` | STR | Vía de pago |
| 69 | `CPag` | `condicion_pago` | STR | PO30, PO60, etc. |
| 70 | `Usuario` | `usuario` | STR |  |
| 71 | `Folio Fiscal` | `uuid` | STR | **UUID del CFDI — llave primaria de matching** |
| 72 | `Compens.` | `fecha_compensacion` | STR | Fecha compensación |
| 73 | `Doc.comp.` | `doc_compensacion` | INT | **Documento de compensación (pago)** |
| 74 | `Anul.con` | `anulacion` | STR |  |

**⚠️ Nota:** La columna `Cta.CP` (col 21\) contiene el número de cuenta SAP interno (ej: `11210050`, `19999020`). Mapear contra el catálogo de bancos PAM para obtener nombre de institución y cuenta bancaria real.

---

### 2.3 Archivo C: `Acreedor` — Datos Maestros de Proveedores

**Hoja:** `Datos Maestros acreedores`  
**Volumen:** \~24,600 registros (maestro completo multi-sociedad)

| Índice Col | Nombre SAP | Nombre Normalizado | Tipo | Notas |
| :---- | :---- | :---- | :---- | :---- |
| 0 | `Soc.` | `sociedad` | INT |  |
| 1 | `Acreedor` | `acreedor_id` | INT | **Llave de lookup desde FBL3N.acreedor\_id** |
| 4 | `Nombre 1` | `nombre_proveedor` | STR | Razón social |
| 13 | `Nº ident.fis.1` | `rfc` | STR | **RFC del proveedor — dato fiscal clave** |
| 16 | `Clave de ramo industrial` | `ramo` | STR | Alimentos, Servicios, etc. |
| 18 | `Cta.asoc.` | `cuenta_asociada` | INT |  |
| 19 | `CPag` | `condicion_pago` | STR |  |
| 22 | `SocGLA` | `soc_gl` | STR |  |

---

### 2.4 Output Objetivo: `ReporteFinal` — Formato Validado por el Cliente

**Hoja referencia:** `EJEMPLO` / `Enero_2020_KTRC_v2`

El output final debe tener las siguientes columnas **en este orden**:

Sociedad | Fe.contab. | Ejercicio | Ej./mes | Pos. | Nº doc. | Clase | ClvCT |

LibrMay  | Cta.CP     | Txt.brv.  | Importe | Moneda | Impte.ML | Mon.local |

Ind.imp. | Folio Fisc | UUID Pos. | RFC Emisor | Acreedor | Nombre |

SocGLAsoc. | Anulación | Registrado | Referencia | Asign. |

\[ENRIQUECIDOS por el agente:\]

Subtotal | IVA | Importe en Moneda Documento | Documento contable Registro |

Texto | Monto Pagado | Fecha de Pago | Forma de Pago |

Documento contable de pago | Institución financiera |

Cuenta bancaria donde se efectuo el pago | Ce.coste | Usuario

---

## 3\. Arquitectura del Agente

### 3.1 Diagrama de Flujo

┌─────────────────────────────────────────────────────────────────┐

│                 ORQUESTADOR PRINCIPAL                           │

│                 orchestrator\_agent.py                           │

│                                                                 │

│  1\. Recibe: archivos AuxGastos, FBL1N (3400), Acreedor         │

│  2\. Invoca subagentes en secuencia                              │

│  3\. Consolida resultado final                                   │

│  4\. Genera reporte Excel con métricas                           │

└──────┬──────────────────────────────────────────────────────────┘

       │

       ├──► SubAgente 1: PARSER & NORMALIZER

       │    agents/parser\_agent.py

       │    • Lee los 3 archivos Excel

       │    • Normaliza headers (encoding UTF-8, nombres SAP → snake\_case)

       │    • Parsea fechas DD.MM.YYYY → date objects

       │    • Filtra: solo cuentas GL de gasto (7XXXXXXX)

       │    • Output: DataFrames limpios en memoria

       │

       ├──► SubAgente 2: MAESTRO ENRICHER

       │    agents/maestro\_agent.py

       │    • Carga catálogo de acreedores

       │    • Para cada registro FBL3N: lookup RFC por acreedor\_id \+ sociedad

       │    • Enriquece FBL3N con RFC cuando no viene en el archivo

       │    • Output: FBL3N enriquecido con RFC

       │

       ├──► SubAgente 3: UUID MATCHER (Matching Exacto)

       │    agents/uuid\_matcher\_agent.py

       │    • Estrategia 1: FBL3N.uuid → FBL1N.uuid (match directo)

       │    • Estrategia 2: FBL3N.nro\_documento → FBL1N.nro\_documento

       │    • Marca registros resueltos con confianza \= "EXACTO"

       │    • Output: Registros resueltos \+ pendientes sin match

       │

       ├──► SubAgente 4: FUZZY MATCHER (Matching Difuso)

       │    agents/fuzzy\_matcher\_agent.py

       │    • Solo procesa registros NO resueltos por SubAgente 3

       │    • Filtro 1: fecha\_contab ± 5 días naturales

       │    • Filtro 2: importe\_ml con holgura ± 1% (permite diferencias de centavos)

       │    • Filtro 3: acreedor\_id coincide (cuando disponible)

       │    • Ranking semántico: similitud entre texto/descripcion\_cuenta vs texto FBL1N

       │    • Marca con confianza \= "DIFUSO\_ALTO" | "DIFUSO\_MEDIO" | "NO\_RESUELTO"

       │    • Output: Registros con mejor candidato \+ score de confianza

       │

       └──► SubAgente 5: REPORT BUILDER

            agents/report\_builder\_agent.py

            • Consolida todos los matches (exactos \+ difusos \+ no resueltos)

            • Construye DataFrame con columnas del formato ReporteFinal

            • Genera Excel con:

              \- Sheet "Resultado": todos los registros enriquecidos

              \- Sheet "Métricas": estadísticas de matching

              \- Sheet "Pendientes": registros NO\_RESUELTO para revisión manual

            • Color coding: verde=EXACTO, amarillo=DIFUSO, rojo=NO\_RESUELTO

---

### 3.2 Lógica de Matching — Detalle

\# PRIORIDAD DE MATCHING (aplicar en orden, detenerse en primer éxito)

ESTRATEGIA\_1 \= "UUID\_DIRECTO"

\# Condición: AuxGastos.uuid IS NOT NULL

\# Join: AuxGastos.uuid \== FBL1N.uuid (col 71\)

\# Confianza: EXACTO ✅

ESTRATEGIA\_2 \= "DOCUMENTO\_CRUCE"  

\# Condición: AuxGastos.nro\_documento tiene entrada en FBL1N

\# Join: AuxGastos.nro\_documento \== FBL1N.nro\_documento (col 6\)

\# Confianza: EXACTO ✅

ESTRATEGIA\_3 \= "FUZZY\_CON\_ACREEDOR"

\# Condición: UUID vacío, pero acreedor\_id está presente

\# Filtros combinados:

\#   \- acreedor\_id coincide

\#   \- |fecha\_contab\_aux \- fecha\_doc\_fbl1n| \<= 5 días

\#   \- |importe\_ml\_aux \- abs(importe\_fbl1n)| / importe\_ml\_aux \<= 0.01 (±1%)

\# Si candidatos \> 1: aplicar similitud de texto (difflib o embeddings)

\# Confianza: DIFUSO\_ALTO si score\_texto \> 0.7, DIFUSO\_MEDIO si \> 0.4

ESTRATEGIA\_4 \= "FUZZY\_SIN\_ACREEDOR"

\# Condición: UUID vacío Y acreedor\_id vacío

\# Filtros:

\#   \- |fecha| \<= 3 días

\#   \- |importe| \<= 1%

\#   \- referencia o texto contienen tokens comunes

\# Confianza: DIFUSO\_MEDIO o DIFUSO\_BAJO

\# ⚠️ Siempre requiere revisión manual

FALLBACK \= "NO\_RESUELTO"

\# Confianza: NO\_RESUELTO ❌ — pasa a sheet "Pendientes"

---

### 3.3 Enriquecimiento del Output

Para cada registro matcheado, extraer de FBL1N y agregar al output:

CAMPOS\_DESDE\_FBL1N \= {

    "fecha\_pago":         "col 13 — Fecha pago",

    "cta\_banco":          "col 21 — Cta.CP (cuenta SAP del banco)",

    "texto\_pago":         "col 22 — Texto del documento de pago",

    "doc\_compensacion":   "col 73 — Doc.comp. (número documento pago)",

    "fecha\_compensacion": "col 72 — Compens.",

    "uuid\_fbl1n":         "col 71 — Folio Fiscal (confirma match)",

    "importe\_pagado":     "col 18 — Impte.MD",

}

CAMPOS\_DESDE\_ACREEDOR \= {

    "rfc\_emisor": "col 13 — Nº ident.fis.1",

    "nombre\_normalizado": "col 4 — Nombre 1",

}

---

## 4\. Estructura de Archivos del Proyecto

pam-uuid-agent/

│

├── README.md                          \# Este archivo

│

├── main.py                            \# Entry point — ejecutar el agente completo

│

├── config/

│   ├── settings.py                    \# Parámetros configurables (tolerancias, rutas)

│   └── column\_maps.py                 \# Mapeo de índices de columnas SAP → nombres normalizados

│

├── agents/

│   ├── orchestrator\_agent.py          \# Orquestador principal

│   ├── parser\_agent.py                \# SubAgente 1: lectura y normalización

│   ├── maestro\_agent.py               \# SubAgente 2: enriquecimiento RFC

│   ├── uuid\_matcher\_agent.py          \# SubAgente 3: matching exacto

│   ├── fuzzy\_matcher\_agent.py         \# SubAgente 4: matching difuso

│   └── report\_builder\_agent.py        \# SubAgente 5: generación reporte

│

├── utils/

│   ├── excel\_reader.py                \# Lectura robusta de Excel con encoding fix

│   ├── date\_parser.py                 \# Parseo fechas formato SAP (DD.MM.YYYY)

│   ├── text\_similarity.py             \# Funciones de similitud semántica

│   └── validators.py                  \# Validación de UUIDs, RFCs, importes

│

├── data/

│   ├── input/                         \# Archivos Excel de entrada (no commitear datos reales)

│   │   ├── AuxGastos\_YYYYMM.xlsx

│   │   ├── FBL1N\_SOCIEDAD\_YYYYMM.xlsx

│   │   └── Acreedor\_maestro.xlsx

│   └── output/                        \# Reportes generados

│       └── ReporteFinal\_SOCIEDAD\_YYYYMM.xlsx

│

├── tests/

│   ├── test\_parser.py

│   ├── test\_uuid\_matcher.py

│   ├── test\_fuzzy\_matcher.py

│   └── fixtures/                      \# Datos de prueba anonimizados

│

└── requirements.txt

---

## 5\. Dependencias (requirements.txt)

pandas\>=2.0.0

openpyxl\>=3.1.0

xlsxwriter\>=3.1.0        \# Para output con formato y colores

python-dateutil\>=2.8.0   \# Parseo flexible de fechas

difflib                  \# stdlib — similitud de strings (no requiere instalación)

anthropic\>=0.25.0        \# Claude API para matching semántico avanzado (opcional en PoC)

python-dotenv\>=1.0.0     \# Variables de entorno para API keys

tqdm\>=4.0.0              \# Progress bars para volúmenes grandes

---

## 6\. Parámetros de Configuración (`config/settings.py`)

\# Tolerancias de matching difuso

FUZZY\_FECHA\_TOLERANCIA\_DIAS \= 5       \# ± días para match de fecha

FUZZY\_IMPORTE\_TOLERANCIA\_PCT \= 0.01   \# ±1% para match de importe

FUZZY\_TEXT\_SCORE\_ALTO \= 0.70          \# Score mínimo para DIFUSO\_ALTO

FUZZY\_TEXT\_SCORE\_MEDIO \= 0.40         \# Score mínimo para DIFUSO\_MEDIO

\# Filtros de cuentas GL a procesar (solo gastos)

CUENTAS\_GL\_GASTO\_PREFIX \= \["71", "72", "73", "74", "75", "76", "77"\]

\# Columnas clave por archivo (índice base 0\)

AUXGASTOS\_COLS \= {

    "sociedad": 0, "fecha\_contab": 1, "ejercicio": 2, "periodo": 3,

    "posicion": 4, "nro\_documento": 5, "clase\_doc": 6, "clave\_ct": 7,

    "cuenta\_gl": 8, "descripcion\_cuenta": 9, "importe": 10, "moneda": 11,

    "importe\_ml": 12, "moneda\_local": 13, "indicador\_iva": 14,

    "folio\_fiscal": 15, "uuid": 16, "cta\_contrapartida": 17,

    "acreedor\_id": 18, "nombre\_proveedor": 19, "soc\_gl": 20,

    "anulacion": 21, "fecha\_registro": 22, "referencia": 23,

    "asignacion": 24, "texto": 25, "centro\_costo": 26,

    "usuario": 27, "doc\_compensacion": 28

}

FBL1N\_COLS \= {

    "referencia": 0, "acreedor\_id": 3, "nombre\_proveedor": 4,

    "clave\_ct": 5, "nro\_documento": 6, "soc\_gl": 7, "clase\_doc": 8,

    "fecha\_documento": 10, "fecha\_contab": 11, "fecha\_pago": 13,

    "importe": 18, "moneda": 19, "sociedad": 20, "cta\_banco": 21,

    "texto": 22, "usuario": 70, "uuid": 71,

    "fecha\_compensacion": 72, "doc\_compensacion": 73

}

ACREEDOR\_COLS \= {

    "sociedad": 0, "acreedor\_id": 1, "nombre\_proveedor": 4,

    "rfc": 13, "ramo": 16, "condicion\_pago": 19, "soc\_gl": 22

}

\# Configuración de output Excel

OUTPUT\_COLORS \= {

    "EXACTO":       "C6EFCE",   \# Verde claro

    "DIFUSO\_ALTO":  "FFEB9C",   \# Amarillo

    "DIFUSO\_MEDIO": "FFCC99",   \# Naranja claro

    "DIFUSO\_BAJO":  "FFC7CE",   \# Rojo claro

    "NO\_RESUELTO":  "FF0000",   \# Rojo

}

---

## 7\. Lógica de Parsing — Consideraciones Especiales

### 7.1 Encoding de Headers SAP

Los headers exportados de SAP tienen caracteres corruptos por encoding:

\# Headers problemáticos observados en los archivos reales:

"NÂº doc."     → normalizar a "nro\_documento"

"AnulaciÃ³n"  → normalizar a "anulacion"

"NÃºmero del documento de compensaciÃ³n" → normalizar a "doc\_compensacion"

\# Solución: usar índices de columna (ya mapeados en settings.py),

\# NO depender del nombre del header para el procesamiento.

### 7.2 Fechas en Formato SAP

\# Formato SAP: "DD.MM.YYYY" como string

\# Solución:

from datetime import datetime

def parse\_sap\_date(val):

    if isinstance(val, datetime):

        return val.date()

    if isinstance(val, str) and len(val) \== 10:

        return datetime.strptime(val, "%d.%m.%Y").date()

    return None

### 7.3 Importes Negativos en FBL1N

\# FBL1N registra pagos como negativos (ej: \-15420.50)

\# Al comparar con FBL3N (positivos), usar valor absoluto:

abs(fbl1n\_row\["importe"\]) vs auxgastos\_row\["importe\_ml"\]

### 7.4 Múltiples Hojas en AuxGastos

\# El archivo puede tener múltiples hojas (Enero\_2020\_KTRC, Febrero\_2020\_KTRC, etc.)

\# Procesar todas las hojas que cumplan el patrón: r'^\[A-Za-z\]+\_\\d{4}\_KTRC$'

\# Hoja "Hoja1" y similares → ignorar

### 7.5 Filas Vacías y Headers Dobles en FBL1N (3400)

\# El archivo 3400.xlsx tiene:

\# \- Row 1: Headers principales

\# \- Row 2: Fila vacía (skip)

\# \- Row 3+: Datos

\# Usar skiprows=\[1\] al leer con pandas, o min\_row=3 con openpyxl

---

## 8\. Métricas de Éxito del PoC

El reporte final debe incluir una hoja `Métricas` con:

| Métrica | Descripción |
| :---- | :---- |
| `total_registros` | Total registros FBL3N procesados |
| `con_uuid_origen` | Registros que tenían UUID en FBL3N |
| `match_exacto_uuid` | Resueltos por Estrategia 1 (UUID directo) |
| `match_exacto_doc` | Resueltos por Estrategia 2 (nro\_documento) |
| `match_difuso_alto` | Resueltos con confianza DIFUSO\_ALTO |
| `match_difuso_medio` | Resueltos con confianza DIFUSO\_MEDIO |
| `no_resuelto` | Sin match — requieren revisión manual |
| `pct_resolucion` | % total resueltos (exacto \+ difuso\_alto) |

**Target mínimo para validar PoC:** ≥ 80% de resolución automática.

---

## 9\. Casos Edge Documentados

| Caso | Descripción | Manejo |
| :---- | :---- | :---- |
| UUID en FBL3N pero no en FBL1N | CFDI registrado pero pago sin UUID | Fallback a Estrategia 2 (nro\_documento) |
| Proveedor extranjero (sin RFC) | Ej: AMERICAN AIRLINES, BOOKER SOFTWARE | RFC \= "XEXX010101000" (RFC genérico SAT para extranjeros) |
| Documento de compensación masivo | Un solo DocComp para múltiples facturas (ej: 880031072\) | Mantener todos los matches, registrar DocComp compartido |
| Importe con diferencia de IVA | FBL3N registra neto, FBL1N registra total con IVA | Ajustar tolerancia o comparar contra Subtotal del CFDI |
| Anulaciones | Registros con `Anulación` poblado | Excluir del matching, marcar en output como ANULADO |
| Cuentas GL de clearing (19999020) | No son gastos — son cuentas puente | Filtrar: no procesar cuentas que comiencen con 1X o 2X |

---

## 10\. Próximos Pasos Post-PoC

Una vez validada la lógica en Claude:

1. **Migración a Google Cloud:** Vertex AI Agent Builder \+ BigQuery como motor de datos  
2. **Automatización de extracción SAP:** Scheduled job via SAP Integration Suite → Google Drive  
3. **Interfaz de usuario:** Google Chat Bot o Looker Studio para que contabilidad ejecute el proceso sin código  
4. **Extensión a otros casos:** IVA No Acreditable por WBS, conciliación bancaria, auditoría CFDI vs. contabilidad  
5. **Relevancia S/4HANA:** En S/4HANA \+ ACDOCA, el UUID queda almacenado en el Universal Journal — este proceso se simplifica sustancialmente post-migración

---

## 11\. Datos de Muestra para Pruebas

Los archivos reales del cliente se encuentran en:

- `data/input/AuxGastos.xlsx` → hoja `Febrero_2020_KTRC` (\~2,405 filas)  
- `data/input/3400.xlsx` → hoja `3400 2020` (\~37,567 filas)  
- `data/input/Acreedor.xlsx` → hoja `Datos Maestros acreedores` (\~24,592 filas)  
- `data/input/ReporteFinal.xlsx` → hoja `EJEMPLO` y `Enero_2020_KTRC_v2` (formato objetivo)

**⚠️ IMPORTANTE:** Estos archivos contienen datos reales del cliente. No commitear a repositorio público. Usar `.gitignore` para excluir `data/input/` y `data/output/`.

---

*Documento generado para Claude Code — PAM Hotels S/4HANA Migration Project*  
*Versión: 1.0 | Fecha: Junio 2026*  
