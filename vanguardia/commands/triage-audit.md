---
description: Toma un PLAN.md/AUDIT.md de /audit-frontend, aplica la matriz de godmode-triage, produce plan de ataque ordenado por herramienta (Direct / codex-delegate / /work / decisión humana / archivar). Reduce fricción post-audit.
---

# /triage-audit

Convierte findings de un audit en plan de ataque ejecutable. Decide qué herramienta usar item por item según las reglas codificadas, ordena substantives por peso, archiva lo que no compensa pre-launch, filtra sugerencias que violan reglas duras Strev.

**Idioma**: respuestas al usuario en castellano.

## Argumentos

```
/triage-audit                              # toma el PLAN.md más reciente en notes/audit-design-*/
/triage-audit <ruta-PLAN.md>               # explícito
/triage-audit notes/audit-design-2026-04-29/PLAN.md
```

## Flujo

### Fase A — Cargar contexto

1. Leer reglas: `~/Desktop/VanguardIA/.claude/rules/godmode-triage.md` (4 tablas).
2. Leer reglas Strev: sección "Reglas duras Strev" del `~/Desktop/VanguardIA/projects/strev/CLAUDE.md` y `.claude/rules/strev-frontend.md` si aplican al audit.
3. Leer PLAN.md target. Si no se pasó arg, buscar el más reciente:
   ```bash
   ls -td ~/Desktop/VanguardIA/notes/audit-design-*/PLAN.md | head -1
   ```
4. Si PLAN.md no existe → abort con mensaje claro.

### Fase B — Aplicar matriz

Para cada finding del PLAN.md (P0/P1/P2 con file:line):

1. **Filtro Tabla 4 (reglas duras Strev)**: si viola alguna, mover a ARCHIVADOS con razón.
2. **Tabla 1 (herramienta)**: asignar UNA herramienta según naturaleza del finding.
3. **Tabla 3 (pre-launch)**: si esfuerzo > valor pre-launch, mover a ARCHIVADOS.
4. **Tabla 2 (orden)**: para items que sobrevivan en `/work` o codex-delegate, calcular peso y ordenar.

### Fase C — Producir 5 listas

Output en pantalla y en `notes/audit-design-YYYY-MM-DD/TRIAGE.md` (overwrite si existe).

Estructura:

```markdown
# Triage — <PLAN.md path>

Generado: YYYY-MM-DD HH:MM
Reglas aplicadas: godmode-triage.md (4 tablas) + reglas duras Strev

## Resumen
- N items totales
- A directo · B codex · C /work · D decisión humana · E archivados

---

## LOTE DIRECTO (Claude aplica sin preguntar — 1 commit)

| ID | Descripción | File:line | Esfuerzo |
|---|---|---|---|
| ... | ... | ... | ... |

Commit propuesto: `fix(<area>): lote directo audit YYYY-MM-DD — N quick wins`

## LOTE codex-delegate (Claude propone target + prompt)

| ID | Descripción | Target | Esfuerzo |
|---|---|---|---|
| ... | ... | ... | ... |

## LOTE /work plan + implement (Claude propone, user confirma, ejecuta)

| ID | Descripción | ADR | Esfuerzo |
|---|---|---|---|
| ... | ... | ... | ... |

## DECISIONES HUMANAS

[Numbered options por item, formuladas concretas]

## ARCHIVADOS (con razón)

| ID | Descripción | Razón |
|---|---|---|
| ... | ... | pre-launch · viola regla X · esfuerzo > valor · duplicado |
```

### Fase D — Ejecutar lote DIRECTO automáticamente

Sin preguntar al user. Aplicar todos los items del lote directo:

1. Hacer todos los edits en archivos correspondientes (file:line cita anti-hallucination).
2. Verificar tests si aplica: `cd src/strev-web && npm test -- <test específico si lo hay>`.
3. Commit limpio: `fix(<area>): lote directo audit YYYY-MM-DD — N quick wins`.
4. Mostrar al user: archivos modificados + diff resumen.

### Fase E — Proponer lotes codex + /work + decisiones

UNA vez completo lote directo, mostrar al user las 3 listas restantes con prompt de continuación:

```
LOTE DIRECTO completo (commit <hash>). Siguiente:

(1) Codex-delegate batch — N items, aprox <esfuerzo total>
    [a] Lanzar todos · [b] Item por item · [c] Skip por ahora

(2) /work plan + implement — M items
    Ordenados por peso. Primero: <ID> <descripción> (peso: <X>)
    [a] Arrancar primero · [b] Cambiar orden · [c] Skip

(3) Decisiones humanas — K items
    [Listar 1 por 1 con numbered options]

¿Por dónde seguir? Responde con (1), (2), (3) o "skip todo" para parar.
```

### Fase F — Loop hasta agotar

Por cada lote escogido por user:
- codex-delegate: ejecuta sin más preguntas si user confirmó "todos"
- /work: ejecuta `/work plan` para el siguiente item, espera converge, pregunta si `/work implement`
- decisiones humanas: numbered options, espera respuesta, aplica resultado

Tras agotar todo (o user dice "stop"): mostrar resumen final con commits creados, items pendientes, archivados.

## Reglas no-negociables

1. **Lote DIRECTO se ejecuta sin preguntar al user**. Es la mecánica del comando — sin esto, no aporta valor sobre /audit-frontend.
2. NO aplicar items que violen Tabla 4 (reglas duras). Archivar siempre con razón.
3. Para items archivados pre-launch: dejarlo claro al user al final ("X items archivados pre-launch — revisar post 1-jun-2026 en backlog").
4. Si user discrepa con la asignación de herramienta de un item ("eso lo quiero con /work, no directo"): respetarlo, mover item al lote correcto, re-ejecutar.
5. NO usar emojis en TRIAGE.md ni commits ni respuestas (regla dura Strev).

## Recovery

- PLAN.md no parseable: abort con mensaje. NO inventar findings.
- Item ambiguo (cae en 2 herramientas): default a la más conservadora (`/work plan` > codex > directo).
- Edit directo falla: revertir el item, mover a "ARCHIVADOS — falló edit, revisar manual".
- Claude no entiende el rules file: re-leerlo. Si persiste, abort y pedir al user que valide reglas.

## Ejemplo de invocación

```
/triage-audit notes/audit-design-2026-04-29/PLAN.md
```

→ Claude lee reglas, parsea PLAN.md (20 items pendientes post-Lote A), produce TRIAGE.md, aplica lote directo (4 items, 1 commit), pregunta cuál de los 3 lotes restantes atacar.
