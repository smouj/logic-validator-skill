---
name: logic-validator
version: 1.3.1
author: OpenClaw Core Team
description: Advanced static analysis and logical consistency validation for OpenClaw scripts and rule systems
tags: [logic, validation, qa, static-analysis, correctness, game-logic, rule-validation]
requires: [python3.9+, openclaw-core>=2.3.0, pyparsing>=3.0, networkx>=2.8]
provides: [logic-analysis, inconsistency-detection, dead-code-reduction, state-reachability, rule-contradiction]
conflicts: [logic-validator-legacy, simple-linter]
---
# Logic Validator Skill

## Purpose

Logic Validator performs deep static analysis on OpenClaw's declarative rule systems, state machines, and script logic to identify:

- **Unreachable states**: Game states that cannot be reached from the initial state given current transition rules
- **Dead code**: Functions, rules, or state handlers that are never invoked
- **Logical contradictions**: Conflicting conditions where multiple rules could fire simultaneously for the same game state
- **Infinite loops**: Cyclic dependencies in state transitions without exit conditions
- **State explosion risks**: Exponential growth of possible states indicating design flaws
- **Missing transitions**: Game states with no valid exit paths
- **Assertion failures**: Conditions that are mathematically impossible given the rule definitions
- **Resource leaks**: Resources allocated but never released across state transitions

### Real Use Cases

1. **Pre-deployment validation** for new rule sets before production
2. **CI/CD integration** to automatically reject logic-breaking changes
3. **Design phase validation** during game mechanics prototyping
4. **Bug diagnosis** for unexpected game behavior by analyzing state graphs
5. **Performance optimization** by identifying redundant logic branches

## Scope

### Commands

#### `logic-validate`
Main validation command with comprehensive analysis.

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
Generate and inspect state transition graphs.

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
Validate a single condition expression against rule context.

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

## Detailed Work Process

### 1. Initialization Phase

1. Load OpenClaw core library and configuration from `openclaw.toml` or `openclaw.json`
2. Parse command-line arguments and validate paths exist
3. Discover rule files (`*.rules`, `*.yaml`, `*.yml`) and script files (`*.js`, `*.lua`, `*.py`) based on configured extensions
4. Load dependency graph to determine load order
5. Initialize validation context with global variables, macro definitions, and type schemas

### 2. Parsing Phase

For each discovered file:

1. Parse syntax using language-specific parsers:
   - YAML-based rules: Validate against JSON schema
   - JavaScript/TypeScript: Use Esprima/Acorn for AST generation
   - Lua: Use Lua parser with OpenClaw extensions
   - Python: Use ast module with custom transformers

2. Extract logical constructs:
   - State definitions and transitions
   - Condition expressions
   - Action handlers and callbacks
   - Rule priorities and冲突
   - Variable declarations and scopes

3. Build intermediate representation (IR):
   - Create nodes for states, rules, conditions, actions
   - Establish edges for transitions and dependencies
   - Track variable definitions and usages with SSA form

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

## Golden Rules

1. **No false positives for unreachability**: If analysis cannot prove a state is unreachable, mark as "info" not "error". Conservative over-approximation preferred.
2. **Preserve semantic equivalence**: Validator transformations must not change game semantics; only analyze, never modify.
3. **Symbolic limits**: Depth-bounded analysis with configurable limits; document what may be missed due to bounds.
4. **Context-aware validation**: Condition validation must consider OpenClaw's built-in functions (e.g., `random()`, `time()`); use abstract interpretation for nondeterminism.
5. **Rule priority is not elimination**: Higher priority rule overriding lower does NOT make lower "dead code" - both are reachable.
6. **Side effect awareness**: Conditions with side effects (rare but possible) are flagged as errors; logic validation assumes pure conditions.
7. **Incremental validation**: Cache parsed ASTs and analysis results between runs when files unchanged (use file hashes).
8. **No network calls**: Validator must be deterministic; no external API calls during analysis.
9. **Clear remediation**: Every violation must include concrete suggestion for fix with code example.
10. **Performance budget**: Analysis time should be < 30s for 10k LOC; use memoization and early pruning.

