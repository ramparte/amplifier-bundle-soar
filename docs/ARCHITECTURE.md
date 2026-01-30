# SOAR Bundle Architecture

This document explains the design decisions and implementation patterns used in the SOAR bundle.

## Design Philosophy

### Thin Bundle Pattern

Following Amplifier's philosophy, SOAR is a **thin bundle**:

```yaml
includes:
  - bundle: amplifier-foundation     # Core capabilities
  - bundle: amplifier-bundle-recipes # Recipe orchestration
  - bundle: soar:behaviors/soar      # SOAR-specific agents
```

- **Foundation provides**: Agent spawning, task coordination, basic patterns
- **Recipes provide**: Loop orchestration, state management, checkpointing
- **SOAR provides**: Specialized agents + meta-learning recipes

### Composed Sub-Recipes

Rather than one monolithic recipe, SOAR uses composition:

```
soar-minimal-test.yaml
├── Initial evaluation (bash)
├── Meta-learning loop (while_condition)
│   └── Calls: soar-iteration.yaml
│       ├── Teacher generates (agent)
│       ├── Student trains (foreach → agent)
│       ├── Student evaluates (foreach → agent)
│       ├── Evaluator scores (agent)
│       └── Convergence check (agent)
└── Final report (bash)
```

**Benefits**:
- Each component testable independently
- Sub-recipe reusable in other contexts
- Clear separation of concerns
- Easier to understand and maintain

## Agent Design

### Agent Roles

| Agent | Type | Stateful? | Responsibility |
|-------|------|-----------|----------------|
| `teacher-agent` | Policy | No | Generate problems (receives history, returns problems) |
| `student-agent` | Learner | Implicit | Solve problems (improves via repetition) |
| `evaluator-agent` | Critic | No | Measure performance (deterministic scoring) |

### Agent Statefulness

**Key insight**: Agents themselves are stateless in Amplifier. State lives in recipe context.

```yaml
context:
  iteration: 0              # Shared state
  score_history: []         # Shared state
  current_score: 0.0        # Shared state

steps:
  - agent: "soar:teacher-agent"
    prompt: |
      Generate problems.
      History: {{score_history}}  # State passed via prompt
```

The "student learning" effect emerges from:
1. Teacher adapting to performance feedback (via prompts)
2. Student seeing increasingly targeted examples
3. Evaluator measuring improvement

There's no explicit "model update" - the meta-learning comes from prompt engineering and problem selection.

## Loop Implementation

### Why `while_condition` + `update_context`?

SOAR needs convergence-based iteration (not fixed-count loops):

```yaml
- id: "meta-learning-loop"
  while_condition: "{{iteration}} < {{max_iterations}} and not {{converged}}"
  type: "recipe"
  recipe: "soar:recipes/soar-iteration.yaml"
  update_context:
    iteration: "{{iteration}} + 1"
    converged: "{{iteration_result.converged}}"
    current_score: "{{iteration_result.score}}"
    score_history: "{{score_history}} + [{{iteration_result.score}}]"
  break_when: "{{iteration_result.converged}} == true"
```

**What this enables**:
1. Loop until convergence (not "run N times")
2. Mutate state between iterations (score history accumulates)
3. Early termination (break when threshold met)
4. Safety limit (max_iterations prevents infinite loops)

### Alternative: `foreach` with Fixed List

