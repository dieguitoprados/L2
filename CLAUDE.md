# CFA L2 Practice App — Especificación para Claude Code

Este documento recoge las decisiones de diseño ya tomadas. Léelo entero antes de escribir código. Si algo aquí entra en conflicto con lo que encuentres en el repo, pregunta antes de cambiarlo.

## 1. Contexto

- **Usuario principal:** candidato a CFA Level II (examen: 19 nov 2026), con poco tiempo. La herramienta tiene que ser usable en pocas semanas, no perfecta.
- **Usuarios secundarios:** un grupo pequeño y cerrado de colegas, también candidatos, invitados por email.
- **Objetivo:** practicar con los *practice problems* del curriculum oficial de CFA Institute en formato de examen (item sets), con sesiones configurables, estadísticas por sesión y por Learning Module, repetición espaciada de fallos y análisis de errores.
- **Fuente de contenido:** PDFs del curriculum 2026 Level II (un volumen por topic). Disponibles ahora: Ethics, FSA, Equity Valuation, Fixed Income y Portfolio Management. Faltan Quant, Economics, Corporate Issuers, Derivatives y Alternatives, que se añadirán después sin cambiar el esquema.

### Formato del examen L2 (condiciona todo el diseño)

- Unidad = **item set**: una vignette con exhibits seguida de varias preguntas (normalmente 4 en el examen; en el curriculum, de 4 a 7).
- Opciones A/B/C (3 opciones).
- Ritmo objetivo: **3 minutos por pregunta**.

### Convenciones

- Código, identificadores, nombres de tablas y comentarios en inglés.
- UI en español. La terminología técnica (LOS, FVOCI, item set…) se mantiene en inglés.
- **El contenido del curriculum nunca entra en git.**
  - `data/` está en `.gitignore`.
  - El banco de preguntas vive solo en la base de datos.
  - No pongas preguntas reales en tests ni fixtures: usa contenido sintético.

## 2. Stack

| Capa | Elección |
|---|---|
| UI | Streamlit (multipage con `st.navigation` o `pages/`) |
| DB | PostgreSQL en Supabase o Neon en producción; SQLite en desarrollo local. El mismo código vía SQLAlchemy 2.x con la URL desde `st.secrets` o variable de entorno |
| Migraciones | Alembic |
| Parsing PDF | PyMuPDF (`fitz`), incluido `page.find_tables()` para los exhibits |
| Tests | pytest |
| Deploy | Streamlit Community Cloud, app privada desde repo privado |

Restricciones de Streamlit Community Cloud:
- El filesystem es efímero, así que nada persistente en disco.
- La cuenta permite una sola app privada.
- Los viewers invitados pueden invitar a otros, por eso existe la allowlist propia de la sección 7.

## 3. Estructura del repo

```
cfa_app/
├── CLAUDE.md
├── README.md
├── requirements.txt
├── .gitignore                  # data/, .streamlit/secrets.toml, *.db, __pycache__
├── .streamlit/
│   └── secrets.toml.example    # plantilla sin valores reales
├── alembic/                    # migraciones
├── data/                       # IGNORADO POR GIT
│   ├── raw/                    # PDFs
│   ├── staging/                # JSON extraído, pendiente de revisión
│   └── assets/                 # recortes de exhibits (PNG) antes de cargarlos
├── ingest/                     # se ejecuta en LOCAL, nunca en la app desplegada
│   ├── extract.py              # PDF → bloques de texto limpios + tablas/recortes
│   ├── parse_curriculum.py     # bloques → item sets / preguntas / soluciones
│   ├── classify.py             # heurística question_type
│   ├── validate.py             # reglas de la sección 5.4
│   ├── load.py                 # staging JSON → DB (idempotente, upsert por id)
│   └── cli.py                  # python -m ingest.cli FSA data/raw/fsa.pdf
├── core/                       # lógica pura, sin imports de streamlit
│   ├── db.py                   # engine, session factory
│   ├── models.py               # SQLAlchemy ORM
│   ├── config.py               # SessionConfig (dataclass/pydantic), topic weights
│   ├── selector.py             # selección de item sets según config
│   ├── engine.py               # máquina de estados de la sesión
│   ├── srs.py                  # Leitner
│   ├── stats.py                # agregaciones
│   └── export.py               # error log markdown, CSV
├── app/
│   ├── main.py                 # entrypoint, auth gate, navegación
│   ├── auth.py                 # require_user(), is_admin()
│   ├── components/             # render de vignette, exhibits, pregunta, timer
│   └── pages/
│       ├── home.py             # dashboard resumen
│       ├── new_session.py      # configuración
│       ├── session.py          # runner
│       ├── review.py           # revisión post-sesión + error tags
│       ├── stats.py
│       ├── srs_review.py       # repaso del día
│       └── editor.py           # SOLO ADMIN: revisar/corregir contenido
└── tests/
    ├── fixtures/               # PDFs/textos SINTÉTICOS con el mismo formato
    ├── test_parser.py
    ├── test_validate.py
    ├── test_engine.py
    ├── test_srs.py
    └── test_stats.py
```

