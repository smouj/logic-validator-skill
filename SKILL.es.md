---
name: logic-validator
version: 1.3.1
author: OpenClaw Core Team
description: Análisis estático avanzado y validación de consistencia lógica para scripts y sistemas de reglas de OpenClaw
tags: [logic, validation, qa, static-analysis, correctness, game-logic, rule-validation]
requires: [python3.9+, openclaw-core>=2.3.0, pyparsing>=3.0, networkx>=2.8]
provides: [logic-analysis, inconsistency-detection, dead-code-reduction, state-reachability, rule-contradiction]
conflicts: [logic-validator-legacy, simple-linter]
---
# Skill de Validador Lógico

## Propósito

El Validador Lógico realiza un análisis estático profundo en los sistemas de reglas declarativas, máquinas de estados y lógica de scripts de OpenClaw para identificar:

- **Estados inalcanzables**: Estados del juego que no pueden ser alcanzados desde el estado inicial dadas las reglas de transición actuales
- **Código muerto**: Funciones, reglas o manejadores de estado que nunca son invocados
- **Contradicciones lógicas**: Condiciones conflictivas donde múltiples reglas podrían activarse simultáneamente para el mismo estado del juego
- **Bucles infinitos**: Dependencias cíclicas en transiciones de estado sin condiciones de salida
- **Riesgos de explosión de estados**: Crecimiento exponencial de estados posibles que indica fallos de diseño
- **Transiciones faltantes**: Estados del juego sin rutas de salida válidas
- **Fallas de aserción**: Condiciones que son matemáticamente imposibles dadas las definiciones de reglas
- **Fugas de recursos**: Recursos asignados pero nunca liberados a través de transiciones de estado

### Casos de Uso Reales

1. **Validación pre-despliegue** para nuevos conjuntos de reglas antes de producción
2. **Integración CI/CD** para rechazar automáticamente cambios que rompen la lógica
3. **Validación en fase de diseño** durante la creación de prototipos de mecánicas de juego
4. **Diagnóstico de errores** para comportamiento inesperado del juego analizando grafos de estado
5. **Optimización de rendimiento** identificando ramas de lógica redundantes

## Alcance

### Comandos

#### `logic-validate`
Comando de validación principal con análisis exhaustivo.

```bash
logic-validate <path> [OPTIONS]

Arguments:
  <path>              Path to directory containing OpenClaw rules/scripts (required)

Options:
  --format <fmt>      Output format: "json", "yaml", "sarif", "console" (default: console)
  --check <type>      Specific check to run: "reachability", "contradictions", 
                      "dead-code", "loops", "all" (default: all)
  --max-depth <n>     Maximum depth for state exploration (default: 100)
  --timeout <sec>     Analysis timeout in seconds (default: 30)
  --rules-dir <dir>   Custom rules directory (default: ./rules)
  --scripts-dir <dir> Custom scripts directory (default: ./scripts)
  --exclude <pat>     Exclude files matching pattern (can repeat)
  --severity <lvl>    Minimum severity to report: "info", "warning", "error", "fatal"
  --output <file>     Write output to file instead of stdout
  --visualize <dir>   Generate state machine graphs in specified directory
  --strict            Fail on warnings (exit code 1)
  --ignore <code>     Ignore specific validation codes (comma-separated)

Examples:
  logic-validate ./my-game --format json --output validation.json
  logic-validate . --check contradictions --severity error
  logic-validate ./mod --visualize ./graphs --max-depth 50
```

#### `logic-graph`
Generar e inspeccionar grafos de transición de estado.

```bash
logic-graph <path> [OPTIONS]

Options:
  --format <fmt>      Output format: "dot", "png", "svg", "pdf" (default: dot)
  --layout <algo>     Graph layout: "dot", "neato", "fdp", "sfdp" (default: dot)
  --highlight <node>  Highlight specific state node
  --show-labels       Show transition condition labels
  --simplify          Remove unreachable nodes automatically
  --subgraph <regex>  Extract subgraph matching state pattern

Examples:
  logic-graph . --format png --layout dot --output state_graph.png
  logic-graph . --highlight "combat_active" --show-labels
```

