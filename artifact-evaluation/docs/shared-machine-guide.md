# Running Tintin on a Shared Machine - Safety & Impact Guide

## ⚠️ What Will Be Impacted

### 1. **Kernel Module Loading** (HIGH IMPACT)
- **What**: Tintin requires loading a modified Linux kernel (v5.15)
- **Impact**: This affects the ENTIRE SYSTEM, not just your user space
- **Risk**: Kernel modifications can crash the system if buggy
- **Requirement**: Root/sudo access needed

### 2. **Hardware Performance Counters** (MEDIUM IMPACT)
- **What**: Tintin monopolizes CPU performance counters
- **Impact**: Other profiling tools (perf, Intel VTune, etc.) won't work properly
- **Scope**: System-wide - affects ALL users
- **Duration**: While Tintin is active

### 3. **CPU Scheduling** (LOW-MEDIUM IMPACT)
- **What**: Tintin adds overhead to context switches (4ms hyperperiod)
- **Impact**: ~2-5% performance overhead for all processes
- **Scope**: System-wide scheduling latency increase

### 4. **System Memory** (LOW IMPACT)
- **What**: Kernel data structures for event tracking
- **Impact**: ~100KB per monitored event
- **Scope**: Kernel memory (not swappable)

## 🛡️ Safe Experimentation Strategy

### Option 1: Use a Test VM (RECOMMENDED)
```bash
# Create an isolated VM with the Tintin kernel
# This way you can't impact the shared machine
```

### Option 2: Coordinate with Other Users
```bash
# 1. Check who's using the machine
who
w

# 2. Notify users before loading Tintin kernel
wall "Loading Tintin kernel module for experiments at 3pm - performance profiling will be unavailable"

# 3. Set a time limit for your experiments
```

### Option 3: Use Existing Trace Data
```bash
# The paper authors might have provided trace data
cd ~/playground/profiling-gym/tintin/paper-source/tintin-user/plots
ls *.csv *.log  # Look for existing experimental data
```

## 📋 Pre-Flight Checklist

### 1. Check Current System State
```bash
# Check if performance events are already in use
sudo perf list
ps aux | grep perf

# Check current kernel version
uname -r

# Check available HPCs
ls /sys/devices/cpu/events/
```

### 2. Backup Current Configuration
```bash
# Save current kernel parameters
sudo sysctl -a > ~/kernel_params_backup.txt

# Note current kernel version for rollback
echo "Current kernel: $(uname -r)" > ~/tintin_experiment_log.txt
```

## 🚀 Minimal Impact Experiment Steps

### Step 1: Build Without Installing (SAFE)
```bash
cd ~/playground/profiling-gym/tintin/paper-source/tintin-kernel

# Just compile, don't install
make -j$(nproc) 
# This only uses CPU, doesn't affect system
```

### Step 2: Test in User Space First (SAFE)
```bash
cd ~/playground/profiling-gym/tintin/paper-source/tintin-user

# Run synthetic tests that don't need kernel
./tests/synthetic_test --no-kernel

# Analyze existing data
python plots/figure-11-uncertainty.py
```

### Step 3: Coordinate Kernel Loading (RISKY)
```bash
# ONLY do this with permission and coordination!

# 1. Schedule a maintenance window
# 2. Have rollback plan ready
# 3. Monitor system closely

# Load Tintin kernel module
sudo insmod tintin.ko

# Enable Tintin scheduling
echo 1 | sudo tee /proc/sys/tintin/tintin_sched_switch_on

# Run quick test
./tests/quick_validation.sh

# IMMEDIATELY disable if issues
echo 0 | sudo tee /proc/sys/tintin/tintin_sched_switch_on
sudo rmmod tintin
```

## 📊 What You CAN Do Without Root

### 1. Study the Implementation
```bash
# Read and understand code (no impact)
cd ~/playground/profiling-gym/tintin/paper-source/tintin-kernel/kernel/events
grep -n "tintin_elastic" *.c

# Generate call graphs
cscope -b -R
```

