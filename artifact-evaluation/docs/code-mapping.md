# Tintin Paper-to-Code Mapping

This document maps the key concepts and algorithms from the Tintin OSDI'25 paper to their actual implementations in the codebase.

## 1. Tintin-Monitor Implementation

### Uncertainty Calculation (Section 5)
- **Paper**: "Tintin-Monitor leverages measurement variance as a proxy for error"
- **Code**: `kernel/events/tintin_uncertainty.c`
  - `tintin_update_uncertainty()` (line 160-164) - Main uncertainty calculation: σ = √(variance) × unmonitored_time
  - Formula matches paper: "σᵢ = √(V(rᵢ)) × t" where t is unmonitored time

### Welford's Method for Variance Updates (Section 5)
- **Paper**: "Tintin-Monitor implements a weighted version of Welford's method for incremental variance updates"
- **Code**: `kernel/events/tintin_uncertainty.c`
  - `tintin_update_variance_by_Welfords_method()` (line 166-220)
  - Key calculations:
    - Rate calculation: `rate = count * SCALE_RATE_FOR_VARIANCE / time` (line 201)
    - Running variance: `variance_running += time * (rate - mean) * (rate - old_mean)` (line 214-216)
    - Final variance: `variance = variance_running / total_weight` (line 218)

### TAM Interpolation (Section 5)
- **Paper**: "Tintin-Monitor uses a custom implementation of the trapezoid area method (TAM)"
- **Code**: `kernel/events/tintin_estimation.c`
  - `tintin_TAM()` (line 20-51) - Main TAM implementation
  - `tintin_interpolate_count_by_TAM()` (line 72-104) - Wrapper function
  - Implements the paper's modified TAM: "compute the change in rate from the ending of the first measurement to the midpoint of the second measurement"

## 2. Tintin-Scheduler Implementation

### Elastic Scheduling Algorithm (Section 6)
- **Paper**: "formulate event scheduling problem... semantically equivalent to elastic scheduling"
- **Code**: `kernel/events/tintin_elastic.c`
  - `tintin_elastic()` (line 24-119) - Main elastic scheduling implementation
  - Key formula from paper: minimize Σ(wᵢV(rᵢ)/xᵢ²)·(1-Uᵢ)²
  - Elasticity calculation: `e = count * count / weight` (line 76)
  - Compression calculation: `compression = (u_sum - u_bound + delta) * e / e_sum` (line 105)

### Scheduling Policies
- **Round-Robin**: `tintin_scheduler.c::tintin_schedule_rr()` (line 158-176)
- **Uncertainty-First**: `tintin_scheduler.c::tintin_schedule_uncertainty_first()` (line 183-213)
- **Elastic**: Called from `tintin_elastic()` as the optimal policy

### Sorting Functions
- **Code**: `kernel/events/tintin_scheduler.c`
  - `event_uncertainty_desc()` (line 105-120) - Sort by uncertainty for scheduling
  - `event_remaining_time_desc()` (line 62-73) - Sort by elasticity for elastic scheduling

## 3. Tintin-Manager Implementation

### Event Profiling Context (ePX) (Section 7)
- **Paper**: "elevate profiling scope via a new first-class kernel object, the Event Profiling Context (ePX)"
- **Code**: `kernel/events/tintin_profiling_context.h`
  ```c
  struct tintin_profiling_context {
      int id;
      bool is_active;
      enum profiling_scope scope_type;    // SCOPE_TASK, SCOPE_CPU, SCOPE_CODE_REGION, SCOPE_ALL
      struct list_head tintin_events;     // List of events in this ePX
      unsigned num_events;
      struct rb_node rb_node;             // Red-black tree node
      raw_spinlock_t lock;
  };
  ```

### ePX Management Functions
- **Code**: `kernel/events/tintin_profiling_context.h`
  - `tintin_init_profiling_context()` - Initialize new ePX
  - `tintin_add_event_into_context()` - Add event to ePX
  - `tintin_remove_event_from_context()` - Remove event from ePX
  - `active_context()` / `deactive_context()` - Activate/deactivate ePX

## 4. System Call Implementations (Table 1 from paper)

### tintin_event_open()
- **Paper**: Extended version of perf_event_open
- **Code**: `kernel/events/tintin_syscalls.c`
  - `SYSCALL_DEFINE5(tintin_event_open, ...)` (line 169)
  - Creates tintin_event structure with uncertainty tracking

### Reading with Uncertainty
- **Paper**: "tintin_read_with_uncertainty(e) - Read both event count and uncertainty"
- **Code**: `kernel/events/tintin_interface.c`
  - `tintin_read_one()` (line 89) - Reads event count using TAM interpolation
  - `tintin_event_count()` (line 16-69) - Core counting logic with interpolation
  - `tintin_uncertainty_extrapolated()` in `tintin_estimation.c` (line 157-165)

## 5. Key Data Structures

### tintin_event (Core event structure)
- **Purpose**: Extends perf_event with uncertainty tracking
- **Key fields**:
  - `uncertainty` - Current uncertainty value
  - `Welfords_variance` - Running variance calculation
  - `Welfords_mean` - Running mean for variance
  - `total_time_monitored` / `total_time_unmonitored` - Time tracking
  - `utilization` - Assigned by elastic scheduler
  - `remaining_time` - Time left in current hyperperiod
  - `count_vec` - Vector of measurements for interpolation

### tintin_measurement
- **Purpose**: Store individual measurements for TAM interpolation
- **Fields**:
  - `count` - Event count during measurement
  - `on_HPC_time` - Time spent on hardware counter
  - `total_time_enabled` - Total time since event creation

## 6. Runtime Configuration (/proc/sys/tintin/)

### Scheduling Parameters
- `tintin_sched_interval_ms` - Hyperperiod (default: 4ms)
- `tintin_sched_quantum` - Minimum time slice (default: 400000ns)
- `tintin_sched_switch_on` - Enable/disable Tintin scheduler
- `tintin_sched_policy` - 0: round-robin, 1: elastic, 2: uncertainty-first

## 7. Key Algorithms Summary

1. **Uncertainty Calculation**: σ = √(variance) × unmonitored_time
2. **Variance Update**: Welford's method with time-weighted samples
3. **Event Interpolation**: Modified TAM using trapezoid area
4. **Event Scheduling**: Elastic scheduling minimizing Σ(uncertainty²)
5. **ePX Management**: Red-black tree for efficient context switching

This mapping demonstrates how Tintin's theoretical design translates into a practical kernel implementation, maintaining the paper's core innovations while integrating with Linux's existing perf_event infrastructure.