# SOAR Meta-Learning Bundle

Self-Optimization via Asymmetric RL for Amplifier - enables AI models to teach themselves by generating adaptive training curricula.

## Overview

SOAR (Self-Optimization via Asymmetric RL) implements the meta-learning pattern from ["Teaching Models to Teach Themselves: Reasoning at the Edge of Learnability"](https://arxiv.org/abs/2601.18778) by Sundaram et al. (2026).

### Key Insight

Models can escape reasoning plateaus by generating "stepping-stone" problems that bridge the gap between current capability and target difficulty. The framework uses bi-level meta-RL where:

1. **Teacher agent** generates synthetic training problems
2. **Student model** trains on those problems
3. **Evaluator** measures student improvement on hard target problems
4. Teacher's reward = student's actual improvement (grounded, not proxy metrics)
5. Loop until convergence

### Core Finding

From the paper: **Structural quality > Solution correctness**. Well-posed problems drive learning more than perfectly correct solutions. Models can generate useful stepping stones even for tasks they can't solve directly.

## Installation

```bash
# Add the bundle to your Amplifier configuration
amplifier bundle add git+https://github.com/[your-org]/amplifier-bundle-soar@main
amplifier bundle use soar
```

## Quick Start

Run the minimal test to see SOAR in action (3 iterations, arithmetic problems):

```bash
amplifier recipes execute --recipe soar:recipes/soar-minimal-test.yaml
```

**What this does:**
1. Establishes baseline: student starts with 0% success on target problems
2. **Iteration 1**: Teacher generates practice problems → student trains → evaluator scores
3. **Iteration 2**: Teacher adapts based on student's performance → more training → evaluation
4. **Iteration 3**: Continue until convergence or max iterations
5. Reports final performance and learning trajectory

## Architecture

### Agents

| Agent | Role | Output |
|-------|------|--------|
| `soar:teacher-agent` | Generates optimized training problems | JSON array of problems |
| `soar:student-agent` | Learns from training, attempts targets | JSON solutions with reasoning |
| `soar:evaluator-agent` | Objective performance assessment | JSON scores and convergence signals |

### Recipes

| Recipe | Purpose | Use Case |
|--------|---------|----------|
| `soar-minimal-test` | 3-iteration arithmetic test | Quick validation |
| `soar-iteration` | Single meta-learning iteration | Sub-workflow called by main loop |

### Loop Structure

```yaml
# Main recipe uses while_condition for convergence-based iteration
- while_condition: "{{iteration}} < {{max_iterations}} and not {{converged}}"
  type: "recipe"
  recipe: "soar:recipes/soar-iteration.yaml"
  update_context:
    iteration: "{{iteration}} + 1"
    current_score: "{{iteration_result.score}}"
    converged: "{{iteration_result.converged}}"
  break_when: "{{iteration_result.converged}} == true"
```

Each iteration:
1. Teacher → generates N problems
2. Student → trains on them (foreach loop)
3. Student → attempts target problems (foreach loop)
4. Evaluator → scores and checks convergence
5. Update state → prepare for next iteration

## Usage Patterns

### For Math Problem Solving

```yaml
context:
  target_problems:
    - problem_id: "hard-001"
      problem_text: "Complex algebra problem..."
      correct_answer: "42"
  convergence_threshold: 0.85
  max_iterations: 10
```

### For Code Generation

Adapt the recipe to use code problems:
- Teacher generates programming exercises
- Student writes code solutions
- Evaluator runs test cases

### For Reasoning Tasks

Use logic puzzles or multi-step reasoning:
- Teacher creates inference problems
- Student applies reasoning strategies
- Evaluator checks logical validity

## Configuration

Key parameters in recipe context:

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `max_iterations` | 3 (test) / 10 (full) | Safety limit |
| `convergence_threshold` | 0.8 | Stop when score ≥ this |
| `promotion_threshold` | 0.6 | When to promote student baseline |

## Requirements

### Recipe Loop Enhancements

This bundle requires PR #18 to `amplifier-bundle-recipes` which adds:
- `while_condition`: Convergence-based loops
- `break_when`: Early termination
- `update_context`: State mutation during iteration

**Status**: PR submitted, pending merge.

### Dependencies

```yaml
includes:
  - bundle: git+https://github.com/microsoft/amplifier-foundation@main
  - bundle: git+https://github.com/microsoft/amplifier-bundle-recipes@feat/convergence-loops-for-soar
```

## Example Output

```
=== SOAR Training Complete ===
Iterations completed: 3
Final score: 0.67
Score progression: [0.0, 0.33, 0.67]
Converged: false
Best score achieved: 0.67

Student improved from 0% → 67% success rate
Teacher successfully generated adaptive curriculum
Convergence not reached (threshold: 80%), but clear progress
```

## Advanced Usage

### Custom Domain

1. Define your target problems in recipe context
2. Adapt agent prompts for your domain
3. Customize evaluation logic
4. Adjust thresholds and iteration limits

### Extending SOAR

Add new agents for:
- **Curriculum planner**: Long-term learning strategy
- **Difficulty estimator**: Predict problem hardness
- **Weakness analyzer**: Deep-dive on student gaps

### Integration

Use SOAR as part of larger workflows:
```yaml
steps:
  - id: "analyze-task"
    agent: "foundation:zen-architect"
    prompt: "Analyze what skills are needed"
  
  - id: "meta-learn"
    type: "recipe"
    recipe: "soar:recipes/soar-minimal-test.yaml"
    context:
      target_problems: "{{task_analysis.required_skills}}"
  
  - id: "apply-learned-skills"
    agent: "soar:student-agent"
    prompt: "Solve the original task using learned skills"
```

## Research Background

**Paper**: [Teaching Models to Teach Themselves](https://arxiv.org/abs/2601.18778)  
**Authors**: Shobhita Sundaram, John Quan, Ariel Kwiatkowski, Kartik Ahuja, Yann Ollivier, Julia Kempe  
**Institution**: MIT, Meta FAIR  
**Date**: January 2026

### Key Results

- Unlocks learning on datasets with 0% initial success rate
- Grounded rewards outperform intrinsic reward schemes (avoids instability/collapse)
- Structural quality of problems > solution correctness
- Models can generate useful stepping stones without solving hard problems themselves

## Contributing

Contributions welcome! Areas for improvement:

1. **More domain examples**: Code, reasoning, multi-step tasks
2. **Checkpointing**: Resume long training runs
3. **Visualization**: Plot learning curves
4. **Advanced evaluation**: Partial credit, rubric-based scoring
5. **Teacher policy updates**: Explicit RL update mechanism

## License

[MIT License](LICENSE)

## Citation

If you use SOAR in research, please cite the original paper:

```bibtex
@article{sundaram2026soar,
  title={Teaching Models to Teach Themselves: Reasoning at the Edge of Learnability},
  author={Sundaram, Shobhita and Quan, John and Kwiatkowski, Ariel and Ahuja, Kartik and Ollivier, Yann and Kempe, Julia},
  journal={arXiv preprint arXiv:2601.18778},
  year={2026}
}
```
