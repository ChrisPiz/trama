# Trama — detalles técnicos

Documentación profunda para usuarios que quieren entender qué hace Trama por dentro, cómo correrlo en cron, qué archivos crea, y cómo se comporta en distintos entornos.

← Volver al [README](README.md).

---

## Funcionalidades completas

| Capacidad | Detalle |
|---|---|
| 🔍 **Búsqueda con citas** | Todas las menciones de un personaje, lugar u objeto, con texto verbatim + capítulo + línea. FTS5 sub-ms incluso en 500k palabras. |
| 📖 **Character bible automatizado** | Extrae personajes, lugares y objetos del texto vía 4 señales (capitalización + dialog tags + relaciones familiares + sujeto de verbo). Clasifica como `character` / `place` / `object` / `unknown`. Genera scaffold markdown editable. |
| ⚠️ **Auditoría de atributos** | Detecta contradicciones explícitas: edad, ojos, pelo, altura, profesión, relaciones. Atribuye claims al dueño correcto (filtra "ojos de Elena" cuando Marta también está en el contexto). Excluye flashbacks ("cuando tenía 12 años") del audit principal. Severidad: `hard` / `soft` / `drift` / `ok`. |
| ⏰ **Línea temporal** | Extrae 8 tipos de marcador (fechas absolutas, saltos relativos, días de la semana, estaciones, edades, próximos días, hedge temporal) en ES/EN. Cruza con audit de atributos para validar coherencia. |
| 🧵 **Hilos narrativos** | Detecta preguntas abiertas, promesas (`prometió`, `juró`, `voy a`), personajes huérfanos (freq baja con dialog tag), objetos introducidos con énfasis y nunca usados. Marca `resolved`/`no` con confianza heurística. |
| 🔁 **Auditoría recurrente** | Cada run crea snapshot timestamped en `runs/<TS>/`. Symlink `current/` apunta al último. `audit-diff.sh` compara dos runs y reporta entidades nuevas/desaparecidas, hilos cerrados o persistentes. Append-only `audit-log.tsv` para tendencias. |
| 🧠 **Subagentes paralelos** | Para sagas (>150k palabras o multi-volumen), orquesta agentes paralelos por arco/entidad/dimensión. Agrega TSVs y genera reporte final. |
| 📁 **Multi-formato** | `.txt`, `.md`, `.docx` (pandoc o python-docx), `.rtf` (pandoc o textutil), o carpetas con varios archivos en orden alfabético. |
| 🌐 **Bilingüe ES/EN** | Detección automática de idioma. Patrones regex separados para capítulos, marcadores temporales, dialog tags, relaciones, atributos. |
| 🗺️ **Mapeo línea → capítulo** | Lookup O(log n) vía `chapters.tsv`. Toda cita lleva capítulo + línea. |
| 💾 **Workspace persistente** | `<dir-del-manuscrito>/trama-doc/<nombre>/` por defecto. Override centralizado con `TRAMA_HOME=/ruta`. |
| 🔍 **FTS5 con acentos** | SQLite FTS5 con `tokenize='unicode61 remove_diacritics 2'` — `anos` encuentra `años`. Operadores `AND`/`OR`/`NEAR(A B, N)`. |
| 🔄 **Auto-update** | Detecta commits nuevos en GitHub al iniciar una auditoría (máx 1x/día). Avisa con el comando para actualizar. |

---

## Arquitectura

```
trama/
├── SKILL.md                          # router + first contact
├── references/                       # módulos cargados bajo demanda
│   ├── prepare.md                    # workspace + conversión
│   ├── index.md                      # SQLite FTS5 + caches
│   ├── entities.md                   # extracción personajes/lugares
│   ├── timeline.md                   # marcadores temporales
│   ├── threads.md                    # hilos sin resolver
│   ├── consistency.md                # auditoría contradicciones
│   ├── query.md                      # respuestas con citas
│   ├── parallel.md                   # subagentes para sagas
│   ├── recurrence.md                 # snapshots + diff
│   └── patterns-bilingual.md         # regex ES/EN
├── scripts/                          # trabajo determinista
│   ├── prepare.sh                    # pipeline completo entrada→workspace
│   ├── index.sh                      # FTS5 + chapters.tsv + wordcount
│   ├── chapter-of-line.sh            # línea N → capítulo
│   ├── extract-entities.sh           # 4 señales clasificación
│   ├── extract-timeline.sh           # marcadores cronológicos
│   ├── extract-threads.sh            # promesas + preguntas + huérfanos
│   ├── fts-query.sh                  # FTS5 wrapper
│   ├── audit-attribute.sh            # cross-check atributos entidad
│   ├── audit-run.sh                  # orquestador completo
│   ├── audit-diff.sh                 # compara runs
│   └── check-update.sh               # auto-update detector
└── templates/
    ├── bible.md                      # scaffold character bible
    └── audit-report.md               # formato reporte final
```

