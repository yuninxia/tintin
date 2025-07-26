# Tintin Navigation Guide - Making Sense of the Codebase

## 🎯 The Big Picture

Tintin is a hardware performance monitoring system that solves two fundamental problems:
1. **Measurement errors** - When you have more events to monitor than hardware counters
2. **Attribution inflexibility** - Current tools can't flexibly define what code/scope to profile

## 📁 Why Two Repositories?

### 1. **tintin-kernel** (The Engine)
- **What**: Modified Linux kernel v5.15
- **Why separate**: Kernel modifications require a full Linux kernel source tree
- **Size**: ~78,000 files (most are original Linux files)
- **Your focus**: Only ~20 files in `kernel/events/tintin_*.{c,h}`

### 2. **tintin-user** (The Interface) 
- **What**: User-space tools, tests, and evaluation scripts
- **Why separate**: Easier to develop/test without rebuilding kernel
- **Size**: Much smaller, ~100 files
- **Your focus**: Case studies, plotting scripts, test programs

## 🗺️ Quick Navigation Map

```
tintin/
├── paper-source/           # Original code from paper authors
│   ├── tintin-kernel/      # HUGE but you only need...
│   │   └── kernel/events/  # 👈 START HERE (Tintin's core ~20 files)
│   │       ├── tintin_uncertainty.c    # Variance & uncertainty calculation
│   │       ├── tintin_elastic.c        # Scheduling algorithm
│   │       ├── tintin_estimation.c     # TAM interpolation
│   │       ├── tintin_interface.c      # Reading events with uncertainty
│   │       ├── tintin_profiling_context.{c,h}  # ePX implementation
│   │       └── core.c                  # Modified Linux file (search "Tintin's code")
│   │
│   └── tintin-user/        # SMALL & focused
│       ├── case-studies/   # Real-world examples
│       │   ├── Pond/       # Cloud resource provisioning
│       │   └── DMon/       # Performance debugging
│       ├── plots/          # Python scripts to reproduce paper figures
│       └── REPRODUCE.md    # Step-by-step evaluation guide
│
└── artifact-evaluation/    # Your own experiments go here
```

## 🔍 Where to Find What You Need

### Core Algorithm Implementations

| What You Want | Where to Find It | Key Function |
|--------------|------------------|--------------|
| How uncertainty is calculated | `tintin_uncertainty.c` | `tintin_update_uncertainty()` line 160 |
| Welford's variance method | `tintin_uncertainty.c` | `tintin_update_variance_by_Welfords_method()` line 166 |
| Event interpolation (TAM) | `tintin_estimation.c` | `tintin_TAM()` line 20 |
| Elastic scheduling | `tintin_elastic.c` | `tintin_elastic()` line 24 |
| ePX data structure | `tintin_profiling_context.h` | `struct tintin_profiling_context` line 20 |

### Configuration & Testing

| What You Want | Where to Find It |
|--------------|------------------|
| Runtime parameters | `/proc/sys/tintin/` (when kernel is running) |
| Benchmark configs | `~/cpu2017_instrumented/config/` |
| Test results | `/tmp/tintin_*.log` |
| Plotting scripts | `tintin-user/plots/figure-*.py` |

## 💡 Understanding the Flow

### 1. **Event Monitoring Flow**
```
User Request → tintin_event_open() → Create tintin_event
     ↓
Timer Interrupt → Update counts → Calculate variance → Update uncertainty
     ↓
Read Request → TAM interpolation → Return (count, uncertainty)
```

### 2. **Scheduling Flow**
```
Every 4ms (hyperperiod) → Collect all events → Calculate elasticity
     ↓
Run elastic algorithm → Assign time shares → Schedule on HPCs
```

### 3. **ePX (Profiling Context) Flow**
```
Create ePX → Define scope (task/CPU/code) → Add events
     ↓
Context switch → Update ePX → Filter unrelated events
```

## 🚀 Quick Start Paths

### Path 1: "I want to understand the algorithms"
1. Read `tintin_uncertainty.c` - How measurement errors are quantified
2. Read `tintin_elastic.c` - How events are scheduled optimally
3. Read `tintin_estimation.c` - How counts are interpolated

### Path 2: "I want to run experiments"
1. Start with `REPRODUCE.md` in tintin-user
2. Look at `plots/` directory for data processing
3. Check `/tmp/tintin_*.log` for results

### Path 3: "I want to modify/extend Tintin"
1. Understand data structures in `include/linux/tintin_event.h`
2. Look at syscalls in `tintin_interface.c`
3. Test with programs in `tintin-user/tests/`

## 📝 Key Concepts Simplified

### Uncertainty = "How wrong might this measurement be?"
- Calculated as: √(variance) × time_not_monitored
- Higher variance = more uncertainty
- Longer unmonitered time = more uncertainty

### Elastic Scheduling = "Give more time to uncertain events"
- Events with high uncertainty get priority
- Minimizes total squared error across all events
- Optimal solution in O(n log n) time

### ePX = "Flexible profiling scope"
- Can profile: specific functions, processes, CPU cores, or combinations
- Resolves conflicts when multiple scopes overlap
- First-class kernel object (like files or processes)

## 🔧 Common Tasks

### Enable Tintin
```bash
echo 1 > /proc/sys/tintin/tintin_sched_switch_on
```

### Switch to specific policy
```bash
echo 1 > /proc/sys/tintin/tintin_sched_policy  # 0=RR, 1=elastic, 2=uncertainty-first
```

### Run a benchmark
```bash
cd ~/cpu2017_instrumented
./bin/runcpu --config=tintin-gcc-linux-x86.cfg --size=ref --action run 500
cat /tmp/tintin_cpu2017_spec_perlbench_500.log
```

## 🎓 Learning Strategy

1. **Start Small**: Focus on the ~20 Tintin files, ignore the 78,000 Linux files
2. **Follow Data**: Trace how an event goes from creation → monitoring → reading
3. **Use REPRODUCE.md**: Run actual experiments to see the system in action
4. **Read Comments**: The code has good documentation, especially in key functions

Remember: You don't need to understand all 78,000 files! The core innovation is in those ~20 tintin_* files in the kernel/events directory.