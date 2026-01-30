# Recipe Loop Enhancements for SOAR

## Overview

This specification defines three new recipe capabilities to enable meta-learning patterns like SOAR:

1. **While loops** - Convergence-based iteration
2. **Break conditions** - Early termination
3. **State mutation** - Dynamic context updates

## 1. While Loops

### Motivation

Current `foreach` loops require predetermined iteration counts. Meta-learning needs convergence-based loops.

### Design

Add `while_condition` field to Step:

```yaml
steps:
  - id: "meta-learning-loop"
    while_condition: "{{performance}} < {{threshold}}"
    max_while_iterations: 100  # Safety limit (default: 100)
    steps:
      - id: "teacher-generate"
        agent: "curriculum:teacher"
        output: "curriculum"
      
      - id: "student-train"
        agent: "curriculum:student"
        output: "results"
      
      - id: "evaluate"
        agent: "curriculum:evaluator"
        prompt: "Measure performance: {{results}}"
        output: "performance"
```

### Semantics

- `while_condition` is evaluated **before each iteration** (like C/Python while loops)
- Supports same expression syntax as `condition` (variables, comparisons, boolean logic)
- `max_while_iterations` prevents infinite loops (default: 100, configurable 1-1000)
- Loop exits when condition becomes false OR max iterations reached
- Can be combined with `foreach` - while loops over each foreach item

### Validation

- `while_condition` must contain `{{variable}}` reference
- Cannot use both `while_condition` and `foreach` on same step (mutually exclusive)
- `max_while_iterations` must be 1-1000

## 2. Break Conditions

### Motivation

Meta-learning needs early exit when convergence detected mid-iteration.

### Design

Add `break_when` field to Step:

```yaml
steps:
  - id: "learning-iterations"
    foreach: "{{range(100)}}"
    as: "iteration"
    break_when: "{{converged}} == 'true'"
    steps:
      - id: "attempt"
        agent: "student"
        output: "result"
      
      - id: "check-convergence"
        agent: "evaluator"
        prompt: "Has student converged? {{result}}"
        output: "converged"
```

### Semantics

- `break_when` is evaluated **after each iteration** completes successfully
- If condition evaluates to true, loop exits immediately (remaining iterations skipped)
- Works with both `foreach` and `while_condition` loops
- Collected results include all iterations up to (and including) the breaking iteration

### Validation

- `break_when` must contain `{{variable}}` reference
- Requires either `foreach` or `while_condition` (break without loop is error)

## 3. State Mutation

### Motivation

Meta-learning needs to update baseline/curriculum state dynamically during loops.

### Design

Add `update_context` field to Step:

```yaml
steps:
  - id: "promote-baseline"
    condition: "{{performance}} > {{promotion_threshold}}"
    update_context:
      baseline_state: "{{trained_student}}"
      promoted_problems: "{{promoted_problems}} + {{new_problems}}"
      best_performance: "{{performance}}"
```

### Semantics

- `update_context` is a dictionary mapping context variable names to expressions
- Expressions are evaluated and assigned to context variables **after step execution**
- Supports all expression syntax (variables, arithmetic, concatenation, function calls)
- Updated variables are available to subsequent steps
- Arrays can be concatenated: `"{{list1}} + {{list2}}"` → merged array

### Validation

- Keys must be valid variable names (alphanumeric with underscores)
- Values must contain valid expressions
- Cannot overwrite reserved names: `recipe`, `session`, `step`

## Implementation Plan

### Phase 1: Models (models.py)

```python
@dataclass
class Step:
    # Existing fields...
    
    # New loop fields
    while_condition: str | None = None  # Convergence-based loop
    max_while_iterations: int = 100     # Safety limit for while loops
    break_when: str | None = None       # Early exit condition
    
    # New state mutation field  
    update_context: dict[str, str] | None = None  # Context variable updates
    
    def validate(self) -> list[str]:
        errors = []
        
        # While loop validation
        if self.while_condition:
            if "{{" not in self.while_condition:
                errors.append(f"while_condition must contain {{{{variable}}}} reference")
            if self.foreach:
                errors.append(f"Cannot use both while_condition and foreach")
            if not 1 <= self.max_while_iterations <= 1000:
                errors.append(f"max_while_iterations must be 1-1000")
        
        # Break validation
        if self.break_when:
            if "{{" not in self.break_when:
                errors.append(f"break_when must contain {{{{variable}}}} reference")
            if not self.foreach and not self.while_condition:
                errors.append(f"break_when requires foreach or while_condition")
        
        # State mutation validation
        if self.update_context:
            for key, value in self.update_context.items():
                if not key.replace("_", "").isalnum():
                    errors.append(f"update_context key must be alphanumeric: {key}")
                if key in ("recipe", "session", "step"):
                    errors.append(f"update_context cannot overwrite reserved name: {key}")
        
        return errors
```

