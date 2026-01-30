# SOAR Example: Math Problem Solving

This example shows how SOAR helps a student model learn to solve arithmetic problems.

## Problem Setup

**Target Problems** (initially too hard for the student):
1. "If a train travels 120 miles in 2 hours, what is its average speed in miles per hour?" → Answer: 60
2. "A rectangle has length 8 and width 5. What is its perimeter?" → Answer: 26
3. "Calculate: (15 + 25) × 2 - 10" → Answer: 70

**Initial State**: Student has 0% success rate (doesn't know how to solve these)

## Training Loop

### Iteration 1: Building Foundations

**Teacher generates** (adapting to beginner level):
- "What is 10 + 5?" → Teaching basic addition
- "What is 3 × 4?" → Teaching multiplication
- "What is 20 - 8?" → Teaching subtraction

**Student trains** on these simple problems, learns basic operations.

**Evaluation**: Student attempts target problems
- Target 1 (speed): 0% → Still doesn't understand division/rates
- Target 2 (perimeter): 33% → Attempts but gets formula wrong
- Target 3 (order of operations): 0% → Doesn't follow PEMDAS

**Score**: 0.11 (barely improved)

### Iteration 2: Bridging the Gap

**Teacher adapts** (seeing student knows basics but not concepts):
- "If you walk 12 miles in 3 hours, what is your speed?" → Teaching rate concept
- "A rectangle has sides 6 and 4. Add all four sides." → Teaching perimeter directly
- "Calculate: (10 + 5) × 2" → Teaching order of operations

**Student trains** on these intermediate problems.

**Evaluation**: Student attempts targets again
- Target 1: 100% → Understands rate = distance/time
- Target 2: 100% → Learned perimeter = 2(l+w)
- Target 3: 66% → Better at order of operations, minor arithmetic error

**Score**: 0.89 (major improvement!)

### Iteration 3: Convergence

**Teacher refines** (seeing student nearly there):
- "120 ÷ 2 = ?" → Practicing exact numbers from target 1
- "2 × (8 + 5) = ?" → Practicing exact formula from target 2
- "(40) × 2 - 10 = ?" → Practicing exact structure from target 3

**Student trains** on these polishing problems.

**Evaluation**: Student attempts targets
- Target 1: 100% ✓
- Target 2: 100% ✓
- Target 3: 100% ✓

**Score**: 1.0 → **CONVERGED!**

## Key Insights

1. **Adaptive curriculum**: Teacher adjusted difficulty based on student performance
2. **Stepping stones**: Intermediate problems bridged the gap (e.g., "walk 12 miles in 3 hours" → "train 120 miles in 2 hours")
3. **Rapid improvement**: From 0% → 11% → 89% → 100% in just 3 iterations
4. **Grounded reward**: Teacher rewarded for actual improvement, not proxy metrics

## Recipe Configuration

```yaml
context:
  max_iterations: 3
  convergence_threshold: 0.8  # We exceeded this!
  target_problems:
    - problem_id: "target-001"
      problem_text: "If a train travels 120 miles in 2 hours..."
      correct_answer: "60"
    # ... more targets
```

## Expected Output

```
=== SOAR Training Complete ===
Iterations completed: 3
Final score: 1.0
Score progression: [0.0, 0.11, 0.89, 1.0]
Converged: true
Best score achieved: 1.0

SUCCESS: Student mastered all target problems!
Teacher's adaptive curriculum was effective.
```

## Try It Yourself

```bash
amplifier recipes execute --recipe soar:recipes/soar-minimal-test.yaml
```

The actual results will vary based on the LLM's responses, but the pattern should be similar: gradual improvement as the teacher generates increasingly targeted practice problems.
