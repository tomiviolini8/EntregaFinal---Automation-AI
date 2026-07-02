# 🤖 Job Matcher IA — Ecosistema de Automatización Autónomo

> **Entrega Final · CoderHouse — IA Automatización · Tomás Violini**

Sistema autónomo construido en **n8n** que cada 3 días busca vacantes laborales en **dos fuentes** (Adzuna y Remotive), las puntúa contra mi CV usando IA (GPT-4o-mini), y me envía un digest con las mejores 10 — **pero solo después de mi aprobación** (Human-in-the-Loop). Toda la operación se registra en **Airtable**, que actúa como memoria y base de datos del sistema.

---

## 📑 Índice

- [Caso de uso](#-caso-de-uso)
- [Arquitectura](#️-arquitectura)
- [Base de datos (Airtable)](#️-base-de-datos-airtable)
- [Ciclo de vida del estado](#-ciclo-de-vida-del-estado-de-una-vacante)
- [Cómo ejecutarlo](#️-cómo-ejecutarlo)
- [Resiliencia y control de bucles](#️-resiliencia-manejo-de-errores-y-control-de-bucles)
- [Detalle técnico del HITL](#-detalle-técnico-del-hitl)
- [Enlaces de la entrega](#-enlaces-de-la-entrega)
- [Cumplimiento de la consigna](#-cumplimiento-de-la-consigna)

---

## 🎯 Caso de uso

Búsqueda laboral automatizada. En vez de revisar portales manualmente, el sistema:

1. Lee mi CV desde Airtable (tabla `Perfil`).
2. Consulta **dos APIs de empleo en paralelo**: Adzuna (UK) y Remotive (remoto LATAM).
3. Unifica ambas fuentes en un formato común.
4. Puntúa cada vacante de 0 a 100 con **GPT-4o-mini** según el ajuste con mi CV.
5. Guarda cada vacante en Airtable (tabla `Vacantes`) con estado `Analizado IA`.
6. Envía un email con el **Top 10** y botones **Aprobar** / **Rechazar**.
7. El flujo **se detiene y espera** mi decisión (HITL).
8. Si apruebo → **actualiza el estado a `Aprobado`** y envía el digest final. Si rechazo → registra el rechazo y no envía nada.

---

## 🏗️ Arquitectura

| Capa | Tecnología | Rol |
|------|-----------|-----|
| **Orquestador** | n8n (Cloud) | Flujo principal |
| **Base de datos / Memoria** | Airtable | Tablas `Perfil` y `Vacantes` **relacionadas** |
| **Procesamiento IA** | OpenAI GPT-4o-mini | Scoring CV↔vacante con prompt estructurado |
| **Fuente de datos 1** | Adzuna API | Empleos UK (free tier) |
| **Fuente de datos 2** | Remotive API | Empleos remotos LATAM (free, sin API key) |
| **Canal de salida** | Gmail | Email HITL de aprobación + digest final |

📄 Diagrama completo en [`Diagrama_Arquitectura_JobMatcher.pdf`](./Diagrama_Arquitectura_JobMatcher.pdf).

### Patrón de doble fuente

Cada API nombra sus campos distinto, así que cada rama tiene su propio **Split Out** y ambas convergen en un **Merge** (Append). Luego un nodo **Normalizar** las lleva a un formato común, y la IA puntúa todo por igual.

```
                  Code Términos → Adzuna → Split (results) ─┐
Set Contexto CV ──┤                                         ├→ Merge → Normalizar → IA → Airtable → HITL
                  └──────────────→ Remotive → Split (jobs) ─┘
```

---

## 🗃️ Base de datos (Airtable)

### Tabla `Perfil` (1 fila — el CV)
| Campo | Tipo | Notas |
|-------|------|-------|
| `Nombre` | Single line text | |
| `CV_Texto` | Long text | CV en texto plano |
| `Keywords` | Single line text | Términos de búsqueda separados por coma |
| `Activo` | Checkbox | El flujo solo procesa perfiles con `Activo = TRUE` |

### Tabla `Vacantes` (se llena sola)
| Campo | Tipo | Valores |
|-------|------|---------|
| `Titulo` | Single line text | |
| `Empresa` | Single line text | |
| `Ubicacion` | Single line text | |
| `URL` | URL | |
| `Score` | Number | 0–100 (puntaje IA) |
| `Justificacion` | Long text | Explicación del match |
| `Fecha` | Single line text | |
| `Estado` | Single select | `Analizado IA` · `Aprobado` · `Error` |
| **`Perfil`** | **Link to another record** | **🔗 Relación con la tabla `Perfil`** |

> 🔗 **Relación entre tablas:** el campo `Perfil` en `Vacantes` es un *linked record* que vincula cada vacante con el perfil que la originó, evitando datos aislados y dando estructura relacional al sistema.

---

## 🔄 Ciclo de vida del estado de una vacante

El campo `Estado` refleja en qué punto del proceso está cada vacante:

```
[nueva] → Analizado IA → (HITL: aprobado) → Aprobado
                       → (HITL: rechazado) → (queda en Analizado IA + log de rechazo)
[fallo de API]        → Error
```

- **`Analizado IA`**: la IA la puntuó y se guardó en la base.
- **`Aprobado`**: pasó la validación humana (HITL). El nodo `Airtable - Marcar Aprobado` actualiza este estado **después** de que el usuario aprueba.
- **`Error`**: un fallo de API (Adzuna, Remotive u OpenAI) se registró vía `Airtable - Log Error`.

---

## ⚙️ Cómo ejecutarlo

### 1. Importar el flujo
En n8n: **Workflows → Import from File →** subir `Entrega_Final_-_Vacantes_Laborales_-_VT.json`.

### 2. Variables (`Settings → Variables`)
| Variable | Valor |
|----------|-------|
| `AIRTABLE_BASE_ID` | ID de la base (empieza con `app...`) |
| `ADZUNA_APP_ID` | de developer.adzuna.com |
| `ADZUNA_APP_KEY` | de developer.adzuna.com |
| `MI_EMAIL` | correo de destino |

> Remotive **no requiere API key** (API pública gratuita).

### 3. Credenciales
- **Airtable**: Personal Access Token con scopes `data.records:read`, `data.records:write`, `schema.bases:read`.
- **OpenAI**: API key con saldo.
- **Gmail**: OAuth2.

Ninguna clave está hardcodeada — todo va por el credential manager y `$vars`.

### 4. Activar
El HITL (Wait con resume por webhook) requiere que n8n sea accesible públicamente. **n8n Cloud ya lo cumple.** Activá el workflow con el toggle **Active**.

---

## 🛡️ Resiliencia, manejo de errores y control de bucles

- **Filtro de entrada (anti-bucle):** el nodo `IF - CV cargado?` valida al inicio que exista un CV **activo** antes de continuar. Si no lo hay, el flujo corta y notifica, evitando ejecuciones vacías o en loop.
- **Trigger controlado:** Schedule cada 3 días (no un trigger continuo), evitando consumo masivo de operaciones.
- **Camino Infeliz (sin CV):** deriva a un email de alerta y detiene el flujo.
- **Fallo de API:** los nodos Adzuna, Remotive y OpenAI usan `onError: continueErrorOutput` → registran el fallo en `Airtable - Log Error` sin romper el lote.
- **Tipos de datos:** el score se castea con `parseInt(...) || 0` antes de comparar.
- **Optimización de costo:** el prompt limita `Max Tokens` en el nodo de IA y recorta la descripción de cada vacante a 1500 caracteres, manteniendo bajo el consumo por llamada aun procesando todas las vacantes del ciclo.

---

## 🔑 Detalle técnico del HITL

El email de aprobación usa la URL de reanudación única de cada ejecución:

```javascript
const resumeUrl = $execution.resumeUrl;                  // ya incluye ?signature=xxx
const urlAprobar  = `${resumeUrl}&decision=aprobar`;     // & (no ?) para no romper la firma
const urlRechazar = `${resumeUrl}&decision=rechazar`;
```

> ⚠️ `$execution.resumeUrl` **ya trae** un `?signature=`. Por eso los parámetros propios se concatenan con `&`. Usar `?` genera doble `?`, corrompe la firma y devuelve `Invalid token`.

El nodo `Wait` está en **Resume: On Webhook Call** con **Authentication: None**. Al reanudar, `IF - Aprobado?` lee `$json.query.decision`:
- **Rama TRUE** → actualiza estado a `Aprobado` en Airtable + envía digest final.
- **Rama FALSE** → registra el rechazo.

---

## 🔗 Enlaces de la entrega

- 🎥 **Video demo (3 min):** _https://drive.google.com/file/d/1yRQ7EvfyTYlWxvCxdsf1ykBlFSlG8DwD/view?usp=drive_link_
- 🗃️ **Base Airtable (modo lectura):** https://airtable.com/invite/l?inviteId=invwATxW1HGnVjXu1&inviteToken=cb996789134ed5e3860da54ad09ff2aac006273fb2ad92ef6bb65324559a3b71&utm_medium=email&utm_source=product_team&utm_content=transactional-alerts
- 📂 **Flujo:** [`Entrega Final - Vacantes Laborales - VT.json`](./Entrega%20Final%20-%20Vacantes%20Laborales%20-%20VT.json)
- 📄 **Diagrama:** [`Diagrama_Arquitectura_JobMatcher.pdf`](./Diagrama_Arquitectura_JobMatcher.pdf)
- 🖼️ **Evidencias:** [Entrega Final - Job Matched IA.pdf](./Entrega%20Final%20-%20Job%20Matched%20IA.pdf)
---

## ✅ Cumplimiento de la consigna

- [x] Orquestador en n8n
- [x] Base de datos (Airtable) con campos de estado **y relación entre tablas** (linked record)
- [x] Procesamiento IA (GPT-4o-mini) con prompt dinámico estructurado
- [x] Canal de salida (Gmail)
- [x] Trigger inteligente (Schedule)
- [x] Rutas de error (Log Error en Airtable)
- [x] Human-in-the-Loop (Wait + email de aprobación end-to-end)
- [x] **Cierre de estado:** las vacantes pasan a `Aprobado` tras el HITL
- [x] **Filtro anti-bucle** en la entrada del flujo
- [x] Nodos nombrados, variables dinámicas, cero hardcode
- [x] Bonus: integración de dos fuentes de datos (Adzuna + Remotive)
