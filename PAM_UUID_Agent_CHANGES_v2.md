# PAM UUID Agent — Cambios v2: Integración Archivo Ecommerce

## Instrucciones de modificación para Claude Code

**Contexto:** Este documento complementa `PAM_UUID_Agent_Architecture.md` (v1).  
El agente original definía 5 subagentes. Este documento agrega el **Archivo D (Ecommerce)**  
y redefine los **Pasos 6 y 7** del flujo de matching para incorporarlo correctamente.  
Implementar como **Jupyter Notebook** (`PAM_UUID_Agent.ipynb`) en lugar de múltiples `.py`.

---

## 1\. Nuevo Archivo Fuente — Archivo D: Ecommerce (CFDIs Recibidos)

### Metadata del archivo real

- **Nombre:** `ecommerce.xlsx`  
- **Hoja:** `CIK130405M86 CFULL Recibidos 20`  
- **Volumen:** 116,335 registros totales / 108,599 vigentes / 7,736 cancelados  
- **RFC Receptor único:** `CIK130405M86` (CORPORACION INMOBILIARIA KTRC SA DE CV) — es el RFC del grupo PAM que recibe todas las facturas  
- **Registros con diferencia FX (Total XML ≠ Total MXN):** 8,270 — facturas en USD que requieren conversión

### Mapeo de columnas (índice base 0\)

| Índice Col | Nombre Original | Nombre Normalizado | Tipo | Notas Críticas |
| :---- | :---- | :---- | :---- | :---- |
| 0 | `XML` | `xml_filename` | STR | Nombre del archivo XML fuente |
| 1 | `UUID` | `uuid` | STR | **Llave primaria. SIEMPRE en minúsculas — normalizar a UPPER antes de join** |
| 2 | `Estatus` | `estatus` | STR | `Vigente` / `Cancelado` — **filtrar solo Vigente** |
| 3 | `Fecha Cancelación` | `fecha_cancelacion` | DATE | Solo si Cancelado |
| 8 | `Tipo` | `tipo_cfdi` | STR | `I`\=Ingreso, `E`\=Egreso, `P`\=Pago |
| 9 | `Serie` | `serie` | STR | Serie del CFDI |
| 10 | `Folio` | `folio` | STR | Folio del CFDI |
| 11 | `Emisión` | `fecha_emision` | INT→DATE | **Fecha en formato serial Excel (ej: 43831 \= 2020-01-01). Convertir con:** `datetime(1899,12,30) + timedelta(days=valor)` |
| 21 | `Fecha timbrado` | `fecha_timbrado` | DATETIME | Ya viene como datetime de Python |
| 23 | `Receptor RFC` | `receptor_rfc` | STR | Siempre `CIK130405M86` en este archivo |
| 24 | `Receptor Nombre` | `receptor_nombre` | STR | Nombre PAM |
| 31 | `Emisor RFC` | `emisor_rfc` | STR | **RFC del proveedor — dato fiscal objetivo del enriquecimiento** |
| 32 | `Emisor Nombre` | `emisor_nombre` | STR | **Razón social del proveedor** |
| 40 | `Conceptos descripción` | `concepto` | STR | Descripción de bienes/servicios — **clave para matching semántico difuso** |
| 45 | `SubTotal` | `subtotal` | FLOAT | Antes de IVA |
| 52 | `IVA` | `iva` | FLOAT | IVA trasladado |
| 60 | `Total original XML` | `total_xml` | FLOAT | Total en moneda del CFDI (puede ser USD) |
| 61 | `Total en MXN` | `total_mxn` | FLOAT | **Total convertido a MXN — usar este para matching de importe** |
| 62 | `TipoCambio` | `tipo_cambio` | FLOAT | Factor de conversión FX |
| 63 | `Moneda` | `moneda` | STR | `MXN`, `USD`, etc. |
| 65 | `Forma Pago` | `forma_pago` | STR | Código SAT (ej: `03`\=Transferencia) |
| 66 | `Forma Pago Descripción` | `forma_pago_desc` | STR | Descripción legible |
| 67 | `Método Pago` | `metodo_pago` | STR | `PUE`\=Una exhibición, `PPD`\=Diferido |
| 68 | `Método Pago Descripción` | `metodo_pago_desc` | STR |  |

### Constante a agregar en `settings` del Notebook