#### `logic-check-condition`
Validar una única expresión de condición contra el contexto de reglas.

```bash
logic-check-condition <condition> [OPTIONS]

Arguments:
  <condition>         Condition expression to validate (required)

Options:
  --context <json>    JSON file providing variable context
  --vars <defs>       Inline variable definitions (format: "x=5,y=3")
  --define <macro>    Define macro for preprocessor (format: "NAME=VALUE")

Examples:
  logic-check-condition "player.health > 0 && enemy.alive"
  logic-check-condition "state == 'attacking'" --context context.json
```

## Proceso de Trabajo Detallado

### 1. Fase de Inicialización

1. Cargar biblioteca núcleo de OpenClaw y configuración desde `openclaw.toml` o `openclaw.json`
2. Analizar argumentos de línea de comandos y validar que las rutas existen
3. Descubrir archivos de reglas (`*.rules`, `*.yaml`, `*.yml`) y archivos de script (`*.js`, `*.lua`, `*.py`) basado en extensiones configuradas
4. Cargar grafo de dependencias para determinar orden de carga
5. Inicializar contexto de validación con variables globales, definiciones de macro y esquemas de tipo

### 2. Parsing Phase

Para cada archivo descubierto:

1. Analizar sintaxis usando analizadores específicos de lenguaje:
   - Reglas basadas en YAML: Validar contra esquema JSON
   - JavaScript/TypeScript: Usar Esprima/Acorn para generación de AST
   - Lua: Usar analizador de Lua con extensiones de OpenClaw
   - Python: Usar módulo ast con transformadores personalizados

2. Extraer constructos lógicos:
   - Definiciones de estado y transiciones
   - Expresiones de condición
   - Manejadores de acción y callbacks
   - Prioridades de reglas y conflictos
   - Declaraciones de variables y ámbitos

3. Construir representación intermedia (IR):
   - Crear nodos para estados, reglas, condiciones, acciones
   - Establecer bordes para transiciones y dependencias
   - Rastrear definiciones y usos de variables con forma SSA

### 3. Analysis Phase

#### Reachability Analysis
```python
# Algorithm: Worklist-based fixed-point iteration
1. Initialize worklist with initial state (from config or "init" convention)
2. While worklist not empty:
   a. Pop state S from worklist
   b. For each rule R applicable in S:
      i. Evaluate condition C using symbolic execution
      ii. If C can be true, add target state T to worklist
      iii. Record edge S -> T with condition C
3. After convergence, compare visited states to total defined states
4. Report states not visited as unreachable
```

#### Contradiction Detection
1. For each state S, collect all rules R that trigger in S
2. For each pair (R1, R2) in S:
   - Check if condition(C1 ∧ C2) is satisfiable using Z3/SMT solver
   - If unsatisfiable, report contradiction
   - If both fire with same priority, warn about ambiguity
3. Check priority rules: ensure no cycles in priority ordering

#### Dead Code Detection
1. Build call graph from:
   - Direct function calls in scripts
   - Event handlers registered via `on()` or `addEventListener`
   - Rule actions that invoke functions
2. Start from entry points:
   - Main game loop callbacks
   - State entry/exit handlers
   - Event dispatch functions
3. Perform reverse reachability analysis
4. Report functions/handlers/rules not reached

