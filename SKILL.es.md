name: bug-whisperer
version: 2.1.0
description: Analiza patrones de error, historial de commits y métricas de calidad de código para predecir posibles puntos críticos de bugs y sugerir correcciones preventivas antes de que se manifiesten en producción.
author: OpenClaw Collective
tags:
  - debugging
  - prediction
  - prevention
  - static-analysis
  - regression
dependencies:
  - git>=2.30
  - python>=3.9
  - nodejs>=16.0
  - rubocop:optional
  - eslint:optional
  - bandit:optional
  - radon:optional
  - semgrep:optional
required_env:
  - GIT_REPO_PATH
  - LOG_LEVEL (default: INFO)
  - PREDICTION_THRESHOLD (default: 0.7)
  - EXCLUDE_PATTERNS (default: "vendor/,node_modules/,*.min.js")
capabilities:
  - pattern_analysis
  - regression_prediction
  - anti_pattern_detection
  - heatmap_generation
  - fix_suggestion
---

# Bug Whisperer Skill

## Propósito

Bug Whisperer sirve tres casos de uso principales en el mundo real:

1. **Evaluación de riesgo pre-despliegue**: Antes de fusionar una rama de feature, ejecute `bug-whisperer analyze` para identificar archivos de alto riesgo con alta complejidad ciclomática, código duplicado o patrones de regresión históricos. Uso en CLI: `bug-whisperer --path ./src --output risk-report.json`

2. **Predicción de puntos calientes de regresión**: Analiza el historial de Git para encontrar qué archivos cambian juntos frecuentemente y causan fallos. Ejemplo: `bug-whisperer history --days 90 --threshold 3` muestra archivos que, al modificarse simultáneamente, han causado bugs 3+ veces en los últimos 90 días.

3. **Seguimiento de tendencias de salud del código**: Genera mapas de calor semanales de métricas de calidad de código (complejidad, cobertura de pruebas, churn) para guiar esfuerzos de refactoreo. Uso: `bug-whisperer trends --weeks 12 --output heatmap.html`

La habilidad se ejecuta sin modificar ningún código, proporcionando informes accionables con números de línea específicos y sugerencias de corrección.

## Alcance

Bug Whisperer proporciona estos comandos CLI concretos (todos bajo `bug-whisperer <command>`):

### `analyze [--path <dir>] [--output <file>] [--min-risk <float>]`
Realiza análisis estático en el directorio especificado. Por defecto usa el directorio de trabajo actual. Genera informe JSON por defecto, HTML si `--output` termina en `.html`. `--min-risk` filtra los informes para mostrar solo predicciones con confianza ≥ umbral (por defecto 0.5).

### `history [--days <int>] [--threshold <int>] [--max-commits <int>]`
Analiza el historial de commits de Git para encontrar patrones de regresión. `--days` mira hacia atrás esta cantidad de días (por defecto 30). `--threshold` establece el recuento mínimo de recurrencia para reportar (por defecto 2). `--max-commits` limita el análisis a los N commits más recientes por rendimiento.

### `trends [--weeks <int>] [--output <file>] [--metrics <list>]`
Genera tendencias de salud del código a lo largo del tiempo. `--weeks` analiza esta cantidad de semanas de historial (por defecto 8). `--metrics` acepta lista separada por comas: `complexity,coverage,churn` (por defecto: todas). `--output` crea mapa de calor HTML o CSV.

### `predict [--files <paths>] [--output <file>] [--format <json|md>]`
Dada una lista de archivos (o `--files .` para todos), predice la probabilidad de bugs basándose en el estado actual y patrones históricos. `--format` controla el formato de salida (por defecto: md para terminal, json para scripts).

### `suggest [--file <path> [--line <int>]]`
Para un archivo específico (y opcionalmente línea), genera sugerencias de refactoreo o pruebas dirigidas. Sin `--line`, analiza el archivo completo.

### `scan [--linter <tool>] [--rules <file>] [--fix]`
Ejecuta linters integrados (eslint, rubocop, semgrep) con conjuntos de reglas específicas para patrones de bugs. `--fix` intenta correcciones automáticas seguras. `--rules` apunta a configuración de reglas personalizadas.

## Proceso de Trabajo

Cuando se invoca, Bug Whisperer ejecuta esta secuencia exacta:

1. **Validación de entorno**:
   ```bash
   validate_git_repo() {
     [ -d "$GIT_REPO_PATH/.git" ] || die "Missing .git directory";
     git rev-parse --is-inside-work-tree >/dev/null 2>&1 || die "Not a git repo";
   }
   ```

2. **Descubrimiento de archivos** (respetando `.gitignore` y `EXCLUDE_PATTERNS`):
   - Ejecutar: `git ls-files '*.py' '*.js' '*.rb' '*.go' '*.java'`
   - Filtrar patrones de variable de entorno `EXCLUDE_PATTERNS`
   - Construir lista de archivos para análisis

3. **Pipeline de análisis estático** (parallelizado por extensión de archivo):
   ```
   Para cada archivo:
     - Calcular complejidad ciclomática (radon para Python, complejidad de eslint para JS)
     - Detectar duplicación de código (simian o detector basado en tokens personalizado)
     - Contar líneas de código, ratio de comentarios
     - Identificar comentarios TODO/FIXME con palabras clave de urgencia
   Salida: metrics.json
   ```