## 4. Modelo de datos

IDs de contenido estables y legibles para que la carga sea idempotente:
- `topic_id`: `FSA`, `EQ`, `FI`, `PM`, `ETH`, `QM`, `ECON`, `CI`, `DER`, `ALT`
- `module_id`: `FSA-LM01`
- `item_set_id`: `FSA-LM01-IS01`
- `question_id`: `FSA-LM01-Q03`. El número de pregunta del libro se conserva.

```
users
  id (pk), email (unique), display_name, is_admin, exam_date (default 2026-11-19), created_at

topics
  id (pk, ej. 'FSA'), name, weight_min, weight_max     -- pesos del examen, ver 9.4

modules
  id (pk, ej. 'FSA-LM01'), topic_id (fk), number, title, los_text (JSON list)

item_sets
  id (pk), module_id (fk), ordinal, q_from, q_to,
  vignette_md (text), source_ref (libro + página), edition ('2026'),
  review_status ('pending'|'ok'|'flagged'), notes

exhibits
  id (pk), item_set_id (fk, nullable), question_id (fk, nullable),  -- exhibits también en soluciones
  label ('Exhibit 1'), title, kind ('table_md'|'image'), content_md (nullable),
  image (bytea / blob, nullable), page, bbox (JSON)

questions
  id (pk), item_set_id (fk, nullable para standalone), module_id (fk), number,
  stem_md, options (JSON: {"A": "...", "B": "...", "C": "..."}),
  correct ('A'|'B'|'C'), explanation_md,
  question_type ('conceptual'|'calculation'|'multi_step'), type_source ('heuristic'|'manual'),
  review_status

sessions
  id (pk), user_id (fk), mode, config (JSON), started_at, ended_at (nullable),
  deadline_at (nullable), status ('active'|'completed'|'abandoned')

session_items                -- orden fijado al crear la sesión (permite reanudar)
  session_id, position, item_set_id

attempts
  id (pk), session_id (fk), user_id (fk), question_id (fk),
  chosen ('A'|'B'|'C'|null si timeout), is_correct, confidence ('sure'|'unsure'|'guess'|null),
  shown_at, answered_at, time_s, error_tag (nullable), error_note (nullable)

srs_state
  user_id, question_id (pk compuesta), box (1-5), due_date, last_reviewed_at
```

Reglas:
- Todas las tablas de actividad (`sessions`, `attempts`, `srs_state`) llevan `user_id`. Cada usuario solo ve sus datos; nunca hay consultas sin filtrar por `user_id`, salvo las agregadas anónimas de la sección 9.3.
- El contenido (`item_sets`, `questions`, `exhibits`) es compartido y solo lo editan los admins.

## 5. Pipeline de ingesta (local)

Se ejecuta en la máquina del usuario con los PDFs en `data/raw/`. Nunca en Streamlit Cloud. Escribe en la DB de producción mediante `DATABASE_URL` en el entorno local.

### 5.1 Estructura observada en los PDFs (volumen FSA, extracción de prueba)

Confirma estos patrones con PyMuPDF antes de fijar las regex; la extracción de prueba se hizo con otra herramienta.

- Cada Learning Module termina con dos secciones:
  1. `PRACTICE PROBLEMS`.
  2. `SOLUTIONS`.
- Después del último módulo viene `Glossary`: parar ahí.
- Inicio de item set: `The following information relates to questions` seguido de un rango `1-6`. **El rango puede estar en la línea siguiente.** Tras él viene la vignette, que puede contener `Exhibit N: título` + tabla.
- La numeración de exhibits **se reinicia en cada item set**. Cada vignette tiene su propio "Exhibit 1".
- Las preguntas tienen la forma `N. enunciado` seguido de `A. …`, `B. …`, `C. …`. Las opciones pueden ocupar varias líneas.
- Soluciones: `N. X is correct.` + explicación. Algunas explicaciones contienen tablas o cálculos en varias líneas.
- Ruido a limpiar:
  - **Cabeceras y pies de página** intercalados en mitad de preguntas, por ejemplo `Practice Problems 47`, `46 Learning Module 1 Intercorporate Investments` o `Solutions 59`. Un enunciado puede partirse por un salto de página justo ahí.
  - **Guiones de partición**: el carácter `\u0002` (`pro\u0002fessional`). Elimínalo uniendo la palabra.
  - Finales de línea `\r\n`.
