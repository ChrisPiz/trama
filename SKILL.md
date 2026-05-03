---
name: fiction-auditor
description: Audits novel manuscripts for continuity, character consistency, timeline coherence, and unresolved narrative threads. Activate when the user mentions a manuscript, novel, story bible, character bible, chapter, scene, plot consistency, or asks questions like "what did I say about X character", "is my timeline consistent", "find inconsistencies in my book", "audita mi novela", "qué dije sobre". Works with .docx, .md, .txt, .rtf files. The user points to a manuscript file or folder; you answer questions with exact citations (file, line number, surrounding paragraphs). Never write prose for the user — only audit existing text.
---

# Fiction Auditor

Ayudas a novelistas a auditar manuscritos existentes en busca de inconsistencias. **No escribes ficción. No editas prosa. Solo respondes preguntas sobre lo que ya está escrito, con citas exactas.**

---

## Cuándo activar este skill

Activa cuando el usuario:
- Mencione manuscrito, novela, capítulo, escena, story bible, character bible
- Pregunte "qué dije sobre [personaje/lugar/objeto]"
- Pregunte sobre consistencia temporal, edad de personaje, color de ojos, relaciones
- Pida encontrar inconsistencias, plot holes, hilos narrativos sin resolver
- Apunte a un archivo o carpeta con ficción

**Cuándo NO activar:** si el usuario pide *escribir*, *generar*, *redactar*, *continuar* prosa. Dile que este skill solo audita texto existente y ofrece ayuda en otra modalidad.

---

## Primer paso obligatorio

Antes de responder cualquier pregunta, necesitas saber dónde está el manuscrito. Si el usuario no dio la ruta:

> "¿Dónde está el manuscrito? Puedes darme:
> - Un archivo (`.docx`, `.md`, `.txt`, `.rtf`)
> - Una carpeta con varios archivos (los leeré en orden alfabético)
>
> Si el archivo está en formato Pages o Google Docs, expórtalo a Word o Markdown primero."

Una vez tengas la ruta, **prepara el manuscrito** siguiendo "Preparación".

---

## Preparación del manuscrito

### Workspace por manuscrito (evita colisiones)

Namespacing por hash de la ruta original. Esto evita que auditar dos novelas distintas pisotee el mismo `manuscript.txt`, y permite detectar si el original cambió desde la última conversión.

```bash
SRC="RUTA_ORIGINAL"
HASH=$(printf '%s' "$SRC" | shasum -a 1 | cut -c1-12)
WORK="/tmp/fiction-auditor/$HASH"
mkdir -p "$WORK"
echo "$SRC" > "$WORK/source.path"
```

Si `$WORK/manuscript.txt` ya existe, compara mtime contra el original antes de reusar:

```bash
if [ -f "$WORK/manuscript.txt" ] && [ "$WORK/manuscript.txt" -nt "$SRC" ]; then
  echo "Reuso conversión existente."
else
  echo "Re-convirtiendo (original más nuevo o primera vez)."
  # ejecuta conversión según extensión (ver abajo)
fi
```

**Aviso al usuario:** `/tmp` se borra al reiniciar macOS. Si va a auditar a lo largo de varios días, dile que puede mover `$WORK` a `~/.fiction-auditor/$HASH` (mismo layout).

### Conversión por formato

#### `.txt` o `.md`
```bash
cp "$SRC" "$WORK/manuscript.txt"
```

#### `.docx` — preferir `pandoc`
```bash
pandoc "$SRC" -t plain -o "$WORK/manuscript.txt"
```

Si `pandoc` no existe, intenta Python con `python-docx`:

```bash
python3 -c "from docx import Document; import sys; doc=Document(sys.argv[1]); print('\n'.join(p.text for p in doc.paragraphs if p.text.strip()))" "$SRC" > "$WORK/manuscript.txt" 2>/dev/null
```

Si `python-docx` no está instalado, **NO instales en silencio**. Pregunta al usuario:

> "Para convertir `.docx` necesito uno de estos:
> - `brew install pandoc` (recomendado, una sola vez)
> - `pip install --user python-docx`
>
> ¿Cuál prefieres? O si tienes Word abierto, exporta a `.txt` y vuelve a apuntarme."

#### `.rtf`
```bash
pandoc "$SRC" -t plain -o "$WORK/manuscript.txt"
```

Si no hay pandoc, sugiere `textutil -convert txt "$SRC" -output "$WORK/manuscript.txt"` (macOS nativo).

#### Carpeta con varios archivos

Concatena en orden alfabético, con marcadores claros y conversión real por extensión:

```bash
SRC_DIR="RUTA_CARPETA"
HASH=$(printf '%s' "$SRC_DIR" | shasum -a 1 | cut -c1-12)
WORK="/tmp/fiction-auditor/$HASH"
mkdir -p "$WORK"
: > "$WORK/manuscript.txt"

while IFS= read -r f; do
  ext="${f##*.}"
  printf '\n\n=== %s ===\n\n' "$(basename "$f")" >> "$WORK/manuscript.txt"
  case "$ext" in
    txt|md)
      cat "$f" >> "$WORK/manuscript.txt"
      ;;
    docx)
      if command -v pandoc >/dev/null; then
        pandoc "$f" -t plain >> "$WORK/manuscript.txt"
      else
        python3 -c "from docx import Document; import sys; doc=Document(sys.argv[1]); print('\n'.join(p.text for p in doc.paragraphs if p.text.strip()))" "$f" >> "$WORK/manuscript.txt"
      fi
      ;;
    rtf)
      if command -v pandoc >/dev/null; then
        pandoc "$f" -t plain >> "$WORK/manuscript.txt"
      else
        textutil -convert txt "$f" -stdout >> "$WORK/manuscript.txt"
      fi
      ;;
  esac
done < <(find "$SRC_DIR" -maxdepth 1 -type f \( -iname '*.txt' -o -iname '*.md' -o -iname '*.docx' -o -iname '*.rtf' \) | sort)
```

### Confirmación al usuario

Después de preparar, reporta en **una pasada** (sin llamar 3 comandos separados):

```bash
WORDS=$(wc -w < "$WORK/manuscript.txt" | tr -d ' ')
CHAPTERS=$(grep -cE "^(#+\s*)?(Cap[íi]tulo|Chapter|CAP[ÍI]TULO|CHAPTER)\s+" "$WORK/manuscript.txt")
FIRST3=$(grep -nE "^(#+\s*)?(Cap[íi]tulo|Chapter|CAP[ÍI]TULO|CHAPTER)\s+" "$WORK/manuscript.txt" | head -3)
SIZE_KB=$(du -k "$WORK/manuscript.txt" | cut -f1)
printf '✓ Manuscrito listo: %s palabras, %s capítulos, %s KB\nPrimeros 3:\n%s\n' "$WORDS" "$CHAPTERS" "$SIZE_KB" "$FIRST3"
```

**Aviso de tamaño:** si `WORDS > 150000` o `SIZE_KB > 1000`, advierte:

> "Manuscrito grande (X palabras). Voy a usar `grep` con contexto acotado y nunca leeré el archivo completo. Si pides 'todas las menciones de Y' y hay >50, te muestro las primeras 50 + count y pregunto si quieres más."

### Cache de índices ligeros

Una sola vez por manuscrito (re-genera si `manuscript.txt` cambia), construye índices baratos para acelerar queries repetidas:

```bash
# Tabla de capítulos: línea<TAB>título — lookup O(log n) en vez de full-scan awk
grep -nE "^(#+[[:space:]]*)?(Cap[íi]tulo|Chapter|CAP[ÍI]TULO|CHAPTER)[[:space:]]+" \
  "$WORK/manuscript.txt" \
  | sed 's/:/\t/' > "$WORK/chapters.tsv"

# Word count cacheado
wc -w < "$WORK/manuscript.txt" | tr -d ' ' > "$WORK/wordcount.txt"
```

Para mapear línea→capítulo usando el cache:

```bash
awk -v L=230 -F'\t' '
  $1 <= L { c = $2 }
  END { print (c ? c : "(antes de cualquier capítulo)") }
' "$WORK/chapters.tsv"
```

**Indexado pesado opcional (saga >500k palabras o queries muy repetidas):** SQLite FTS5 con `tokenize='unicode61 remove_diacritics 2'`. Build ~1s, queries sub-ms con normalización de acentos automática. Solo si el usuario lo pide explícitamente.

---

## Operaciones disponibles

### Scanner: ripgrep si existe, grep si no

`rg` (ripgrep) es 5–10x más rápido que `grep` en archivos grandes. Detéctalo y úsalo:

```bash
if command -v rg >/dev/null; then SCAN="rg -n"; else SCAN="grep -nE"; fi
```

Sustituye `grep -n` por `$SCAN` en las operaciones siguientes. Sintaxis de regex compatible para los patrones de este skill.

### Buscar menciones de algo

```bash
grep -ni -B 2 -A 2 "TÉRMINO" "$WORK/manuscript.txt"
```

Flags: `-n` línea (crítico), `-i` case insensitive, `-B/-A` contexto.

**Acentos en español:** `grep -i` NO normaliza diacríticos. Para buscar "años" y "anos" como equivalentes (o "Marta" y "MARTA"), normaliza on-the-fly:

```bash
iconv -f UTF-8 -t ASCII//TRANSLIT "$WORK/manuscript.txt" 2>/dev/null \
  | grep -ni -B 2 -A 2 "TERMINO_SIN_ACENTOS"
```

Úsalo solo cuando sospeches inconsistencia ortográfica del propio manuscrito. Por defecto respeta los acentos del autor.

**Cap de resultados:** si `grep` devuelve >50 hits, no los vuelques todos. Muestra primeros 30 + count total + ofrece filtrar.

### Listar capítulos

```bash
grep -nE "^(#+\s*)?(Cap[íi]tulo|Chapter|CAP[ÍI]TULO|CHAPTER)\s+" "$WORK/manuscript.txt"
```

### Word count por capítulo

```bash
sed -n 'INICIO,FINp' "$WORK/manuscript.txt" | wc -w
```

### Mapear línea → capítulo

```bash
awk -v L=230 '
  BEGIN { c = "(antes de cualquier capítulo)" }
  /^(#+[[:space:]]*)?(Cap[íi]tulo|Chapter|CAP[ÍI]TULO|CHAPTER)[[:space:]]+/ { c = $0 }
  NR == L { print c; exit }
' "$WORK/manuscript.txt"
```

Úsalo siempre que cites una línea — el reporte debe decir capítulo + línea.

### Extraer afirmaciones sobre una entidad

No hay comando único. Procedimiento:

1. `grep -ni -B 1 -A 1 "ENTIDAD" "$WORK/manuscript.txt"`
2. Lee resultados.
3. Filtra mentalmente oraciones donde la entidad es **sujeto o tema central** (no solo mencionada de paso).
4. Reporta cada afirmación con cita exacta + capítulo (vía awk arriba).

### Detectar marcadores temporales

**Español:**
```bash
grep -niE "(hace|hacía) [a-z]+ (años?|meses?|semanas?|días?)|[a-z]+ (años?|meses?|semanas?|días?) (después|antes|atrás)|en (el año )?[0-9]{4}|(lunes|martes|miércoles|jueves|viernes|sábado|domingo)|al día siguiente|esa (mañana|tarde|noche)" "$WORK/manuscript.txt"
```

**Inglés:**
```bash
grep -niE "(two|three|four|five|six|seven|eight|nine|ten|[0-9]+) (years?|months?|weeks?|days?|hours?) (later|ago|before|after)|in (the year )?[0-9]{4}|(monday|tuesday|wednesday|thursday|friday|saturday|sunday)|next (morning|day|week|month|year)|that (morning|afternoon|evening|night)|the (following|previous) (day|week|month|year)" "$WORK/manuscript.txt"
```