### Phase 2: Executor (executor.py)

#### While Loop Execution

```python
async def _execute_while_loop(
    self,
    step: Step,
    context: dict[str, Any],
    project_path: Path,
    recursion_state: RecursionState,
    # ... other params
) -> list[Any]:
    """Execute while loop until condition false or max iterations."""
    results = []
    iteration = 0
    
    while iteration < step.max_while_iterations:
        # Evaluate condition BEFORE iteration
        try:
            condition_met = evaluate_condition(step.while_condition, context)
        except ExpressionError as e:
            raise ValueError(f"while_condition evaluation failed: {e}")
        
        if not condition_met:
            break  # Exit loop
        
        # Execute step body
        # (similar to foreach sequential execution)
        
        # Check break_when AFTER iteration
        if step.break_when:
            try:
                should_break = evaluate_condition(step.break_when, context)
                if should_break:
                    break
            except ExpressionError:
                pass  # Continue if break condition evaluation fails
        
        iteration += 1
    
    return results
```

#### State Mutation

```python
def _apply_context_updates(
    self,
    step: Step,
    context: dict[str, Any]
) -> None:
    """Apply update_context mappings to context."""
    if not step.update_context:
        return
    
    for var_name, expression in step.update_context.items():
        try:
            # Use expression evaluator to resolve expression
            value = self._evaluate_expression(expression, context)
            context[var_name] = value
        except ExpressionError as e:
            raise ValueError(
                f"Step '{step.id}': update_context['{var_name}'] failed: {e}"
            )
```

### Phase 3: Integration Points

Modifications needed in `_execute_step_iteration`:

```python
# After step execution, before storing output:
# 1. Apply context updates
self._apply_context_updates(step, context)

# 2. Check break condition
if step.break_when and self._in_loop:
    should_break = evaluate_condition(step.break_when, context)
    if should_break:
        raise BreakLoopException()  # New exception type
```

## Example: SOAR Recipe

```yaml
name: "soar-meta-learning"
version: "1.0.0"
description: "SOAR meta-RL with convergence-based loops"

context:
  performance: 0.0
  threshold: 0.85
  baseline_state: ""
  promoted_problems: []

steps:
  - id: "initialize"
    type: "bash"
    command: |
      echo '{"performance": 0.0}' > baseline.json
    output: "init"
  
  # Convergence-based meta-learning loop
  - id: "meta-learning"
    while_condition: "{{performance}} < {{threshold}}"
    max_while_iterations: 50
    break_when: "{{converged}} == 'true'"
    steps:
      - id: "teacher-generate"
        agent: "soar:teacher"
        prompt: |
          Generate curriculum for baseline: {{baseline_state}}
          Current performance: {{performance}}
        output: "curriculum"
      
      - id: "student-train"
        agent: "soar:student"
        prompt: |
          Train on: {{curriculum}}
          Starting from: {{baseline_state}}
        output: "trained_student"
      
      - id: "evaluate"
        agent: "soar:evaluator"
        prompt: |
          Evaluate student: {{trained_student}}
          Report: performance (float), converged (bool)
        parse_json: true
        output: "evaluation"
      
      - id: "update-state"
        update_context:
          performance: "{{evaluation.performance}}"
          converged: "{{evaluation.converged}}"
      
      - id: "promote-if-improved"
        condition: "{{evaluation.performance}} > {{performance}}"
        update_context:
          baseline_state: "{{trained_student}}"
          promoted_problems: "{{promoted_problems}} + {{curriculum}}"
```

## Backward Compatibility

- All new fields are optional with sensible defaults
- Existing recipes continue to work unchanged
- New features opt-in via explicit field usage

## Testing Strategy

1. **Unit tests**: Test each new field validation in models.py
2. **Expression tests**: Test while_condition and break_when evaluation
3. **Integration tests**: Test complete SOAR-like workflow
4. **Edge cases**: Empty loops, condition evaluation failures, max iterations

## Migration Path

Phase 1: Implement models + validation → PR
Phase 2: Implement executor logic → PR  
Phase 3: Add documentation + examples → PR
Phase 4: Build SOAR bundle using new features → Separate repo