- Las tablas se aplanan al extraer el texto (las columnas se pierden). Por eso los exhibits necesitan tratamiento propio (5.2).
- Las LOS aparecen al inicio de cada módulo bajo `LEARNING OUTCOMES`. Guárdalas en `modules.los_text`. Las preguntas se asocian al **módulo**, no a una LOS individual.

### 5.2 Exhibits

1. Localiza en la página la región de cada `Exhibit N:`, desde el título hasta el siguiente bloque de texto que no sea tabla.
2. Intenta `page.find_tables()` en esa región y convierte el resultado a tabla markdown (`kind='table_md'`).
3. Si falla, o la tabla resultante tiene celdas vacías o desalineadas, recorta la región como PNG a 2x de resolución (`kind='image'`).
4. Marca siempre el item set como `review_status='pending'`: los exhibits se revisan a mano en el editor.

Lo mismo aplica a las tablas dentro de las soluciones.

### 5.3 Clasificación `question_type` (heurística, editable)

- `calculation`: la explicación contiene operaciones numéricas (`=`, `×`, `x`, `/` entre números, `%` calculados) o verbos como *calculate*, *compute*.
- `multi_step`: es `calculation` y la explicación tiene ≥3 líneas de cálculo o referencia a varios exhibits.
- `conceptual`: el resto.

Se guarda con `type_source='heuristic'`. El editor permite cambiarlo a `manual`.

### 5.4 Validación (bloqueante: si falla, no se carga ese módulo)

- La numeración de preguntas es contigua `1..N` dentro del módulo.
- Los rangos de item sets cubren todas las preguntas sin huecos ni solapes. Puede haber preguntas standalone fuera de rangos; se reportan aparte.
- Cada pregunta tiene exactamente las opciones A, B y C, no vacías.
- Cada pregunta tiene una solución con `correct ∈ {A,B,C}` y una explicación no vacía.
- Cada referencia a `Exhibit N` en un enunciado tiene su exhibit capturado en ese item set.
- No quedan restos de cabeceras de página ni `\u0002` en ningún campo.

`validate.py` imprime un informe por módulo: nº de item sets, nº de preguntas, exhibits en tabla frente a imagen, y errores.

**Prueba de aceptación del parser:** el LM1 de FSA (Intercorporate Investments) debe dar **34 preguntas en 6 item sets** (rangos 1-6, 7-11, 12-16, 17-21, 22-28, 29-34).

### 5.5 Carga

- `load.py` hace upsert por ID.
- Recargar un módulo actualiza el contenido, pero **no toca** los items con `review_status='ok'` salvo con `--force`. Así las correcciones manuales no se pierden.
- Registra la edición (`'2026'`) y `source_ref` (volumen + página).

## 6. App: comportamiento

### 6.1 Configuración de sesión (`SessionConfig`)

```
mode:            'practice' | 'exam_sim' | 'srs' | 'mistakes'
scope:           topics[] y/o modules[]              (vacío = todo)
n_item_sets:     int                                  (la unidad es el item set, no la pregunta)
timing:          'none' | 'per_question' (3 min por defecto, editable) | 'total' (minutos)
feedback:        'immediate' | 'per_item_set' | 'end'
filters:         unseen_only, failed_before, guessed_before, slow_before (> objetivo),
                 question_type[]
confidence:      bool (default true)
shuffle_sets:    bool (default true)       -- NUNCA barajar opciones ni preguntas dentro de un set
```

Presets por modo:
- `exam_sim`: 3 min por pregunta con deadline total, `feedback='end'`, mezcla por pesos de topic (9.4), sin filtros. Si faltan topics en la DB, avisa en pantalla de que la mezcla es parcial.
- `srs`: sirve solo las preguntas con `due_date <= hoy`, agrupadas por su item set (se muestra la vignette completa, pero solo se puntúan las preguntas vencidas).
- `mistakes`: preguntas falladas o adivinadas, con su vignette.

### 6.2 Motor de sesión (`core/engine.py`)