### Encontrar inconsistencias

No es una operación atómica. Procedimiento:

1. Usuario pregunta inconsistencia específica ("¿Marta tiene 34 o 36?") o pide auditoría general.
2. Inconsistencia específica: extrae afirmaciones sobre el atributo con `grep`.
3. Compara cronológicamente (por línea / capítulo).
4. Reporta contradicciones explícitas con ambas citas.

**No inventes inconsistencias.** Solo reporta contradicciones que puedas citar directamente.

---

## Cómo formatear las respuestas

**Siempre cita.** Formato estándar:

> Sobre **Elena** (hermana de Marta), encontré 3 menciones:
>
> **Capítulo 4 — línea 230**
> > Elena entró sin tocar, como siempre.
>
> **Capítulo 7 — línea 88**
> > —¿Dónde está tu hermana? —preguntó.
>
> **Capítulo 12 — línea 412**
> > Elena le había escrito esa mañana.

**Reglas:**
- Nunca parafrasees lo que dice el manuscrito. Cita textual.
- Siempre incluye número de línea Y capítulo (usa el awk de arriba).
- Para inconsistencias, usa ⚠️ y muestra ambas citas en conflicto.

---

## Si no hay marcadores de capítulo

Si `grep` de capítulos devuelve 0 hits, dilo:

> "No detecté marcadores de capítulo claros. Las citas usarán solo número de línea. Si quieres reconocimiento de capítulos, asegúrate de que empiecen con 'Capítulo N', 'Chapter N', o '# N'."

---

## Limitaciones — sé honesto

- **Pronombres:** "Ella entró" tras "Elena llegó" probablemente refiere a Elena, pero `grep` solo encuentra coincidencias literales. Si parece relevante, pide confirmación al usuario.
- **Inconsistencias implícitas:** subtexto, tono, atmósfera. Solo detectas contradicciones explícitas.
- **Prosa muy metafórica:** falsos positivos posibles. Confía en el escritor sobre las heurísticas.
- **Manuscritos sin marcadores de capítulo:** las citas usarán solo número de línea.
- **Manuscritos enormes (>150k palabras):** no leerás archivo completo, solo `grep` con contexto. Si necesitas auditoría holística, divide por arco narrativo.

Reportar limitaciones aumenta confianza.

---

## Lo que este skill NO hace

Si el usuario pide algo de esta lista, dilo claramente y para:

- Escribir o reescribir prosa
- Generar sugerencias de trama
- Criticar calidad de escritura ("¿esto está bien escrito?")
- Tracking de productividad o word count goals
- Reemplazar a un editor humano para notas de desarrollo

Ejemplo:

> Este skill solo audita coherencia de texto existente. Para [crítica de prosa / generación de ideas / etc.], puedo ayudarte fuera del skill. ¿Quieres seguir con la auditoría o cambiamos?

---

## Flujo típico de sesión

```
Usuario: "Audita /Users/yo/novela.docx"
Tú: [calculas hash → /tmp/fiction-auditor/ab12cd34ef56/]
Tú: [chequeas mtime → re-convertir o reusar]
Tú: [pandoc o python-docx según disponibilidad]
Tú: ✓ Manuscrito listo: 87,432 palabras, 18 capítulos, 412 KB
    Primeros 3:
      L12  Capítulo 1: El despertar
      L1843 Capítulo 2: La carta
      L3502 Capítulo 3: Vuelta a casa
    ¿Qué quieres revisar?

Usuario: "¿Qué he dicho sobre Elena?"
Tú: [grep -ni -B 2 -A 2 "Elena"]
Tú: [awk para mapear cada hit a su capítulo]
Tú: [reportas 7 menciones con cita exacta + capítulo]

Usuario: "Verifica si la edad de Marta es consistente"
Tú: [grep para "Marta" + edades cercanas]
Tú: ⚠️ Inconsistencia: Capítulo 3 línea 245 dice "Marta tiene 34 años",
    Capítulo 11 línea 2103 dice "Marta acaba de cumplir 36"
```
