---
bundle:
  name: soar
  version: 1.0.0
  description: Self-Optimization via Asymmetric RL meta-learning capability

includes:
  - bundle: git+https://github.com/microsoft/amplifier-foundation@main
  - bundle: git+https://github.com/ramparte/amplifier-bundle-recipes@feat/convergence-loops-for-soar#subdirectory=.
  - bundle: soar:behaviors/soar
---

# SOAR Meta-Learning System

@soar:context/soar-instructions.md

---

@foundation:context/shared/common-system-base.md