- Al crear la sesión se fija el orden en `session_items`. Esto permite **reanudar** una sesión `active` tras cerrar la pestaña.
- **Cada respuesta se persiste en `attempts` en el momento del envío**, no al final de la sesión.
- Tiempos medidos con timestamps del servidor (`shown_at`, `answered_at`). El timer visual es informativo.
- Deadline con `timing='total'` o `exam_sim`:
  - Se guarda `deadline_at`.
  - En cada rerun, si `now > deadline_at`, la sesión se cierra y las preguntas no respondidas cuentan como fallo con `chosen=null`.
  - Para el countdown visual vale `streamlit-autorefresh` (cada 1-5 s) o un componente JS mínimo. No dependas del refresco para la lógica.
- Todo el estado de UI va en `st.session_state`. El engine es puro y testeable: recibe el estado y un evento, y devuelve el nuevo estado.

### 6.3 Pantalla de pregunta

- La vignette y sus exhibits quedan visibles mientras se responden las preguntas de ese set (layout de dos columnas en escritorio y apilado en móvil, con la vignette plegable).
- Con `confidence=true`, el botón "Enviar" se sustituye por tres botones: **Seguro / Dudoso / Adivino**. Un clic registra la respuesta y la confianza a la vez. La confianza se captura **siempre en el momento**, nunca al final.
- Con `feedback='immediate'`: tras enviar, se muestra correcto/incorrecto, la opción correcta y la explicación.
- Con `per_item_set`: igual, pero al terminar todas las preguntas del set.

### 6.4 Revisión post-sesión (`review.py`)

- Resumen:
  - Acierto total.
  - Acierto por módulo.
  - Tiempo medio frente al objetivo.
  - Acierto por nivel de confianza.
- Lista de preguntas con la respuesta dada, la correcta y la explicación.
- **Etiquetado de errores aquí, al final, solo para fallos** (y opcional para aciertos con confianza `guess`). Etiquetas:
  - `concept` — no conocía o no entendía el concepto.
  - `formula` — fórmula olvidada o mal recordada.
  - `calculation` — error de cálculo o de calculadora.
  - `vignette_reading` — dato mal leído o no encontrado en la vignette o el exhibit.
  - `question_trap` — mala lectura del enunciado (*most likely*, *least likely*, EXCEPT…).
  - `time` — respondido con prisa o sin tiempo.
- Una etiqueta por fallo y una nota libre opcional.
- Etiquetar con un clic por fila; no obligatorio.

### 6.5 SRS (Leitner, `core/srs.py`)

- Cajas 1-5 con intervalos de 1, 3, 7, 14 y 30 días.
- Transiciones tras cada intento:
  - Fallo, o acierto con `guess` → caja 1.
  - Acierto con `unsure` → misma caja, `due_date` recalculada.
  - Acierto con `sure` → caja +1 (máximo 5).
  - Con `confidence` desactivada, acierto = `sure`.
- Una pregunta entra en SRS en su primer fallo o acierto adivinado.
- **Tope por fecha de examen:** `due_date` nunca posterior a `users.exam_date - 3 días`. Si el intervalo la sobrepasa, se fija en ese tope.

## 7. Autenticación y acceso

- Despliegue como app privada en Streamlit Community Cloud. Los usuarios entran con el email invitado.
- Allowlist propia en secrets, comprobada en `app/auth.py::require_user()` al inicio de **todas** las páginas:
  ```toml
  [auth]
  allowed_emails = ["...", "..."]
  admin_emails = ["..."]
  ```
- El email del usuario logueado se obtiene de `st.user` (verifica la API exacta en la doc de la versión de Streamlit instalada). Si el email no está en la lista, muestra un mensaje y llama a `st.stop()`.
- El primer acceso de un email permitido crea su fila en `users`.
- Desarrollo local: si `DEV_USER_EMAIL` está definido y no hay `st.user` disponible, usarlo.
- `editor.py` solo para `is_admin`.

## 8. Despliegue

1. Repo privado en GitHub.
2. Proyecto en Supabase (o Neon) → `DATABASE_URL`. Usa el connection string con pooler y `pool_pre_ping=True` en el engine, cacheado con `st.cache_resource`.
3. `alembic upgrade head` contra esa DB desde local.
4. Ingesta desde local contra esa DB.
5. Streamlit Community Cloud: New app → repo privado → `app/main.py` → pegar los secrets (`DATABASE_URL`, `[auth]`).
6. Configurar la app como privada e invitar los emails de la allowlist.

El README debe documentar estos pasos. Los pasos 2, 5 y 6 los hace el usuario a mano.

## 9. Estadísticas (`core/stats.py`)

### 9.1 Por usuario

Todas las métricas se pueden filtrar por rango de fechas y topic.