ECOMMERCE\_COLS \= {

    "xml\_filename":    0,

    "uuid":            1,   \# LOWERCASE — normalizar a .upper() antes de join

    "estatus":         2,   \# Filtrar: solo 'Vigente'

    "fecha\_cancelacion": 3,

    "tipo\_cfdi":       8,

    "serie":           9,

    "folio":           10,

    "fecha\_emision":   11,  \# Serial Excel → convertir a date

    "fecha\_timbrado":  21,  \# Ya es datetime

    "receptor\_rfc":    23,

    "receptor\_nombre": 24,

    "emisor\_rfc":      31,  \# RFC del proveedor — DATO OBJETIVO

    "emisor\_nombre":   32,  \# Razón social proveedor — DATO OBJETIVO

    "concepto":        40,  \# Para matching semántico difuso

    "subtotal":        45,

    "iva":             52,

    "total\_xml":       60,

    "total\_mxn":       61,  \# Usar este para matching de importe (ya en MXN)

    "tipo\_cambio":     62,

    "moneda":          63,

    "forma\_pago":      65,

    "forma\_pago\_desc": 66,

    "metodo\_pago":     67,

    "metodo\_pago\_desc":68,

}

\# Filtros de carga obligatorios para Ecommerce

ECOMMERCE\_SHEET \= "CIK130405M86 CFULL Recibidos 20"

ECOMMERCE\_ESTATUS\_VALIDO \= "Vigente"   \# Excluir Cancelados (7,736 registros)

---

## 2\. Advertencias Críticas de Datos — Leer Antes de Programar

### ⚠️ A. UUID en minúsculas vs MAYÚSCULAS

\# Ecommerce UUID:  '03c5983b-6abb-40fc-ac25-bb88bb7fd9ad'  ← minúsculas

\# AuxGastos UUID:  'AE9E7899-7081-4B1F-8D4D-C23E50D955C4'  ← MAYÚSCULAS

\# FBL1N UUID:      '027ECA4F-DB6E-5A8C-6B65-73A149A5B103'  ← MAYÚSCULAS

\# SOLUCIÓN OBLIGATORIA — normalizar en el momento de carga:

df\_ecommerce\['uuid'\] \= df\_ecommerce\['uuid'\].str.upper().str.strip()

df\_auxgastos\['uuid'\] \= df\_auxgastos\['uuid'\].str.upper().str.strip()

df\_fbl1n\['uuid'\]     \= df\_fbl1n\['uuid'\].str.upper().str.strip()

### ⚠️ B. Fecha de Emisión en formato serial Excel

\# La columna 'Emisión' (col 11\) llega como entero: 43831, 43832, etc.

\# NO es una fecha Python. Convertir así:

from datetime import datetime, timedelta

EXCEL\_EPOCH \= datetime(1899, 12, 30\)

def excel\_serial\_to\_date(serial):

    if isinstance(serial, (int, float)) and serial \> 0:

        return (EXCEL\_EPOCH \+ timedelta(days=int(serial))).date()

    elif isinstance(serial, datetime):

        return serial.date()

    return None

df\_ecommerce\['fecha\_emision'\] \= df\_ecommerce\['fecha\_emision\_raw'\].apply(excel\_serial\_to\_date)

### ⚠️ C. Importe a usar para matching

\# Usar SIEMPRE 'Total en MXN' (col 61\) para comparación con AuxGastos (que está en MXN)

\# NO usar 'Total original XML' (col 60\) cuando moneda \!= MXN

\# Hay 8,270 registros con diferencia FX (facturas en USD)

\# Comparación correcta:

abs(ecommerce\_row\['total\_mxn'\] \- auxgastos\_row\['importe\_ml'\]) / auxgastos\_row\['importe\_ml'\] \<= FUZZY\_IMPORTE\_TOLERANCIA\_PCT

### ⚠️ D. Volumen y performance

\# 116,335 registros totales en ecommerce — cargar solo Vigentes (108,599)

\# Para matching difuso, pre-indexar por:

\#   \- emisor\_rfc (para filtrar candidatos por proveedor)

\#   \- fecha\_emision por mes (para limitar ventana temporal)

\# Usar pandas merge/groupby, NO loops sobre el DataFrame completo

\# Carga optimizada:

