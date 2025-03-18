# Philosophers Project: Problem Understanding & Design Plan

## Core Concepts

The Dining Philosophers problem represents fundamental challenges in concurrent programming:

- **Resource Allocation**: Multiple actors need shared, limited resources
- **Deadlock Prevention**: Ensuring the system never reaches a state where progress is impossible
- **Race Condition Management**: Preventing unpredictable behavior when multiple threads access shared data
- **Starvation Avoidance**: Ensuring every philosopher gets a chance to eat

## Project Requirements Analysis

### Philosophers Simulation

- Each philosopher (numbered 1 to N) alternates between eating, sleeping, and thinking
- Philosophers share forks (one between each pair)
- A philosopher needs two forks to eat
- A philosopher dies if they don't start eating within `time_to_die` milliseconds of their last meal
- Death must be reported within 10ms of occurrence
- Simulation can end when:
  - A philosopher dies
  - All philosophers have eaten `number_of_times_each_philosopher_must_eat` times (optional)


### Technical Requirements

- **Mandatory Part**:
  - Each philosopher is a thread
  - One fork between each philosopher (protected by a mutex)
  - No global variables (except constants)
  - Death detection must be race-condition free

- **Bonus Part** (optional):
  - Each philosopher is a process
  - Forks represented by a semaphore
  - Main process isn't a philosopher



## Design Considerations

### Data Structures

**Philosopher Structure**:
   
typedef struct s_philo
{
	int             id;
	long long       last_meal_time;
	int             meals_eaten;
	pthread_t       thread;
	pthread_mutex_t *left_fork;
	pthread_mutex_t *right_fork;
	// Additional fields for simulation state
} t_philo;


**Simulation Data**:

typedef struct s_data
{
	int             num_philosophers;
	int             time_to_die;
	int             time_to_eat;
	int             time_to_sleep;
	int             must_eat_count;
	long long       start_time;
	int             simulation_end;
	pthread_mutex_t *forks;
	pthread_mutex_t write_mutex;  // For non-overlapping output
	pthread_mutex_t death_mutex;  // For checking death condition
	t_philo         *philosophers;
} t_data;




### Critical Sections

1. **Fork Access**: Mutexes to ensure exclusive access to each fork
2. **Death Checking**: Mutex to protect last_meal_time access and updates
3. **Output Handling**: Mutex to prevent interleaved log messages
4. **Simulation State**: Mutex to protect the simulation_end flag



### Algorithms

1. **Deadlock Prevention Strategy**:
   - Even-numbered philosophers take right fork first, odd take left fork first
   - OR: Resource hierarchy (all philosophers take lower-numbered fork first)

2. **Philosopher Lifecycle**:

WHILE simulation is running:
	1. Take forks (with deadlock prevention strategy)
	2. Eat (update last_meal_time with mutex protection)
	3. Release forks
	4. Sleep
	5. Think
END WHILE


3. **Death Monitoring**:
   - Separate monitoring thread or routine
   - Periodically check if current_time - last_meal_time > time_to_die
   - Use mutex to protect this check

### Timing Considerations

- Use `gettimeofday()` for precise millisecond timing
- Convert to milliseconds: `(tv_sec * 1000) + (tv_usec / 1000)`
- Consider implementing a custom `ft_usleep()` that checks conditions while sleeping



## Potential Challenges

1. **Race Conditions**:
   - Protecting last_meal_time updates/reads
   - Ensuring death is detected accurately

2. **Deadlocks**:
   - Preventing circular wait conditions
   - Careful fork acquisition order

3. **Timing Precision**:
   - Ensuring death detection within 10ms
   - Accounting for system scheduling delays

4. **Resource Management**:
   - Proper initialization and cleanup of mutexes
   - Handling thread creation failures



## Testing Plan

1. **Basic Functionality**:
   - Test with 5 philosophers, ample time (no deaths expected)
   - Test with mandatory meals count to ensure proper termination

2. **Edge Cases**:
   - Single philosopher (should die unable to eat)
   - Large number of philosophers (test scalability)

3. **Death Scenarios**:
   - Tight timing to ensure deaths occur and are detected correctly
   - Verify death reporting is within 10ms

4. **Resource Cleanup**:
   - Ensure all threads terminate properly
   - Verify all mutexes are properly destroyed



## Implementation Phases

1. **Setup Phase**:
   - Parse arguments
   - Initialize data structures
   - Create mutexes

2. **Core Simulation**:
   - Create philosopher threads
   - Implement philosopher lifecycle
   - Implement death monitoring

3. **Cleanup & Termination**:
   - Handle simulation end conditions
   - Clean up resources
   - Join threads

4. **Bonus Part** (if applicable):
   - Adapt for processes instead of threads
   - Implement semaphore-based fork management