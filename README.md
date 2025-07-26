# Tintin Artifact Evaluation Documentation

This repository contains documentation and resources for evaluating and understanding the Tintin system from the OSDI'25 paper.

## 📚 Documentation Structure

1. **[paper.md](artifact-evaluation/docs/paper.md)** - The original Tintin OSDI'25 paper in markdown format

2. **[code-mapping.md](artifact-evaluation/docs/code-mapping.md)** - Maps key concepts from the paper to their actual code implementations
   - Uncertainty calculation locations
   - Elastic scheduling algorithm
   - ePX implementation details
   - System call mappings

3. **[navigation-guide.md](artifact-evaluation/docs/navigation-guide.md)** - Helps navigate the large codebase (~78,000 files)
   - Explains why there are two repositories
   - Identifies the ~20 core files that matter
   - Provides quick navigation paths
   - Common tasks and commands

4. **[shared-machine-guide.md](artifact-evaluation/docs/shared-machine-guide.md)** - Safety guide for running experiments
   - System impacts and risks
   - Safe experimentation strategies
   - Alternative approaches without kernel access
   - Emergency rollback procedures

## 🎯 Reading Order

For understanding Tintin:
1. Start with `paper.md` to understand the concepts
2. Use `code-mapping.md` to find implementations
3. Refer to `navigation-guide.md` when exploring code

For running experiments:
1. **First** read `shared-machine-guide.md` for safety
2. Follow the safe alternatives before kernel experiments
3. Use `navigation-guide.md` for build commands

## ⚡ Quick Reference

- **Core implementation**: `paper-source/tintin-kernel/kernel/events/tintin_*.c`
- **User tools**: `paper-source/tintin-user/`
- **Runtime config**: `/proc/sys/tintin/` (when kernel is loaded)
- **Key algorithms**: Uncertainty calculation, TAM interpolation, Elastic scheduling