We could generate a fixed list `[1, 2, 3, ..., 10]` and use `foreach`. Problems:
- Wasteful if converging early (run 10 iterations even if done at 3)
- No early termination (can't break_when)
- Doesn't match the conceptual model (convergence-driven, not count-driven)

## State Management

### Context Variables

| Variable | Type | Updated By | Used By |
|----------|------|------------|---------|
| `iteration` | int | `update_context` | Teacher (adapts based on iteration) |
| `score_history` | list[float] | `update_context` | Teacher (sees trend), Evaluator (checks convergence) |
| `current_score` | float | `update_context` | Teacher (adapts curriculum) |
| `converged` | bool | `update_context` | Loop condition, break_when |
| `best_score` | float | `update_context` | Final report |

### Data Flow

```
Iteration 0:
  score_history = []
  ↓
Teacher → problems → Student → solutions → Evaluator → score
  ↓
update_context: score_history = [0.33]
  ↓
Iteration 1:
  score_history = [0.33]  # Available to teacher!
  ↓
Teacher → (adapts based on [0.33]) → problems → ...
  ↓
update_context: score_history = [0.33, 0.67]
```

## JSON Parsing Strategy

### Why `parse_json: true`?

Agents return structured data that must be parsed for subsequent steps:

```yaml
- agent: "soar:teacher-agent"
  parse_json: true  # Extract problems array
  output: "generated_problems"

- foreach: "{{generated_problems.problems}}"  # Access .problems field
```

**Without parsing**: Would get string `"{\"problems\": [...]}"`  
**With parsing**: Get object with `.problems` field accessible

### Where NOT to parse

Bash steps that construct JSON:

```yaml
- type: "bash"
  command: |
    cat << 'EOF'
    {"score": {{evaluation.overall_score}}}
    EOF
  parse_json: true  # Parse the bash output
```

## Error Handling

### Fail-Fast Philosophy

Following recipe design principles, SOAR fails fast:

```yaml
foreach: "{{generated_problems.problems}}"
# If any student solution fails → entire iteration fails → meta-loop fails
```

**Rationale**: In early development, silent failures hide bugs. Fail visibly, fix immediately.

### Future: Partial Completion

For production use, add:

```yaml
foreach: "{{generated_problems.problems}}"
continue_on_error: true
collect_errors: "training_errors"
```

Then handle errors explicitly (skip problem, retry with hint, etc.).

## Testing Strategy

### Layered Testing

1. **Unit**: Test each agent individually (mock inputs)
2. **Sub-recipe**: Test `soar-iteration.yaml` with known inputs
3. **Integration**: Test full `soar-minimal-test.yaml` end-to-end
4. **Validation**: Use `recipes:result-validator` to check outcomes

### Minimal Test Design

The minimal test uses:
- **3 iterations**: Fast feedback, shows progression
- **Simple arithmetic**: Easy to verify correctness
- **Mocked values**: Example JSON in prompts (guides agent output)

This validates:
- ✓ Recipe syntax
- ✓ Loop logic
- ✓ State updates
- ✓ Agent coordination
- ✓ JSON parsing
- ✓ Convergence detection

## Extension Points

### Adding New Agents

```yaml
# In behaviors/soar.yaml
agents:
  include:
    - soar:agents/teacher-agent
    - soar:agents/student-agent
    - soar:agents/evaluator-agent
    - soar:agents/curriculum-planner  # NEW
```

Then use in recipes:

```yaml
- id: "plan-curriculum"
  agent: "soar:curriculum-planner"
  prompt: "Design 10-iteration learning path for {{target_problems}}"
```

### Adding Evaluation Metrics

Extend evaluator to return:

```json
{
  "overall_score": 0.67,
  "subscores": {
    "correctness": 0.8,
    "reasoning_quality": 0.6,
    "confidence_calibration": 0.5
  }
}
```

Then use in convergence:

```yaml
while_condition: "{{evaluation.subscores.correctness}} < {{threshold}}"
```

### Custom Domain Adaptation

1. Replace `target_problems` with your domain's problem format
2. Update agent prompts for domain-specific language
3. Customize evaluator to check domain-specific correctness
4. Adjust thresholds and iteration limits for your difficulty

## Performance Considerations

### Token Usage

Each iteration spawns multiple agents:
- 1 teacher call (generate problems)
- N student calls (training, foreach)
- M student calls (evaluation, foreach)
- 1 evaluator call
- 1 convergence check

**Optimization**: Use `parallel: true` for independent student calls:

```yaml
foreach: "{{generated_problems.problems}}"
parallel: 5  # Max 5 concurrent student calls
```

### Cost Management

Use provider preferences for cost/performance trade-offs:

```yaml
steps:
  - id: "generate-problems"
    agent: "soar:teacher-agent"
    provider: "anthropic"
    model: "claude-sonnet-*"  # Balance of speed/quality
  
  - id: "quick-convergence-check"
    agent: "soar:evaluator-agent"
    provider: "anthropic"
    model: "claude-haiku"  # Fast/cheap for simple logic
```

## Future Enhancements

### 1. Explicit RL Updates

Current: Implicit learning via prompt engineering  
Future: Explicit policy gradient updates

### 2. Checkpointing

Current: No mid-training resume  
Future: Save state after each iteration, resume on failure

### 3. Multi-Student Comparison

Current: Single student  
Future: Train multiple students, compare strategies

### 4. Visualization

Current: Text output only  
Future: Learning curve plots, problem difficulty heatmaps

## References

- [Amplifier Recipe Schema](https://github.com/microsoft/amplifier-bundle-recipes/blob/main/docs/RECIPE_SCHEMA.md)
- [SOAR Paper](https://arxiv.org/abs/2601.18778)
- [Foundation Bundle Guide](https://github.com/microsoft/amplifier-foundation/blob/main/docs/BUNDLE_GUIDE.md)