#### Loop Detection
1. Extract state transition graph G(V,E)
2. Run cycle detection (Tarjan's algorithm for SCCs)
3. For each cycle C:
   - Check if at least one transition in C has decrementing measure
   - If no measure found, report potential infinite loop
   - If measure exists but could overflow, warn

### 4. Reporting Phase

1. Aggregate all violations found
2. Filter by severity threshold
3. Deduplicate based on root cause
4. Format output according to requested format:
   - Console: Color-coded with file:line:column references
   - JSON: Structured array with code, message, severity, location, suggestions
   - SARIF: Static Analysis Results Interchange Format for IDE integration
   - YAML: Human-readable with nested detail sections
5. If `--visualize` specified:
   - Generate Graphviz DOT file
   - Render to PNG/SVG using `dot` or `neato`
   - Color nodes by reachability status (green=reachable, red=unreachable)
   - Highlight contradiction edges in red, dashed

### 5. Exit Codes

- 0: No issues found (or only info-level)
- 1: Warnings or errors found (or strict mode with warnings)
- 2: Analysis failed (syntax error, invalid config, timeout)
- 3: Invalid command-line arguments

## Reglas de Oro

1. **Sin falsos positivos para inalcanzabilidad**: Si el análisis no puede probar que un estado es inalcanzable, marcarlo como "info" no "error". Se prefiere una sobreaproximación conservadora.
2. **Preservar equivalencia semántica**: Las transformaciones del validador no deben cambiar la semántica del juego; solo analizar, nunca modificar.
3. **Límites simbólicos**: Análisis acotado en profundidad con límites configurables; documentar lo que pueda perderse debido a los límites.
4. **Validación consciente del contexto**: La validación de condiciones debe considerar las funciones incorporadas de OpenClaw (ej. `random()`, `time()`); usar interpretación abstracta para no determinismo.
5. **La prioridad de regla no es eliminación**: Una regla de mayor prioridad que anula una inferior NO hace que la inferior sea "código muerto" - ambas son alcanzables.
6. **Conciencia de efectos secundarios**: Condiciones con efectos secundarios (raros pero posibles) se marcan como errores; la validación lógica asume condiciones puras.
7. **Validación incremental**: Almacenar en caché ASTs analizados y resultados de análisis entre ejecuciones cuando los archivos no cambian (usar hashes de archivo).
8. **Sin llamadas de red**: El validador debe ser determinista; sin llamadas a API externas durante el análisis.
9. **Remediación clara**: Cada violación debe incluir sugerencia concreta para la corrección con ejemplo de código.
10. **Presupuesto de rendimiento**: El tiempo de análisis debería ser < 30s para 10k LOC; usar memoización y poda temprana.

## Ejemplos

### Ejemplo 1: Detección de Estado Inalcanzable

**Archivos:**
`rules/combat.rules`:
```yaml
states:
  - name: idle
    transitions:
      - to: attacking
        condition: player.pressed_attack
  - name: attacking
    transitions:
      - to: idle
        condition: attack_complete
      - to: stunned
        condition: hit_by_enemy
  - name: stunned   # <-- unreachable from idle? Let's see
    on_enter: disable_controls
```

`rules/movement.rules`:
```yaml
states:
  - name: idle
    transitions:
      - to: running
        condition: movement_key_pressed
  - name: running
    transitions:
      - to: idle
        condition: no_movement
```

**Problema**: Dos reglas definen ambas el estado `idle` con diferentes transiciones. Pero OpenClaw fusiona estados a través de archivos. El estado `stunned` es alcanzable desde `attacking` que es alcanzable desde `idle` mediante `player.pressed_attack`. De hecho, esto es alcanzable.

**Ejemplo Modificado** (inalcanzable):
```yaml
states:
  - name: boss_defeated
    on_enter: play_victory
```
No hay transición que conduzca a `boss_defeated` desde ningún estado alcanzable.

**Comando:**
```bash
logic-validate ./rules --format console --severity warning
```

**Salida:**
```
VALIDATION REPORT
=================
File: rules/boss.rules:12
  [WARN] State 'boss_defeated' is unreachable (LVL-REACH-002)
    No transition from any reachable state leads to 'boss_defeated'
    Suggestion: Add transition from 'boss_dying' or remove state

File: rules/combat.rules:8
  [INFO] State 'stunned' reachable via path: idle->attacking->stunned (LVL-REACH-001)
```

### Ejemplo 2: Reglas Contradictorias

**Archivo:** `rules/damage.rules`
```yaml
rules:
  - name: take_damage_if_low_health
    condition: health < 30
    action: take_damage(10)
    priority: 100

  - name: heal_if_low_health
    condition: health < 30
    action: heal(5)
    priority: 100  # <-- Same priority, both fire when health<30
```

**Comando:**
```bash
logic-validate . --check contradictions --format json
```

**Salida (JSON):**
```json
{
  "violations": [
    {
      "code": "LVL-CONT-001",
      "severity": "error",
      "file": "rules/damage.rules",
      "line": 3,
      "message": "Conflicting rules with same priority fire on identical condition",
      "rule1": "take_damage_if_low_health",
      "rule2": "heal_if_low_health",
      "suggestion": "Adjust priorities: make healing lower priority (e.g., 50) or add negation condition"
    }
  ],
  "summary": {"errors": 1, "warnings": 0, "info": 0}
}
```

### Ejemplo 3: Detección de Bucle Infinito

**Archivo:** `scripts/ai_behavior.js`
```javascript
onState('aggressive', () => {
  if (player.distance < 5) {
    setState('attacking');
  } else if (player.distance > 10) {
    setState('chasing');
  }
});

onState('attacking', () => {
  if (player.distance >= 5) {
    setState('aggressive');
  }
});

onState('chasing', () => {
  if (player.distance <= 10) {
    setState('attacking');
  }
});
```

**Problema**: Ciclo en el grafo de estados: aggressive -> attacking -> aggressive. Pero la condición en attacking tiene `player.distance >= 5` mientras que aggressive tiene `player.distance < 5`. Estas son mutuamente excluyentes, por lo que el ciclo no puede repetirse infinitamente. El validador NO debería reportar esto como bucle infinito porque hay una medida (distancia) que cambia.

**Ejemplo Modificado** (bucle infinito verdadero):
```javascript
onState('loop_state', () => {
  setState('loop_state');  // Transición a sí mismo incondicional
});
```

**Comando:**
```bash
logic-validate . --check loops --max-depth 10
```

**Salida:**
```
[ERROR] LVL-LOOP-001: Potential infinite loop detected
  File: scripts/ai_behavior.js:28
  Cycle: loop_state -> loop_state (self-loop)
  No exit condition found on any transition in cycle
  Suggestion: Add condition to transition or ensure state modifies something to eventually break cycle
```

### Ejemplo 4: Uso de Variable No Definida

**Archivo:** `scripts/combat.js`
```javascript
onState('attacking', () => {
  if (player.health < 0) {  // player.health might be undefined if player is null
    // ...
  }
  damage = weapons[current_weapon].damage;  // current_weapon not defined anywhere
});
```

**Comando:**
```bash
logic-check-condition "player.health < 0" --vars "player=null"
```

**Salida:**
```
[WARNING] Possible null dereference: 'player' may be null
  Expression accesses 'health' property
  Suggestion: Add guard: player != null && player.health < 0
```

### Ejemplo 5: Visualizar Grafo de Estado

**Comando:**
```bash
logic-graph ./rules --format png --show-labels --output state_graph.png
```

Genera PNG con:
- Nodos coloreados por estado de alcanzabilidad (verde=alcanzable, gris=inalcanzable)
- Bordes etiquetados con expresiones de condición (simplificadas)
- Ciclos resaltados con fondo rojo
- Subgrafos para conjuntos de reglas modulares

### Ejemplo 6: Integración CI/CD

`.github/workflows/logic-validation.yml`:
```yaml
name: Logic Validation
on: [push, pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install OpenClaw CLI
        run: pip install openclaw
      - name: Run Logic Validator
        run: |
          logic-validate ./rules ./scripts \
            --format sarif \
            --output results.sarif \
            --strict
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: results.sarif
```

El código de salida 1 de `--strict` hace fallar la compilación ante advertencias.

## Variables de Entorno

- `OPENCLAW_VALIDATOR_TIMEOUT`: Sobrescribir tiempo de espera predeterminado (segundos)
- `OPENCLAW_VALIDATOR_MAX_DEPTH`: Sobrescribir profundidad máxima de exploración
- `OPENCLAW_VALIDATOR_CACHE_DIR`: Directorio para caché de ASTs analizados (predeterminado: `~/.cache/openclaw/validator`)
- `OPENCLAW_VALIDATOR_SMT_TIMEOUT`: Tiempo de espera para solucionador SMT por consulta (ms, predeterminado: 1000)
- `OPENCLAW_VALIDATOR_DISABLE_CACHE`: Establecer a "1" para deshabilitar caché
- `OPENCLAW_RULES_PATH`: Directorios de reglas adicionales separados por dos puntos para buscar`

## Dependencias y Requisitos

### Dependencias en Tiempo de Ejecución
- Python ≥3.9
- `openclaw-core` ≥2.3.0 (proporciona abstracciones de AST)
- `pyparsing` ≥3.0 (para análisis de condiciones personalizadas)
- `networkx` ≥2.8 (algoritmos de grafos)
- `z3-solver` ≥4.8.12 (resolución SMT para contradicciones)
- `graphviz` paquete del sistema (para características de visualización)

### Dependencias Opcionales
- `pydot` o `pygraphviz` (renderizado alternativo de grafos)
- `js2py` o `slimit` (respaldo para análisis de JavaScript)
- `lupa` (integración de analizador de Lua)

### Requisitos del Sistema
- 512MB RAM mínimo, 2GB recomendado
- 100MB de espacio en disco para caché
- Ejecutable `dot` de Graphviz en PATH para características de visualización

## Pasos de Verificación

Después de ejecutar el validador:

1. **Verificar código de salida**:
   ```bash
   logic-validate . && echo "PASS" || echo "FAIL"
   ```

2. **Verificar que no haya errores** (las advertencias pueden ser aceptables):
   ```bash
   logic-validate . --format json | jq '.violations[] | select(.severity=="error")' | wc -l
   ```

3. **Asegurar que todos los archivos fueron analizados** (ningún archivo omitido por error de sintaxis):
   ```bash
   logic-validate . --verbose 2>&1 | grep -c "Parsed"
   ```

4. **Validar conectividad del grafo de estado** (todos los estados alcanzables):
   ```bash
   logic-graph . --simplify > /dev/null && echo "Graph contains reachable nodes"
   ```

5. **Verificar que el solucionador SMT funciona** (probar detección de contradicciones):
   ```bash
   echo 'condition: "x > 5 && x < 3"' > test.yaml
   logic-validate . --check contradictions --format json | grep -q "unsatisfiable" && echo "SMT OK"
   ```

6. **Verificación de rendimiento** (completa dentro del tiempo de espera):
   ```bash
   time logic-validate ./large_project --max-depth 100 --timeout 30
   ```

## Solución de Problemas Comunes

### "Error de tiempo de espera del solucionador"
- Aumentar `OPENCLAW_VALIDATOR_SMT_TIMEOUT` o `--timeout`
- Usar `--max-depth` para limitar explosión de estados
- Dividir validación por subdirectorio

### "Memoria agotada"
- Reducir `--max-depth`
- Usar `--check` para ejecutar una verificación a la vez
- Aumentar swap del sistema o RAM

### "Graphviz no encontrado"
- Instalar: `apt-get install graphviz` o `brew install graphviz`
- Usar `--format dot` y renderizar por separado

### Falso positivo "estado inalcanzable"
- El estado puede ser alcanzado vía evento externo (ej. mensaje de red, entrada de usuario de otro módulo)
- Usar `--context` para proporcionar puntos de entrada adicionales
- Marcar estado con comentario `# validator: reachable` para suprimir

### "Errores de análisis de archivo faltante"
- Asegurar que los archivos tienen la extensión correcta (.rules, .js, .lua, .py)
- Verificar sintaxis con l