---

## Workspace

### Modo por defecto: junto al manuscrito (visible)

Si auditás `/Users/yo/Documents/novela.docx`, Trama crea:

```
/Users/yo/Documents/
├── novela.docx              ← tu manuscrito
└── trama-doc/
    └── novela/              ← workspace, todo visible aquí
        ├── manuscript.txt   # versión normalizada
        ├── meta.json        # hash, idioma, word count
        ├── chapters.tsv     # línea<TAB>título
        ├── wordcount.txt
        ├── fts5.db          # índice búsqueda
        ├── entities.tsv     # personajes/lugares/objetos
        ├── timeline.tsv     # marcadores temporales
        ├── threads.tsv      # hilos sin resolver
        ├── current → runs/<TS>/
        ├── runs/
        │   ├── 2026-05-03T16-30-00Z/
        │   │   ├── meta.json
        │   │   ├── entities.tsv
        │   │   ├── timeline.tsv
        │   │   ├── threads.tsv
        │   │   ├── audit-summary.txt
        │   │   └── diff-from-<TS>.md
        │   └── ...
        └── audit-log.tsv    # append-only: tendencias
```

Para sagas (carpeta con varios archivos), Trama crea `trama-doc/<nombre-de-carpeta>/` en el directorio padre.

### Modo centralizado opcional

Si preferís todos los workspaces en un solo lugar (útil para sync iCloud, o si auditás libros desde directorios read-only):

```bash
export TRAMA_HOME=~/.trama
```

Los workspaces se crean en `$TRAMA_HOME/<hash>/` indexados por SHA-1 de la ruta.

---

## Comandos manuales (avanzado)

Normalmente Claude Code orquesta todo en lenguaje natural. Si querés correr scripts a mano:

```bash
# Run completo + snapshot + log
bash ~/.claude/skills/trama/scripts/audit-run.sh /ruta/manuscrito.docx
bash ~/.claude/skills/trama/scripts/audit-run.sh /ruta/manuscrito.docx --all          # incluye audit cross-entity
bash ~/.claude/skills/trama/scripts/audit-run.sh /ruta/manuscrito.docx --note "v3 pre-beta"

# Diff vs run anterior (auto: último vs penúltimo)
bash ~/.claude/skills/trama/scripts/audit-diff.sh "$WORK"

# Tendencias
column -t -s$'\t' < "$WORK/audit-log.tsv"
```

Donde `$WORK` es la carpeta `trama-doc/<nombre>/`.

### Cron diario

Si escribís activamente, podés programar auditorías nocturnas:

```bash
crontab -e
```

Agregá:
```
0 23 * * * /bin/bash $HOME/.claude/skills/trama/scripts/audit-run.sh /ruta/novela.docx --note "auto-nightly"
```

Por la mañana corrés `audit-diff.sh` y ves qué cambió mientras dormías.

Alternativa sin cron: el skill `/schedule` del harness de Claude Code.

---

## Compatibilidad por entorno

| Función | Claude Code (CLI) | Claude.ai web + Desktop |
|---|---|---|
| Convertir `.docx`/`.md`/`.rtf` → texto | ✅ | ✅ (subes el archivo al chat) |
| Búsqueda con citas | ✅ | ✅ |
| Character bible automatizado | ✅ | ✅ |
| Auditoría de atributos + flashbacks | ✅ | ✅ |
| Línea temporal | ✅ | ✅ |
| Hilos narrativos | ✅ | ✅ |
| FTS5 sub-ms | ✅ Garantizado | ⚠️ Depende del sandbox; fallback automático a `grep` si falla |
| Workspace persistente | ✅ Sobrevive reboots | ❌ Se borra al cerrar la conversación |
| Auditoría recurrente | ✅ | ❌ Sin persistencia entre sesiones |
| Diff entre runs | ✅ | ❌ No hay run anterior |
| Log de tendencias | ✅ | ❌ Se pierde |
| Cron / scheduled audits | ✅ | ❌ No aplica |
| Apuntar a `/Users/yo/novela.docx` | ✅ | ❌ Subir como attachment |