df\_eco \= pd.read\_excel('ecommerce.xlsx', 

                        sheet\_name=ECOMMERCE\_SHEET,

                        usecols=\[1,2,3,8,9,10,11,21,23,24,31,32,40,45,52,60,61,62,63,65,66,67,68\])

df\_eco \= df\_eco\[df\_eco.iloc\[:,1\] \== 'Vigente'\]  \# filtrar Cancelados inmediatamente

---

## 3\. Redefinición del Flujo — Pasos 6 y 7

El flujo completo del Notebook queda así. **Los pasos 1-5 no cambian** respecto a v1. Se agregan/redefinen pasos 6 y 7:

PASO 1: Carga y normalización de los 4 archivos

         AuxGastos \+ FBL1N (3400) \+ Acreedor \+ \[NUEVO\] Ecommerce

PASO 2: Enriquecimiento desde Maestro Acreedor

         AuxGastos.acreedor\_id → Acreedor.rfc (cuando no viene en FBL3N)

PASO 3: Matching Exacto UUID — AuxGastos → Ecommerce  \[NUEVO PASO 6\]

PASO 4: Matching Difuso — AuxGastos → Ecommerce        \[NUEVO PASO 6 fallback\]

PASO 5: Matching UUID → FBL1N (pagos)                 \[NUEVO PASO 7\]

PASO 6: Fallback Difuso → FBL1N                        \[NUEVO PASO 7 fallback\]

PASO 7: Consolidación y Reporte Final

---

## 4\. Paso 6 — Detalle Completo: AuxGastos ↔ Ecommerce

**Objetivo:** Para cada línea del auxiliar (FBL3N), obtener desde el archivo Ecommerce: `emisor_rfc`, `emisor_nombre`, `subtotal`, `iva`, `total_mxn`, `concepto`, `forma_pago`, `metodo_pago`

### Estrategia 6-A: UUID Exacto (AuxGastos → Ecommerce)

def paso6a\_uuid\_exacto(df\_aux, df\_eco):

    """

    Condición: df\_aux\['uuid'\] IS NOT NULL

    Join: df\_aux\['uuid'\] \== df\_eco\['uuid'\]  (ambos normalizados a UPPER)

    """

    aux\_con\_uuid \= df\_aux\[df\_aux\['uuid'\].notna() & (df\_aux\['uuid'\] \!= '')\]

    

    merged \= aux\_con\_uuid.merge(

        df\_eco\[\['uuid','emisor\_rfc','emisor\_nombre','subtotal','iva',

                'total\_mxn','concepto','forma\_pago','metodo\_pago','fecha\_emision'\]\],

        on='uuid',

        how='left'

    )

    

    merged\['confianza\_eco'\] \= merged\['emisor\_rfc'\].apply(

        lambda x: 'EXACTO\_UUID' if pd.notna(x) else 'UUID\_NO\_ENCONTRADO'

    )

    

    return merged

\# Registros resueltos por 6-A → pasar directo a Paso 7

\# Registros UUID\_NO\_ENCONTRADO → intentar 6-B

\# Registros sin UUID desde origen → intentar 6-C o 6-D

### Estrategia 6-B: Fallback por Número de Documento (cuando UUID no matchea)

def paso6b\_doc\_cruce(df\_aux\_pendientes, df\_eco):

    """

    Algunos registros tienen UUID en FBL3N pero el mismo UUID no está en Ecommerce.

    Intentar cruce por: acreedor\_id → emisor\_rfc (via maestro) \+ referencia/folio

    """

    \# Si df\_aux tiene acreedor\_id con RFC conocido, buscar en ecommerce por RFC \+ importe \+ fecha

    pass  \# Ver lógica en 6-C

### Estrategia 6-C: Matching Difuso con RFC \+ Fecha \+ Importe