### 2. Simulate Algorithms
```python
# Create Python simulations of Tintin algorithms
# tintin_simulation.py
import numpy as np

def welford_variance(samples):
    """Simulate Welford's method without kernel"""
    mean = 0.0
    M2 = 0.0
    for i, x in enumerate(samples):
        delta = x - mean
        mean += delta / (i + 1)
        M2 += delta * (x - mean)
    return M2 / len(samples)

def elastic_scheduling(events, m, hyperperiod):
    """Simulate elastic scheduling algorithm"""
    # Your simulation code here
    pass
```

### 3. Analyze Paper Results
```bash
cd ~/playground/profiling-gym/tintin/artifact-evaluation

# Create analysis notebooks
jupyter notebook tintin_paper_analysis.ipynb

# Reproduce figures from paper data
python reproduce_paper_figures.py
```

## 🔍 Monitoring Impact During Experiments

If you do get permission to run experiments:

```bash
# Terminal 1: Monitor system load
htop

# Terminal 2: Monitor kernel messages
sudo dmesg -w | grep -i tintin

# Terminal 3: Monitor performance counters
watch -n 1 'sudo perf stat -a sleep 0.1 2>&1'

# Terminal 4: Run your experiment
./run_experiment.sh
```

## 🚨 Emergency Rollback

If something goes wrong:

```bash
# 1. Disable Tintin immediately
echo 0 | sudo tee /proc/sys/tintin/tintin_sched_switch_on

# 2. Unload module
sudo rmmod tintin

# 3. Check system stability
dmesg | tail -50

# 4. Reboot if necessary (coordinate with users!)
# sudo reboot
```

## 💡 Alternative Approaches

### 1. Request a Dedicated Test Machine
- Ask your advisor/IT for a test server
- Use cloud instances (AWS/GCP) with saved snapshots

### 2. Use Tintin's Simulation Mode
```bash
# Some research kernels have simulation modes
make CONFIG_TINTIN_SIMULATION=y
```

### 3. Focus on Algorithm Analysis
- Implement Tintin algorithms in user space
- Use existing perf data with Tintin's analysis methods
- Create theoretical models without kernel changes

## 📚 Learning Without Risk

### Safe Activities:
1. **Code Review**: Understand the implementation deeply
2. **Algorithm Study**: Implement elastic scheduling in Python
3. **Data Analysis**: Use provided experimental data
4. **Documentation**: Create better docs for the project
5. **Testing Framework**: Build better tests (without running)

### Example Safe Study Session:
```bash
# Session 1: Understand uncertainty calculation
cd ~/playground/profiling-gym/tintin
grep -r "Welford" --include="*.c" 
# Take notes in your own documentation

# Session 2: Trace scheduling decisions
# Create sequence diagrams of the scheduling flow

# Session 3: Analyze paper's experimental methodology
# Plan improvements without running code
```

## 🤝 Best Practices for Shared Machines

1. **Always Communicate**: Email other users before system-wide changes
2. **Time-box Experiments**: Set hard limits (e.g., 2-hour windows)
3. **Monitor Actively**: Watch for impact while running
4. **Document Everything**: Keep logs of what you did and when
5. **Have Rollback Plan**: Know how to quickly undo changes

## 📧 Sample Communication Template

```
Subject: Tintin Kernel Experiments - Scheduling Request

Hi all,

I need to run some kernel-level performance monitoring experiments using Tintin (OSDI'25 paper) on [machine name].

Impact:
- Performance monitoring tools (perf, VTune) will be unavailable
- ~2-5% system-wide performance overhead
- Requires loading custom kernel module

Proposed Time: [Date] from [start] to [end] (2 hours max)

I will:
1. Monitor system stability throughout
2. Immediately rollback if any issues
3. Restore original configuration when done

Please let me know if this time doesn't work for you.

Thanks,
[Your name]
```

Remember: On a shared machine, it's better to do thorough preparation and simulation first, then run minimal real experiments with coordination. The learning value is in understanding the system, not necessarily running it at scale.