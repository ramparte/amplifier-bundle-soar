# SOAR Meta-Learning System

You have access to the SOAR (Self-Optimization via Asymmetric RL) meta-learning system.

## Concept

SOAR implements the meta-learning pattern from "Teaching Models to Teach Themselves: Reasoning at the Edge of Learnability" (https://arxiv.org/abs/2601.18778):

1. **Teacher Agent** generates synthetic training problems tailored to student's current skill level
2. **Student Model** trains on those problems
3. **Evaluator** measures student performance on hard target problems
4. Teacher's reward = student's improvement on targets
5. Loop continues until convergence (performance threshold met)

## Key Insight

The paper shows that models can escape reasoning plateaus by generating "stepping-stone" problems that bridge the gap between current capability and target difficulty. Critically:
- **Structural quality > Solution correctness**: Well-posed problems matter more than perfect answers
- **Grounded rewards**: Teacher rewarded based on actual student improvement, not proxy metrics
- **Latent pedagogical ability**: Models can generate useful training problems even for tasks they can't solve directly

## Available Agents

- **soar:teacher-agent**: Generates optimized training problems targeting student weaknesses
- **soar:student-agent**: Learns from training problems, attempts target problems
- **soar:evaluator-agent**: Measures performance objectively on target problem set

## Available Recipes

### soar-minimal-test
Quick 3-iteration test with mock training for validation.

```bash
amplifier recipes execute --recipe soar:recipes/soar-minimal-test.yaml
```

### soar-meta-learning (Future)
Full SOAR training loop with convergence detection.

## Usage Pattern

SOAR is designed to work with domains that have:
- **Sparse rewards**: Initial success rate near 0%
- **Hard target problems**: The problems you want the model to learn
- **Verifiable outcomes**: Can determine if solution is correct
- **Problem generation capacity**: Model can create intermediate problems

## Example Domains

- **Math problem solving**: Teacher generates practice problems at increasing difficulty
- **Code generation**: Teacher creates programming exercises targeting specific concepts
- **Reasoning tasks**: Teacher designs logic puzzles that build prerequisite skills

## Convergence Criteria

Training stops when:
- Student performance ≥ threshold (configurable, default 0.85)
- OR max iterations reached (safety limit)

## Architecture

SOAR uses Amplifier's recipe loop enhancements:
- `while_condition`: Loop until convergence
- `break_when`: Early termination when threshold met
- `update_context`: State mutation during iteration
- Sub-recipes: Student training isolated for independent testing

See `@soar:recipes/soar-minimal-test.yaml` for implementation example.