## Examples

### Example 1: Unreachable State Detection

**Files:**
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

**Problem**: Two rules both define `idle` state with different transitions. But OpenClaw merges states across files. The `stunned` state is reachable from `attacking` which is reachable from `idle` via `player.pressed_attack`. Actually, this is reachable.

**Modified Example** (unreachable):
```yaml
states:
  - name: boss_defeated
    on_enter: play_victory
```
No transition leads to `boss_defeated` from any reachable state.

**Command:**
```bash
logic-validate ./rules --format console --severity warning
```

**Output:**
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

### Example 2: Contradictory Rules

**File:** `rules/damage.rules`
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

**Command:**
```bash
logic-validate . --check contradictions --format json
```

**Output (JSON):**
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

### Example 3: Infinite Loop Detection

**File:** `scripts/ai_behavior.js`
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

**Problem**: State graph cycle: aggressive -> attacking -> aggressive. But condition in attacking has `player.distance >= 5` while aggressive has `player.distance < 5`. These are mutually exclusive, so cycle cannot repeat infinitely. Validator should NOT report this as infinite loop because measure (distance) changes.

**Modified Example** (true infinite loop):
```javascript
onState('loop_state', () => {
  setState('loop_state');  // Unconditional self-transition
});
```

**Command:**
```bash
logic-validate . --check loops --max-depth 10
```

**Output:**
```
[ERROR] LVL-LOOP-001: Potential infinite loop detected
  File: scripts/ai_behavior.js:28
  Cycle: loop_state -> loop_state (self-loop)
  No exit condition found on any transition in cycle
  Suggestion: Add condition to transition or ensure state modifies something to eventually break cycle
```

### Example 4: Variable Undefined Use

**File:** `scripts/combat.js`
```javascript
onState('attacking', () => {
  if (player.health < 0) {  // player.health might be undefined if player is null
    // ...
  }
  damage = weapons[current_weapon].damage;  // current_weapon not defined anywhere
});
```

**Command:**
```bash
logic-check-condition "player.health < 0" --vars "player=null"
```

**Output:**
```
[WARNING] Possible null dereference: 'player' may be null
  Expression accesses 'health' property
  Suggestion: Add guard: player != null && player.health < 0
```

### Example 5: Visualize State Graph

**Command:**
```bash
logic-graph ./rules --format png --show-labels --output state_graph.png
```

Generates PNG with:
- Nodes colored by reachability (green=reachable, gray=unreachable)
- Edges labeled with condition expressions (simplified)
- Cycles highlighted with red background
- Subgraphs for modular rule sets

### Example 6: CI/CD Integration

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

Exit code 1 from `--strict` fails the build on warnings.

## Environment Variables

- `OPENCLAW_VALIDATOR_TIMEOUT`: Override default timeout (seconds)
- `OPENCLAW_VALIDATOR_MAX_DEPTH`: Override maximum exploration depth
- `OPENCLAW_VALIDATOR_CACHE_DIR`: Directory for caching parsed ASTs (default: `~/.cache/openclaw/validator`)
- `OPENCLAW_VALIDATOR_SMT_TIMEOUT`: Timeout for SMT solver per query (ms, default: 1000)
- `OPENCLAW_VALIDATOR_DISABLE_CACHE`: Set to "1" to disable caching
- `OPENCLAW_RULES_PATH`: Additional colon-separated rule directories to search

## Dependencies and Requirements

### Runtime Dependencies
- Python ≥3.9
- `openclaw-core` ≥2.3.0 (provides AST abstractions)
- `pyparsing` ≥3.0 (for custom condition parsing)
- `networkx` ≥2.8 (graph algorithms)
- `z3-solver` ≥4.8.12 (SMT solving for contradictions)
- `graphviz` system package (for visualization)