def paso6c\_fuzzy\_rfc\_fecha\_importe(df\_aux\_pendientes, df\_eco, df\_acreedor):

    """

    Condición: UUID ausente en AuxGastos, pero acreedor\_id presente

    

    Paso 1: Obtener RFC del proveedor desde maestro (si no viene en FBL3N)

    Paso 2: Filtrar Ecommerce por emisor\_rfc \== RFC del proveedor

    Paso 3: Dentro de ese subconjunto, filtrar por fecha ± FUZZY\_FECHA\_TOLERANCIA\_DIAS

    Paso 4: Dentro de ese subconjunto, filtrar por importe ± FUZZY\_IMPORTE\_TOLERANCIA\_PCT

    Paso 5: Si quedan múltiples candidatos → ranking semántico por concepto

    """

    resultados \= \[\]

    

    for \_, aux\_row in df\_aux\_pendientes.iterrows():

        

        \# Obtener RFC del proveedor

        rfc\_proveedor \= aux\_row.get('rfc\_from\_maestro') or aux\_row.get('rfc\_emisor')

        if not rfc\_proveedor:

            resultados.append({\*\*aux\_row, 'confianza\_eco': 'SIN\_RFC'})

            continue

        

        \# Filtrar Ecommerce por RFC

        candidatos \= df\_eco\[df\_eco\['emisor\_rfc'\] \== rfc\_proveedor\].copy()

        if candidatos.empty:

            resultados.append({\*\*aux\_row, 'confianza\_eco': 'RFC\_NO\_EN\_ECOMMERCE'})

            continue

        

        \# Filtrar por fecha ± tolerancia

        fecha\_aux \= aux\_row\['fecha\_contab'\]  \# date object

        candidatos \= candidatos\[

            candidatos\['fecha\_emision'\].apply(

                lambda f: abs((f \- fecha\_aux).days) \<= FUZZY\_FECHA\_TOLERANCIA\_DIAS

            )

        \]

        

        \# Filtrar por importe ± tolerancia (usar total\_mxn)

        importe\_aux \= abs(aux\_row\['importe\_ml'\])

        candidatos \= candidatos\[

            candidatos\['total\_mxn'\].apply(

                lambda t: abs(t \- importe\_aux) / importe\_aux \<= FUZZY\_IMPORTE\_TOLERANCIA\_PCT

                if importe\_aux \> 0 else False

            )

        \]

        

        if candidatos.empty:

            resultados.append({\*\*aux\_row, 'confianza\_eco': 'SIN\_CANDIDATOS\_DIFUSO'})

            continue

        

        if len(candidatos) \== 1:

            best \= candidatos.iloc\[0\]

            confianza \= 'DIFUSO\_ALTO'

        else:

            \# Ranking semántico por similitud de concepto

            from difflib import SequenceMatcher

            texto\_aux \= str(aux\_row.get('texto', '') or aux\_row.get('descripcion\_cuenta', ''))

            candidatos \= candidatos.copy()

            candidatos\['score'\] \= candidatos\['concepto'\].apply(

                lambda c: SequenceMatcher(None, texto\_aux.upper(), str(c).upper()).ratio()

            )

            candidatos \= candidatos.sort\_values('score', ascending=False)

            best \= candidatos.iloc\[0\]

            score \= best\['score'\]

            confianza \= 'DIFUSO\_ALTO' if score \>= FUZZY\_TEXT\_SCORE\_ALTO else \\

                        'DIFUSO\_MEDIO' if score \>= FUZZY\_TEXT\_SCORE\_MEDIO else 'DIFUSO\_BAJO'

        

        resultados.append({

            \*\*aux\_row,

            'uuid\_ecommerce':  best\['uuid'\],     \# UUID encontrado — usar en Paso 7

            'emisor\_rfc':      best\['emisor\_rfc'\],

            'emisor\_nombre':   best\['emisor\_nombre'\],

            'subtotal':        best\['subtotal'\],

            'iva':             best\['iva'\],

            'total\_mxn\_eco':   best\['total\_mxn'\],

            'concepto\_eco':    best\['concepto'\],

            'forma\_pago':      best\['forma\_pago'\],

            'metodo\_pago':     best\['metodo\_pago'\],

            'confianza\_eco':   confianza,

        })

    

    return pd.DataFrame(resultados)

### Estrategia 6-D: Matching Difuso Solo por Fecha \+ Importe (sin RFC)

def paso6d\_fuzzy\_sin\_rfc(df\_aux\_pendientes, df\_eco):

    """

    Condición: UUID ausente Y RFC no disponible

    Mayor riesgo de falsos positivos — siempre marcar como DIFUSO\_BAJO

    Requiere revisión manual obligatoria

    """

    \# Misma lógica que 6-C pero sin filtro de RFC

    \# Aumentar restricción de importe a ±0.5% para compensar

    \# Siempre confianza \= 'DIFUSO\_BAJO'

    pass