- Acierto por sesión, topic y módulo, con nº de intentos (no mostrar porcentajes con n<5 sin indicarlo).
- Tendencia temporal del acierto: media móvil por semana.
- Tiempo medio y mediano por pregunta frente a 3 min, por topic y por `question_type`.
- **Calibración:** acierto por nivel de confianza. Ejemplo de señal: acertar el 60% de lo marcado como "seguro" indica sobreconfianza.
- **Aciertos frágiles:** % de aciertos marcados `guess`.
- Distribución de `error_tag` por topic.
- Cobertura: preguntas vistas sobre disponibles, por módulo.
- **Vista de riesgo:** (1 − acierto del topic) × peso medio del topic. Indica dónde se pierden más puntos esperados, no solo dónde se falla más.

### 9.2 Dificultad empírica

La "dificultad" no viene en el libro; se construye.
- Por usuario: acierto y tiempo medio en esa pregunta.
- Filtro "difícil" de la configuración = preguntas que el usuario falló, adivinó o en las que tardó más del objetivo.

### 9.3 Agregado anónimo entre usuarios (fase 2)

Acierto global por pregunta con n≥3 usuarios, solo como dato orientativo. Nunca exponer datos individuales de otros usuarios.

### 9.4 Pesos de topic (en `core/config.py`)

Rangos L2 a usar como valor inicial; verifica contra la web de CFA Institute antes de fijarlos.

| Topic | Peso |
|---|---|
| ETH | 10-15% |
| QM | 5-10% |
| ECON | 5-10% |
| FSA | 10-15% |
| CI | 5-10% |
| EQ | 10-15% |
| FI | 10-15% |
| DER | 5-10% |
| ALT | 5-10% |
| PM | 10-15% |

`exam_sim` usa el punto medio, normalizado sobre los topics disponibles en la DB.

## 10. Exportaciones (`core/export.py`)

- **Error log en markdown:** fallos con fecha, módulo, enunciado resumido (primeras ~20 palabras), respuesta dada, correcta, `error_tag` y nota. Pensado para subirlo al Claude Project y analizar patrones de error.
- **CSV de acierto por módulo:** columnas `topic`, `module_id`, `title`, `attempts`, `accuracy`, `avg_time_s`. Pensado para cruzarlo con el tracker de Excel del usuario.

## 11. Plan de trabajo (en este orden; no avanzar sin cumplir los criterios)

### Hito 1 — Parser (local)

- `extract.py`, `parse_curriculum.py` y `validate.py` sobre el volumen de FSA.
- **Criterio:** el LM1 da 34 preguntas y 6 item sets con 0 errores de validación, y todos los módulos de FSA validan o reportan errores concretos y accionables.
- Tests con fixtures sintéticos que reproduzcan los patrones de 5.1: rango en línea siguiente, cabecera de página a mitad de enunciado, `\u0002` y opción en varias líneas.

### Hito 2 — Esquema + carga

- Modelos, Alembic, `load.py` idempotente y SQLite local.
- **Criterio:** cargar FSA dos veces no duplica nada; una corrección `review_status='ok'` sobrevive a una recarga.

### Hito 3 — Modo práctica

- Pantallas `new_session`, `session` y `review` con `mode='practice'`, los tres modos de feedback, confianza con tres botones, timing `none`/`per_question`, persistencia por intento, reanudación y error tags.
- **Criterio:** una sesión de 3 item sets de punta a punta, interrumpida a mitad y reanudada, sin perder intentos.

### Hito 4 — Despliegue multiusuario

- Postgres, auth y allowlist, editor de admin, README con los pasos de despliegue.
- **Criterio:** dos emails distintos ven solo sus propias estadísticas; un email fuera de la lista no pasa de la pantalla de acceso.

### Hito 5 — Resto

- Estadísticas completas (sección 9), SRS, `exam_sim`, `mistakes`, exportaciones e ingesta de los demás volúmenes.
- Modo "worked example" para los Examples y Knowledge Checks del texto: enunciado → revelar solución → autoevaluación (bien / parcial / mal), sin opciones inventadas.

## 12. Fuera de alcance

- Generar preguntas o distractores con LLM.
- Barajar opciones.
- Gamificación.
- Notificaciones.
- App móvil nativa.
- Cualquier despliegue público.
- Contenido de Kaplan Schweser (posible fase posterior con su propio parser; el esquema ya lo admite vía `source_ref`).

## 13. Nota sobre el contenido

El curriculum es material con copyright de CFA Institute. Su propia página de copyright prohíbe la reproducción en sistemas de almacenamiento o recuperación de información.

El acceso a la app debe limitarse a la allowlist de candidatos. El contenido no puede:
- subirse a git;
- exponerse en ningún endpoint público;
- incluirse en logs, mensajes de error ni tests.