4. **Correlación de historial**:
   ```bash
   git log --since="$START_DATE" --format="%H %s" --name-only | \
   awk '/^[0-9a-f]/ {commit=$1} /\.(py|js|rb|go|java)$/ {files[$2]++}'
   ```
   Construir matriz de co-cambios: archivos modificados juntos frecuentemente. Luego:
   - Filtrar pares con ocurrencia conjunta ≥ `--threshold`
   - Cruzar con correcciones de bugs: `git log --grep="bug\|fix\|regression"` para encontrar qué co-cambios introdujeron bugs

5. **Modelo de predicción** (puntuación Bayesiana simple):
   ```
   BugScore(archivo) = 
     0.4 * complejidad_normalizada +
     0.3 * churn_normalizado +
     0.2 * tasa_histórica_de_romper +
     0.1 * brecha_de_cobertura_de_pruebas
   ```
   Archivos con puntuación > `PREDICTION_THRESHOLD` (por defecto 0.7) son marcados como alto riesgo.

6. **Generación de informes**:
   - Formato JSON: array de `{file, line, risk_score, factors, suggested_fixes}`
   - Markdown: tabla legible con códigos de color (rojo >0.8, naranja 0.6-0.8, amarillo 0.4-0.6)
   - HTML: mapa de calor interactivo con árbol de archivos

7. **Motor de sugerencia de correcciones**:
   Para cada elemento de alto riesgo, cargar plantillas específicas del lenguaje:
   ```
   if complexity > 10: sugerir refactorización "Extract Method"
   if duplicate code found: sugerir "Extract Class/Module"
   if high churn + no tests: sugerir "Add unit tests for X function"
   ```
   Proporcionar fragmentos de código exactos con before/after.

## Reglas de Oro

- **Nunca modificar archivos fuente**: Bug Whisperer es de solo lectura. Todas las sugerencias son consultivas.
- **Siempre respetar `.gitignore`**: Nunca analizar archivos ignorados, incluso si existen en el árbol de trabajo.
- **Precisión histórica**: Solo considerar commits fusionados a la rama main/master para el análisis de historial. Ignorar ramas de feature a menos que se solicite explícitamente con `--include-branches`.
- **Transparencia de umbral**: Documentar el `PREDICTION_THRESHOLD` exacto usado en el informe. Nunca cambiar umbrales silenciosamente.
- **Sin falsos positivos en pruebas**: No marcar archivos de prueba como fuentes de bugs; en su lugar, marcar pruebas faltantes para código riesgoso.
- **Protección de rendimiento**: Si el repositorio tiene >10k archivos, por defecto usar muestreo (top 1000 por churn) a menos que se especifique `--full-scan`.
- **Amigable con CI/CD**: Código de salida 0 si no se encontraron bugs de alto riesgo, 1 si se encontró alguno, 2 en error. Sin prompts interactivos.
- **Reproducibilidad de informes**: Incluir hash de commit de git, timestamp y versiones de herramientas en cada informe.

## Ejemplos

### Ejemplo 1: Verificación pre-commit
El usuario ejecuta:
```bash
bug-whisperer analyze --path ./src --min-risk 0.6 --output risk.json
```
La habilidad produce:
```json
{
  "meta": {
    "git_commit": "a1b2c3d",
    "timestamp": "2026-03-04T10:30:00Z",
    "threshold": 0.6,
    "files_scanned": 142
  },
  "high_risk": [
    {
      "file": "src/auth/login.py",
      "lines": [45, 112],
      "risk_score": 0.82,
      "factors": {
        "complexity": 15,
        "historical_breakage": 4,
        "test_coverage": 0.35
      },
      "suggestions": [
        "Extract method 'validate_token' starting at line 45",
        "Add unit tests covering edge cases in login flow"
      ]
    }
  ]
}
```

### Ejemplo 2: Descubrimiento de patrones de regresión
El usuario ejecuta:
```bash
bug-whisperer history --days 60 --threshold 3
```
La habilidad produce:
```
Regression Patterns (co-changed files causing bugs ≥3 times):
1. src/models/user.js + src/api/auth.js
   Occurrences: 5
   Sample bugs: "Login fails after password reset", "User session lost"
2. src/components/Form.js + src/utils/validation.js
   Occurrences: 3
   Sample bugs: "Validation bypassed on Safari", "Form submits twice"
Run: bug-whisperer suggest --file src/models/user.js for detailed fixes.
```

### Ejemplo 3: Integración CI/CD (GitHub Actions)
```yaml
- name: Predict Bugs
  run: |
    bug-whisperer predict --files . --format json > predictions.json
    if jq -e '.high_risk | length > 0' predictions.json > /dev/null; then
      echo "::error::High-risk files detected. See predictions.json"
      exit 1
    fi
```

## Comandos para Revertir

Bug Whisperer es de solo lectura y solo produce informes. No se modifica el estado del sistema. Para "revertir":

1. **Descartar archivos de informes generados**:
   ```bash
   rm -f risk-report.json heatmap.html predictions.md
   ```

2. **Deshacer cambios accidentales** (no debería ocurrir nunca, pero si se usó `--fix` por error con un linter):
   ```bash
   git reset --hard HEAD
   git clean -fd
   ```

3. **Revertir cambios de entorno** (si el usuario configuró manualmente `EXCLUDE_PATTERNS` o `PREDICTION_THRESHOLD` en el shell):
   ```bash
   unset EXCLUDE_PATTERNS PREDICTION_THRESHOLD
   # O restaurar desde backup: source ~/.openclaw/env-backup.sh
   ```

4. **Eliminar caché de análisis temporal** (almacenado en `.bugwhisperer_cache/`):
   ```bash
   rm -rf .bugwhisperer_cache
   ```

Dado que la habilidad solo lee código y produce archivos, la reversión consiste simplemente en eliminar sus salidas.