Claude Desktop usa el mismo sandbox que Claude.ai web (sincroniza desde la cuenta). Mismas limitaciones.

**Conclusión:** Claude.ai/Desktop = auditor one-shot. Claude Code = compañero de escritura.

### Instalación en Claude.ai / Claude Desktop

1. Descarga [`trama.zip`](https://github.com/ChrisPiz/trama/raw/main/trama.zip)
2. Abre [claude.ai](https://claude.ai) → **Settings → Capabilities → Skills**
3. Click **Upload skill** y selecciona el ZIP
4. Sincroniza automáticamente a Claude Desktop

> Requiere plan Pro/Team/Enterprise. En plan Free solo están disponibles los skills oficiales de Anthropic.

### Otros entornos

Trama sigue el formato Anthropic Skills estándar (SKILL.md + frontmatter YAML + módulos en `references/` + scripts en `scripts/`). Compatible con:

- **Copilot CLI** (Microsoft) — soporta Anthropic Skills nativos
- **Gemini CLI** (Google) — soporta vía `activate_skill`

Cursor, Zed, VS Code: NO soportan Anthropic Skills nativos.

---

## Actualización

Trama auto-detecta updates: cuando ejecutas una auditoría, el skill chequea silenciosamente si hay commits nuevos en GitHub (máx 1 vez al día) y te avisa si hay novedades.

Cuando veas el aviso:
```bash
cd ~/.claude/skills/trama && git pull
```

Reinicia Claude Code después del pull. **Tu workspace `trama-doc/` no se toca** — solo se actualiza el código del skill.

Desactivar el check:
```bash
export TRAMA_NO_UPDATE_CHECK=1
```

Para Claude Desktop / Claude.ai (web), no hay auto-detect — re-descarga el ZIP y re-súbelo.

---

## Limitaciones honestas

- **Pronombres no resueltos:** "Ella entró" tras "Elena llegó" probablemente refiere a Elena, pero el extractor no hace coreference. Pide confirmación si es relevante.
- **Inconsistencias implícitas:** subtexto, tono, atmósfera quedan fuera. Solo detecta contradicciones explícitas con citas directas.
- **Prosa muy metafórica:** falsos positivos en marcadores temporales ("hace mil años que no te veo"). Reporta el match crudo y deja al escritor juzgar.
- **Personajes referidos solo por descripción** ("el viejo del faro") no aparecen en el extractor.
- **Apodos:** "Marta" y "Martita" cuentan como entidades distintas. Regístralos en `aliases.tsv` para fusionarlos.
- **Manuscritos sin marcadores de capítulo:** las citas usan solo número de línea.
- **Cambios deliberados** (personaje envejece entre tomos, cambia profesión): regístralos en `exceptions.tsv` para que el audit los respete.

Reportar limitaciones aumenta confianza. El skill no infla certezas.

---

## Dependencias

Solo herramientas estándar:

- `bash`, `grep`, `awk`, `sed`, `wc`, `find`, `shasum`, `iconv`, `python3`
- **Crítico:** `sqlite3` con FTS5 (preinstalado en macOS y Linux modernos)
- **Recomendado:** `pandoc` (`brew install pandoc`) — el conversor más fiable
- **Opcional:** `ripgrep` (`brew install ripgrep`) — 5–10x más rápido que `grep`
- **Fallback `.docx`:** `python-docx` (`pip install --user python-docx`)
- **Fallback `.rtf` macOS:** `textutil` (preinstalado)

El skill **nunca instala dependencias en silencio** — si falta algo, pregunta antes.

---

## Formatos soportados

| Formato | Conversor preferido | Fallback |
|---------|---------------------|----------|
| `.txt`, `.md` | `cp` directo | — |
| `.docx` | `pandoc` | `python-docx` |
| `.rtf` | `pandoc` | `textutil` (macOS) |
| Carpeta con varios | concatenación alfabética con marcadores `=== filename ===` | — |

Para Pages o Google Docs: exportar primero a Word o Markdown.

PDF no soportado — el OCR/extracción tiene demasiada pérdida para auditoría con citas exactas.

---

## Contribuir

Reportes y PRs en https://github.com/ChrisPiz/trama.

Para añadir un nuevo patrón regex (atributo, marcador temporal), edita `references/patterns-bilingual.md` (fuente única) y referencia desde el script consumidor. No dupliques regex en múltiples archivos.