### Output del Paso 6

Para cada registro de AuxGastos, el Paso 6 agrega estas columnas:

COLUMNAS\_PASO6 \= \[

    'uuid\_ecommerce',    \# UUID resuelto (puede venir de FBL3N original o de match difuso)

    'emisor\_rfc',        \# RFC del proveedor — DATO FISCAL OBJETIVO

    'emisor\_nombre',     \# Razón social del proveedor

    'subtotal',          \# Subtotal del CFDI

    'iva',               \# IVA del CFDI

    'total\_mxn\_eco',     \# Total en MXN del CFDI

    'concepto\_eco',      \# Concepto/descripción de la factura

    'forma\_pago',        \# Forma de pago SAT (03=Transferencia, etc.)

    'metodo\_pago',       \# PUE / PPD

    'confianza\_eco',     \# EXACTO\_UUID | DIFUSO\_ALTO | DIFUSO\_MEDIO | DIFUSO\_BAJO | SIN\_CANDIDATOS\_DIFUSO

\]

---

## 5\. Paso 7 — Detalle Completo: Resultado Paso 6 ↔ FBL1N (Pagos)

**Objetivo:** Usando el UUID ya resuelto en Paso 6, ligar con FBL1N para obtener: `fecha_pago`, `cta_banco`, `doc_compensacion`, `importe_pagado`

**Premisa:** El UUID del Paso 6 (`uuid_ecommerce`) es ahora la llave para buscar en FBL1N.

### Estrategia 7-A: UUID Exacto (uuid\_ecommerce → FBL1N.uuid)

def paso7a\_uuid\_a\_fbl1n(df\_paso6\_resultado, df\_fbl1n):

    """

    Usar uuid\_ecommerce (ya en UPPER) para buscar en FBL1N col 71 (Folio Fiscal)

    """

    merged \= df\_paso6\_resultado.merge(

        df\_fbl1n\[\['uuid','nro\_documento','fecha\_pago','cta\_banco',

                  'texto','doc\_compensacion','fecha\_compensacion','importe','sociedad'\]\],

        left\_on='uuid\_ecommerce',

        right\_on='uuid',

        how='left',

        suffixes=('', '\_fbl1n')

    )

    

    merged\['confianza\_pago'\] \= merged\['fecha\_pago'\].apply(

        lambda x: 'EXACTO\_UUID' if pd.notna(x) else 'UUID\_NO\_EN\_FBL1N'

    )

    

    return merged

### Estrategia 7-B: Fallback por nro\_documento (AuxGastos → FBL1N)

def paso7b\_doc\_a\_fbl1n(df\_pendientes\_7, df\_fbl1n):

    """

    Cuando uuid\_ecommerce no está en FBL1N

    Cruce: AuxGastos.nro\_documento \== FBL1N.nro\_documento

    """

    merged \= df\_pendientes\_7.merge(

        df\_fbl1n\[\['nro\_documento','fecha\_pago','cta\_banco',

                  'doc\_compensacion','fecha\_compensacion','importe'\]\],

        on='nro\_documento',

        how='left',

        suffixes=('', '\_fbl1n')

    )

    

    merged\['confianza\_pago'\] \= merged\['fecha\_pago'\].apply(

        lambda x: 'EXACTO\_DOC' if pd.notna(x) else 'SIN\_MATCH\_FBL1N'

    )

    

    return merged

### Estrategia 7-C: Fallback Difuso → FBL1N (RFC \+ Fecha \+ Importe)

def paso7c\_fuzzy\_a\_fbl1n(df\_pendientes\_7, df\_fbl1n):

    """

    Cuando UUID y nro\_documento fallan

    Condición: emisor\_rfc conocido desde Paso 6

    

    Cruce difuso:

      \- FBL1N.acreedor\_id → debe coincidir con proveedor (via maestro RFC)  

      \- |fecha\_pago\_fbl1n \- fecha\_contab\_aux| \<= tolerancia

      \- |importe\_fbl1n| ≈ importe\_ml\_aux (±1%)

    """

    \# Mismo patrón de lógica que paso6c pero sobre FBL1N

    \# confianza\_pago \= 'DIFUSO\_ALTO' | 'DIFUSO\_MEDIO' | 'SIN\_MATCH\_FBL1N'

    pass