### Optional Dependencies
- `pydot` or `pygraphviz` (alternative graph rendering)
- `js2py` or `slimit` (JavaScript parsing fallback)
- `lupa` (Lua parser integration)

### System Requirements
- 512MB RAM minimum, 2GB recommended
- 100MB disk space for cache
- Graphviz `dot` executable in PATH for visualization features

## Verification Steps

After running validator:

1. **Check exit code**:
   ```bash
   logic-validate . && echo "PASS" || echo "FAIL"
   ```

2. **Verify no errors** (warnings may be acceptable):
   ```bash
   logic-validate . --format json | jq '.violations[] | select(.severity=="error")' | wc -l
   ```

3. **Ensure all files parsed** (no file skipped due to syntax error):
   ```bash
   logic-validate . --verbose 2>&1 | grep -c "Parsed"
   ```

4. **Validate state graph connectivity** (all states reachable):
   ```bash
   logic-graph . --simplify > /dev/null && echo "Graph contains reachable nodes"
   ```

5. **Check SMT solver works** (test contradiction detection):
   ```bash
   echo 'condition: "x > 5 && x < 3"' > test.yaml
   logic-validate . --check contradictions --format json | grep -q "unsatisfiable" && echo "SMT OK"
   ```

6. **Performance sanity** (completes within timeout):
   ```bash
   time logic-validate ./large_project --max-depth 100 --timeout 30
   ```

## Troubleshooting Common Issues

### "Solver timeout" errors
- Increase `OPENCLAW_VALIDATOR_SMT_TIMEOUT` or `--timeout`
- Use `--max-depth` to limit state explosion
- Split validation by subdirectory

### "Memory exhausted"
- Reduce `--max-depth`
- Use `--check` to run one check at a time
- Increase system swap or RAM

### "Graphviz not found"
- Install: `apt-get install graphviz` or `brew install graphviz`
- Use `--format dot` and render separately

### False positive "unreachable state"
- State may be reached via external event (e.g., network message, user input from another module)
- Use `--context` to provide additional entry points
- Mark state with comment `# validator: reachable` to suppress

### Missing file parse errors
- Ensure files have correct extension (.rules, .js, .lua, .py)
- Check syntax with language-specific linter first
- Use `--verbose` to see which files are skipped

### Contradiction not detected
- Condition may be too complex for solver; simplify
- Use abstract domain: try `--domain intervals` or `--domain octagons` if available
- Check that condition uses only supported operators

## Rollback Commands

If validation finds critical issues and you need to revert to previous known-good state:

```bash
# 1. Restore from git if under version control
git checkout HEAD~1 -- rules/ scripts/

# 2. Or restore from backup created by --output
cp /path/to/backup/validation.json.bak ./rules/

# 3. Clear validator cache if analysis was wrong due to old cache
rm -rf ~/.cache/openclaw/validator/*

# 4. Re-run validation on restored state
logic-validate . --format json --output current_validation.json

# 5. Verify rollback success by comparing to previous report
diff current_validation.json backup_validation.json | head -20
```

**Note**: Validator never modifies files; rollback is simply restoring previous versions.

## Advanced: Custom Domain Specific Language (DSL) for Conditions

OpenClaw supports a simplified condition DSL that validator understands:

```
Operators:
  &&   and
  ||   or
  !    not
  >, <, >=, <=, ==, !=
  in   membership (x in [1,2,3])
  matches regex match (name matches "Player*")

Built-in functions:
  random() < 0.5      - treated as nondeterministic (both branches possible)
  time() > 1000       - treated as unbounded (monotonic)
  exists(entity)      - entity may or may not exist

Example:
  (player.health < 30 && player.alive) || 
  (enemy.type in ["boss", "elite"] && time() > 60)
```

Validator abstracts nondeterministic functions to over-approximate reachability. For `random()`, both true and false branches are explored.
```