### Output del Paso 7

Para cada registro, el Paso 7 agrega estas columnas al DataFrame final:

COLUMNAS\_PASO7 \= \[

    'fecha\_pago',           \# Fecha real en que se pagó la factura

    'cta\_banco',            \# Cuenta SAP del banco (ej: 11210050\)

    'doc\_compensacion',     \# Número del documento de pago (F110)

    'fecha\_compensacion',   \# Fecha de compensación en FBL1N

    'importe\_pagado',       \# Importe del pago en FBL1N

    'confianza\_pago',       \# EXACTO\_UUID | EXACTO\_DOC | DIFUSO\_ALTO | SIN\_MATCH\_FBL1N

\]

---

## 6\. Estructura del Notebook (`PAM_UUID_Agent.ipynb`)

Reemplaza los múltiples `.py` de v1. Organizar en celdas/secciones así:

📓 PAM\_UUID\_Agent.ipynb

│

├── \[Celda 1\] — CONFIGURACIÓN Y PARÁMETROS

│   settings dict, column maps para los 4 archivos, tolerancias

│

├── \[Celda 2\] — FUNCIONES UTILITARIAS

│   parse\_sap\_date(), excel\_serial\_to\_date(), normalize\_uuid(),

│   normalize\_importe(), text\_similarity()

│

├── \[Celda 3\] — PASO 1: CARGA DE ARCHIVOS

│   load\_auxgastos(), load\_fbl1n(), load\_acreedor(), load\_ecommerce()

│   → Print: shape, % UUID poblado, fechas min/max de cada archivo

│

├── \[Celda 4\] — PASO 2: ENRIQUECIMIENTO MAESTRO RFC

│   enrich\_from\_maestro()

│   → Print: % registros enriquecidos con RFC

│

├── \[Celda 5\] — PASO 6-A: MATCH UUID EXACTO → ECOMMERCE

│   paso6a\_uuid\_exacto()

│   → Print: N resueltos, N pendientes

│

├── \[Celda 6\] — PASO 6-C: MATCH DIFUSO → ECOMMERCE

│   paso6c\_fuzzy\_rfc\_fecha\_importe()  \[para registros sin UUID\]

│   paso6d\_fuzzy\_sin\_rfc()            \[para registros sin RFC\]

│   → Print: distribución por nivel de confianza

│

├── \[Celda 7\] — PASO 7-A: MATCH UUID → FBL1N

│   paso7a\_uuid\_a\_fbl1n()

│   → Print: N resueltos con fecha\_pago y banco

│

├── \[Celda 8\] — PASO 7-B/C: FALLBACK → FBL1N

│   paso7b\_doc\_a\_fbl1n()

│   paso7c\_fuzzy\_a\_fbl1n()

│   → Print: N adicionales resueltos

│

├── \[Celda 9\] — CONSOLIDACIÓN Y MÉTRICAS

│   Unir todos los DataFrames parciales

│   Calcular métricas de resolución por estrategia

│   → Print: tabla resumen de métricas

│

└── \[Celda 10\] — GENERACIÓN REPORTE EXCEL FINAL

    Columnas en orden del formato ReporteFinal validado por el cliente

    Color coding por nivel de confianza (verde/amarillo/naranja/rojo)

    Sheets: 'Resultado', 'Métricas', 'Pendientes'

    → Guardar: ReporteFinal\_SOCIEDAD\_YYYYMM.xlsx

---

## 7\. Métricas Adicionales para el Reporte (Celda 9\)

Agregar a la hoja `Métricas` del Excel final:

| Métrica | Descripción |
| :---- | :---- |
| `eco_total_vigentes` | CFDIs vigentes cargados del Ecommerce |
| `eco_uuid_exacto` | AuxGastos resueltos por UUID exacto en Ecommerce |
| `eco_difuso_alto` | Resueltos por matching difuso confianza alta |
| `eco_difuso_medio` | Resueltos por matching difuso confianza media |
| `eco_difuso_bajo` | Resueltos con confianza baja (requieren revisión) |
| `eco_sin_match` | Sin match en Ecommerce |
| `pago_uuid_exacto` | Registros con fecha\_pago resueltos por UUID en FBL1N |
| `pago_doc_exacto` | Resueltos por nro\_documento en FBL1N |
| `pago_difuso` | Resueltos por difuso en FBL1N |
| `pago_sin_match` | Sin fecha\_pago identificada |
| `pct_trazabilidad_completa` | % con RFC \+ UUID \+ banco \+ fecha\_pago resueltos |

**KPI objetivo del PoC:** `pct_trazabilidad_completa` ≥ 80%

---

## 8\. Columnas del Output Final (Orden Validado con Cliente)

COLUMNAS\_OUTPUT\_FINAL \= \[

    \# — Desde AuxGastos (FBL3N) —

    'sociedad', 'fecha\_contab', 'ejercicio', 'periodo', 'posicion',

    'nro\_documento', 'clase\_doc', 'clave\_ct', 'cuenta\_gl', 'cta\_contrapartida',

    'descripcion\_cuenta', 'importe', 'moneda', 'importe\_ml', 'moneda\_local',

    'indicador\_iva', 'folio\_fiscal', 'uuid',

    'acreedor\_id', 'centro\_costo', 'usuario',

    \# — Enriquecido desde Ecommerce (Paso 6\) —

    'emisor\_rfc',        \# RFC del proveedor

    'emisor\_nombre',     \# Razón social del proveedor

    'subtotal',          \# Subtotal CFDI

    'iva',               \# IVA CFDI

    'total\_mxn\_eco',     \# Total CFDI en MXN

    'concepto\_eco',      \# Concepto/descripción factura

    'forma\_pago',        \# Forma de pago SAT

    'metodo\_pago',       \# PUE / PPD

    'confianza\_eco',     \# Nivel de confianza match Ecommerce

    \# — Enriquecido desde FBL1N (Paso 7\) —

    'fecha\_pago',           \# Fecha real de pago

    'cta\_banco',            \# Cuenta bancaria SAP

    'doc\_compensacion',     \# Documento de pago (F110)

    'fecha\_compensacion',   \# Fecha compensación

    'importe\_pagado',       \# Monto pagado

    'confianza\_pago',       \# Nivel de confianza match pago

\]

---

## 9\. Casos Edge Adicionales (Ecommerce)

| Caso | Descripción | Manejo |
| :---- | :---- | :---- |
| CFDI cancelado en Ecommerce | `estatus == 'Cancelado'` | Excluir del pool de candidatos en la carga inicial |
| Factura en USD | `moneda != 'MXN'` → `total_xml != total_mxn` | Usar siempre `total_mxn` (col 61\) para matching |
| Un proveedor con múltiples conceptos por línea | `concepto` contiene lista separada por `|` (ej: `FRAMBUESA|MANGO|...`) | Tomar solo los primeros 100 chars para similitud, o tokenizar por `|` |
| RFC del emisor con variaciones tipográficas | Ej: espacios o guiones extra | `str.strip().upper()` en carga |
| Mismo UUID en múltiples filas de Ecommerce | Facturas con múltiples conceptos/líneas | Deduplicar por UUID antes del join, quedarse con primera ocurrencia o agregar conceptos |
| Fecha emisión \= 0 o nula en Ecommerce | Serial inválido | Devolver `None`, no bloquear el proceso |

---

## 10\. Archivos Fuente — Resumen Actualizado

| Archivo | Hoja | Filas | Columnas Clave | Rol |
| :---- | :---- | :---- | :---- | :---- |
| `AuxGastos.xlsx` | `Febrero_2020_KTRC` | \~2,405 | UUID (16%), importe\_ml, acreedor\_id | Registro de gastos FBL3N — **archivo base** |
| `3400.xlsx` | `3400 2020` | \~37,567 | uuid (col 71), fecha\_pago (col 13), cta\_banco (col 21\) | Partidas proveedor FBL1N — **datos de pago** |
| `Acreedor.xlsx` | `Datos Maestros acreedores` | \~24,592 | acreedor\_id, rfc | Maestro de proveedores — **enriquecimiento RFC** |
| `ecommerce.xlsx` | `CIK130405M86 CFULL Recibidos 20` | 108,599 vigentes | uuid (lowercase), emisor\_rfc, total\_mxn, fecha\_emision (serial) | CFDIs recibidos — **fuente de RFC y datos fiscales** |

---

*Complemento de `PAM_UUID_Agent_Architecture.md` v1*  
*Versión: 2.0 | Fecha: Junio 2026 | Para uso en Claude Code*  
