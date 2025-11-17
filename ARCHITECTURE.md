# OpenEvolve Architecture Documentation

## Table of Contents

1. [Introduction](#introduction)
2. [How OpenEvolve Works: The Big Picture](#how-openevolve-works-the-big-picture)
3. [Prompt Evolution: How the LLM is Guided](#prompt-evolution-how-the-llm-is-guided)
4. [The LLM Ensemble System](#the-llm-ensemble-system)
5. [MAP-Elites and Island-Based Evolution](#map-elites-and-island-based-evolution)
6. [The Evaluation System](#the-evaluation-system)
7. [Parallel Execution Architecture](#parallel-execution-architecture)
8. [Configuration and Customization](#configuration-and-customization)

---

## Introduction

OpenEvolve is an open-source implementation of evolutionary code optimization using Large Language Models. At its core, it's a system that starts with an initial program and iteratively improves it by:

1. Asking an LLM to suggest improvements
2. Evaluating the suggested changes
3. Keeping the good changes and discarding the bad ones
4. Repeating this process thousands of times

What makes OpenEvolve sophisticated is **how** it does this - using advanced evolutionary algorithms to maintain diversity, avoid getting stuck in local optima, and efficiently search the space of possible programs.

---

## How OpenEvolve Works: The Big Picture

### The Evolution Loop

Here's what happens in each iteration of OpenEvolve:

1. **Select a Parent Program**: The system chooses a program from its current population to evolve
2. **Build a Rich Prompt**: It creates a detailed prompt for the LLM that includes:
   - The current program code
   - Performance metrics from when it ran
   - Examples of other high-performing programs
   - Execution errors or outputs (if any)
   - Suggestions for what to improve
3. **Generate an Improved Version**: The LLM suggests changes (either as diffs or a complete rewrite)
4. **Evaluate the New Program**: Run it through test cases to measure performance
5. **Store in the Database**: If it's novel and good, add it to the population

This happens in parallel across multiple worker processes, with hundreds or thousands of iterations running over time.

### Key Components

**Controller (`openevolve/controller.py`)**: The main orchestrator. It sets up all the components, manages the evolution loop, handles checkpoints, and saves results.

**Database (`openevolve/database.py`)**: Stores all programs using a MAP-Elites algorithm with island-based populations. This maintains diversity - not just the best program, but many good programs that work in different ways.

**LLM Ensemble (`openevolve/llm/ensemble.py`)**: Manages multiple LLM models (like GPT-4, Claude, etc.) and selects between them with configurable weights.

**Prompt Sampler (`openevolve/prompt/sampler.py`)**: Builds rich, context-aware prompts that guide the LLM effectively.

**Evaluator (`openevolve/evaluator.py`)**: Runs programs through test cases with a smart 3-stage cascade that filters out bad programs early.

**Parallel Controller (`openevolve/process_parallel.py`)**: Distributes work across multiple CPU cores using process-based parallelism.

---

## Prompt Evolution: How the LLM is Guided

The quality of evolution depends heavily on the prompts sent to the LLM. OpenEvolve uses a sophisticated prompt construction system that provides rich context.

### What Goes Into a Prompt

Every prompt sent to the LLM includes:

#### 1. Current Program State

The prompt shows the LLM:
- The complete current program code
- Its performance metrics (accuracy, speed, etc.)
- Its "fitness" score (the overall quality metric)
- Its position in the feature space (e.g., "complexity=5, diversity=3")

This gives the LLM full context about what it's working with.

#### 2. Improvement Guidance

The system analyzes the current program and provides specific guidance:

**Fitness Tracking**: If the last change improved fitness, it says so:
```
Fitness improved: 0.7500 → 0.8234
```

If fitness declined, it warns:
```
Fitness declined: 0.8234 → 0.7500. Consider revising recent changes.
```

**Feature Exploration**: It tells the LLM what region of the solution space is being explored:
```
Exploring [complexity=5, diversity=3] region of solution space
```

**Code Quality**: If the code is getting too long:
```
Consider simplifying - code length exceeds 5000 characters
```

#### 3. Evolution History

The LLM sees what's been tried before:

**Previous Attempts** (last 3 tries):
```markdown
### Attempt 95
- Changes: Optimized loop iteration
- Performance: accuracy: 0.8234, speed: 0.6543
- Outcome: Improvement in all metrics

### Attempt 94
- Changes: Added caching layer
- Performance: accuracy: 0.7500, speed: 0.7000
- Outcome: Mixed results
```

**Top Performing Programs** (best 3-5):
```markdown
### Program 1 (Score: 0.9234)
```python
def optimized_function(data):
    # Uses dynamic programming approach
    cache = {}
    # ... implementation
```
Key features: Excellent accuracy, fast execution
```

This shows the LLM what works well.

**Inspiration Programs** (diverse approaches):

These are programs that may not be the absolute best, but represent different creative solutions. They encourage the LLM to explore new approaches rather than just tweaking the current best.

#### 4. Execution Artifacts

If the program crashed or produced errors, the LLM sees them:

```markdown
## Last Execution Output

### stderr
```
Traceback (most recent call last):
  File "program.py", line 42, in compute
    return sum(values) / len(values)
ZeroDivisionError: division by zero
```

### execution_trace
```
Step 1: Load data (SUCCESS)
Step 2: Validate input (SUCCESS)
Step 3: Compute result (FAILED - division by zero)
```
```

This helps the LLM understand what went wrong and fix it.

### Two Types of Evolution

OpenEvolve supports two evolution modes:

**Diff-Based Evolution** (default): The LLM generates specific changes in SEARCH/REPLACE format:

```
<<<<<<< SEARCH
for i in range(m):
    for j in range(p):
        result[i] += data[j]
=======
# Reorder for better cache locality
for j in range(p):
    for i in range(m):
        result[i] += data[j]
>>>>>>> REPLACE
```

This is safer - it makes targeted changes without breaking working code.

**Full Rewrite**: The LLM generates a complete new program. This is more exploratory but riskier.

### Template System

Prompts are built from templates in `openevolve/prompts/defaults/`:

- `system_message.txt`: Sets the LLM's role ("You are an expert software developer...")
- `diff_user.txt`: Template for diff-based evolution
- `full_rewrite_user.txt`: Template for complete rewrites
- `fragments.json`: Reusable text snippets for dynamic messages

You can override these with custom templates to change how the LLM is guided.

### Why This Matters

The prompt engineering is critical. By showing the LLM:
- What the current state is
- What's been tried before
- What works well
- What went wrong

...it can make much better suggestions than if it only saw the code in isolation.

---

## The LLM Ensemble System

OpenEvolve can use multiple LLM models simultaneously, choosing between them based on configurable weights.

### Why Use an Ensemble?

**Diversity**: Different models have different strengths. GPT-4 might be great at optimization, while Claude might be better at creative solutions.

**Cost Optimization**: You can mix expensive models (GPT-4) with cheaper ones (GPT-3.5-turbo):
```yaml
llm:
  models:
    - name: "gpt-4"
      weight: 0.5      # 50% of requests
    - name: "gpt-3.5-turbo"
      weight: 0.5      # 50% of requests, much cheaper
```

**Robustness**: If one API is down, others can continue working.

### How It Works

The `LLMEnsemble` class manages multiple models:

```python
class LLMEnsemble:
    def __init__(self, models_cfg: List[LLMModelConfig]):
        # Initialize all configured models
        self.models = [OpenAILLM(cfg) for cfg in models_cfg]

        # Normalize weights to sum to 1.0
        self.weights = [m.weight for m in models_cfg]
        total = sum(self.weights)
        self.weights = [w / total for w in self.weights]
```

When generating code, it selects a model randomly based on weights:

```python
def _sample_model(self) -> LLMInterface:
    # Weighted random choice
    # If weights are [0.7, 0.3], first model chosen 70% of time
    index = self.random_state.choices(
        range(len(self.models)),
        weights=self.weights,
        k=1
    )[0]
    return self.models[index]
```

The random state is seeded for reproducibility - with the same random seed, you get the same sequence of model selections.

### Retry Logic and Error Handling

Each LLM client (`OpenAILLM`) has robust error handling:

```python
async def generate_with_context(self, system_message, messages, **kwargs):
    retries = kwargs.get("retries", self.retries)  # Default: 3
    retry_delay = kwargs.get("retry_delay", self.retry_delay)  # Default: 2 seconds

    for attempt in range(retries + 1):
        try:
            response = await asyncio.wait_for(
                self._call_api(params),
                timeout=self.config.timeout  # Default: 120 seconds
            )
            return response
        except asyncio.TimeoutError:
            if attempt < retries:
                logger.warning(f"Timeout, retrying... ({attempt + 1}/{retries})")
                await asyncio.sleep(retry_delay)
            else:
                raise
        except Exception as e:
            if attempt < retries:
                logger.warning(f"Error: {e}, retrying...")
                await asyncio.sleep(retry_delay)
            else:
                raise
```

This means individual API failures don't crash the entire evolution run.

### Support for Different Model Types

OpenEvolve handles different OpenAI model types differently:

**Reasoning Models** (o1, o3, GPT-5): These don't support temperature/top_p, but support special parameters:
```python
if is_reasoning_model:
    params = {
        "model": self.model,
        "messages": messages,
        "max_completion_tokens": max_tokens,
        "reasoning_effort": "medium"  # For o1/o3
    }
```

**Standard Models** (GPT-4, Claude, etc.): Use normal parameters:
```python
params = {
    "model": self.model,
    "messages": messages,
    "temperature": 0.7,
    "top_p": 1.0,
    "max_tokens": 4096
}
```

### Configuration Example

```yaml
llm:
  # Evolution ensemble (generates code)
  models:
    - name: "gpt-4"
      weight: 0.7
      temperature: 0.7
      api_base: "https://api.openai.com/v1"

    - name: "claude-3-opus"
      weight: 0.3
      temperature: 0.7
      api_base: "https://api.anthropic.com/v1"

  # Evaluator ensemble (optional LLM feedback)
  evaluator_models:
    - name: "gpt-3.5-turbo"
      weight: 1.0

  timeout: 120
  retries: 3
```

---

## MAP-Elites and Island-Based Evolution

This is where OpenEvolve gets sophisticated. Instead of just keeping the single best program, it maintains a diverse collection of good programs using two complementary algorithms.

### MAP-Elites: Maintaining Diversity

MAP-Elites is a **quality diversity** algorithm. The idea is to divide the solution space into cells based on "feature dimensions" and keep the best solution in each cell.

**Example**: Suppose we're optimizing code with two features:
- `complexity`: How many lines of code (0-10 scale)
- `diversity`: How different from other solutions (0-10 scale)

MAP-Elites creates a 10x10 grid:

```
Diversity
   ↑
10 │ [P3]  []   [P7]  []   ...
 9 │ []    [P2] []    [P9] ...
 8 │ [P1]  []   []    []   ...
   │ ...
 1 │ [P5]  [P4] []    []   ...
   └────────────────────────→ Complexity
    1  2  3  4  ... 10
```

Each cell contains the program with the **highest fitness** that has those feature coordinates.

**Why This Matters**:
- You might have a simple, fast solution at (complexity=2, diversity=3)
- And a complex, accurate solution at (complexity=8, diversity=7)
- Both are kept because they're in different cells
- This prevents premature convergence to a single local optimum

### Feature Dimensions

Feature dimensions are metrics that describe **what** a solution is, not how **good** it is. They're specified in the evaluator:

```python
def evaluate(program_path):
    # ... run program ...

    return {
        # Fitness metrics (how good)
        "accuracy": 0.95,
        "combined_score": 0.92,

        # Feature dimensions (what type of solution)
        "complexity": 156,      # lines of code
        "diversity": 0.73,      # edit distance from others
    }
```

Configuration:
```yaml
database:
  feature_dimensions:
    - "complexity"
    - "diversity"
  feature_bins: 10  # 10x10 grid
```

The system automatically scales features to bins:
- `complexity: 156` might map to bin 7 (if 156 is in the mid-high range)
- `diversity: 0.73` might map to bin 8

### Islands: Parallel Populations

On top of MAP-Elites, OpenEvolve adds **island-based evolution**. The population is divided into multiple isolated "islands" that evolve independently.

**Example with 5 islands**:
```
Island 0: 200 programs evolving
Island 1: 200 programs evolving
Island 2: 200 programs evolving
Island 3: 200 programs evolving
Island 4: 200 programs evolving
```

Each island:
- Has its own MAP-Elites grid
- Evolves independently
- Explores different regions of the solution space

**Migration**: Periodically (every 50 generations by default), the top 10% of programs from each island migrate to the next island:

```
Island 0 ──[Top 10%]──> Island 1
Island 1 ──[Top 10%]──> Island 2
Island 2 ──[Top 10%]──> Island 3
Island 3 ──[Top 10%]──> Island 4
Island 4 ──[Top 10%]──> Island 0 (circular)
```

This shares good solutions while maintaining diversity.

### Selection Strategy

When choosing a parent program to evolve, OpenEvolve uses a mixed strategy:

- **Exploration** (20%): Random selection from the island
- **Exploitation** (70%): Select from the elite archive (best programs)
- **Fitness-Weighted** (10%): Weighted random based on fitness scores

This balances between improving good solutions and exploring new areas.

### Novelty Detection

To prevent wasting time on duplicates, OpenEvolve can check if a new program is "novel" before adding it:

**Embedding-Based**: Compare code embeddings using cosine similarity
```yaml
database:
  embedding_model: "text-embedding-3-small"
  similarity_threshold: 0.99  # Reject if >99% similar
```

**LLM-Based**: Ask an LLM "Is this meaningfully different from existing programs?"
```yaml
database:
  novelty_llm: true
```

Programs that are too similar to existing ones are rejected.

### Why Islands + MAP-Elites?

**MAP-Elites alone** ensures diversity across feature dimensions within a single population.

**Islands** add another layer of diversity by maintaining separate populations that explore different strategies.

**Together** they prevent premature convergence and explore the solution space much more thoroughly than a simple "keep the best" approach.

---

## The Evaluation System

The evaluator is responsible for running programs and measuring their performance. OpenEvolve uses a smart cascading evaluation system to save time.

### Cascade Evaluation: The Three-Stage Filter

Most evolutionary systems evaluate every program fully. This wastes time - if a program fails basic tests, why run expensive comprehensive tests?

OpenEvolve uses a **three-stage cascade**:

#### Stage 1: Quick Validation (~0.1 seconds)

Basic smoke test:
- Does it run without crashing?
- Does it produce any output?
- Does it pass trivial test cases?

**Threshold**: 0.5
- If score < 0.5: **STOP**, don't run further stages
- If score ≥ 0.5: Continue to stage 2

#### Stage 2: Moderate Testing (~1 second)

Test on small datasets:
- Run on representative test cases
- Check correctness
- Basic performance measurement

**Threshold**: 0.75
- If score < 0.75: **STOP**, return stage 2 score
- If score ≥ 0.75: Continue to stage 3

#### Stage 3: Comprehensive Evaluation (~10 seconds)

Full test suite:
- Large datasets
- Edge cases
- Performance benchmarks
- Stress tests

**Result**: Final comprehensive score

### Why This Works

**Time Savings**: If only 20% of programs pass all stages, you save 80% of evaluation time:
- 100% evaluated in stage 1 (cheap)
- ~50% evaluated in stage 2 (moderate cost)
- ~20% evaluated in stage 3 (expensive)

**Early Termination**: Bad programs are filtered out quickly without wasting resources.

### User Evaluator Example

Users implement cascade stages in their evaluator:

```python
def evaluate_stage1(program_path: str) -> dict:
    """Quick validation - should run in <0.1 seconds"""
    spec = importlib.util.spec_from_file_location("program", program_path)
    program = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(program)

    # Quick smoke test
    try:
        result = program.solve([1, 2, 3])
        score = 1.0 if result is not None else 0.0
    except Exception:
        score = 0.0

    return {"stage1_passed": score, "combined_score": score}


def evaluate_stage2(program_path: str) -> dict:
    """Moderate testing - should run in ~1 second"""
    spec = importlib.util.spec_from_file_location("program", program_path)
    program = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(program)

    # Test on 10 representative cases
    test_cases = load_moderate_test_set()
    correct = sum(1 for tc in test_cases if program.solve(tc.input) == tc.expected)
    score = correct / len(test_cases)

    return {"stage2_passed": score, "combined_score": score}


def evaluate_stage3(program_path: str) -> dict:
    """Comprehensive - can take ~10 seconds"""
    spec = importlib.util.spec_from_file_location("program", program_path)
    program = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(program)

    # Full test suite: 1000+ cases
    test_cases = load_comprehensive_test_set()

    correct = 0
    total_time = 0
    for tc in test_cases:
        start = time.time()
        result = program.solve(tc.input)
        total_time += time.time() - start

        if result == tc.expected:
            correct += 1

    accuracy = correct / len(test_cases)
    speed_score = 1.0 / (1.0 + total_time)

    return {
        "accuracy": accuracy,
        "speed_score": speed_score,
        "combined_score": 0.7 * accuracy + 0.3 * speed_score,
        "stage3_passed": 1.0 if accuracy > 0.9 else 0.0
    }


# Fallback if cascade disabled
def evaluate(program_path: str) -> dict:
    return evaluate_stage3(program_path)
```

Configuration:
```yaml
evaluator:
  cascade_evaluation: true
  cascade_thresholds: [0.5, 0.75, 0.9]
  timeout: 300
  max_retries: 3
```

### The Artifact System

Programs can return not just metrics, but also **artifacts** - execution outputs, error messages, debug traces, etc.

```python
from openevolve.evaluation_result import EvaluationResult

def evaluate(program_path):
    # ... run program ...

    artifacts = {
        "stdout": "Processing 1000 items...\nComplete!",
        "stderr": "Warning: deprecated function used",
        "execution_trace": "Step 1: Load\nStep 2: Process\nStep 3: Output"
    }

    return EvaluationResult(
        metrics={
            "accuracy": 0.95,
            "combined_score": 0.92
        },
        artifacts=artifacts
    )
```

These artifacts are included in the prompt to the LLM, helping it understand what happened during execution.

**Small artifacts** (<32KB) are stored in the database as JSON.
**Large artifacts** (>32KB) are saved as separate files.

---

## Parallel Execution Architecture

OpenEvolve uses **process-based parallelism** to run multiple evolution iterations simultaneously across CPU cores.

### Why Process-Based, Not Threading?

Python's Global Interpreter Lock (GIL) prevents true parallel execution with threads. Using processes instead:
- Each process has its own Python interpreter
- True parallelism across CPU cores
- Linear speedup (4 cores = 4x faster)

### The Worker Pool Architecture

```
Main Process (Controller)
    │
    ├─ Creates ProcessPoolExecutor with N workers
    │
    ├─ For each iteration:
    │   │
    │   ├─ Create database snapshot (serialize all programs)
    │   ├─ Sample parent + inspirations for each worker
    │   ├─ Submit tasks to worker pool
    │   │
    │   └─ Workers execute in parallel:
    │       ├─ Worker 1: Evolve program A
    │       ├─ Worker 2: Evolve program B
    │       ├─ Worker 3: Evolve program C
    │       └─ Worker 4: Evolve program D
    │
    └─ Aggregate results back to main database
```

### Database Snapshot

Since workers run in separate processes, they can't share memory directly. Instead, the main process creates a **snapshot** of the database:

```python
def _create_database_snapshot(self) -> dict:
    return {
        "programs": {
            pid: prog.to_dict()
            for pid, prog in self.database.programs.items()
        },
        "islands": [list(island) for island in self.database.islands],
        "feature_dimensions": self.database.config.feature_dimensions,
        "artifacts": {...}  # Limited to recent 100 programs for size
    }
```

This snapshot (typically 1-5MB) is pickled and sent to each worker.

### Worker Execution

Each worker process:

1. **Deserializes the snapshot** to reconstruct programs
2. **Builds a prompt** using the prompt sampler
3. **Generates code** via the LLM
4. **Evaluates** the new program
5. **Returns a serializable result** back to main process

Workers are initialized lazily - expensive components (LLM clients, evaluators) are only created when first needed.

### Result Aggregation

The main process collects results from all workers:

```python
for result in results:
    if result and not result.error:
        # Reconstruct program from dict
        child_program = Program(**result.child_program_dict)

        # Add to database (with novelty check, MAP-Elites placement, etc.)
        self.database.add(child_program, iteration=result.iteration)

        # Store artifacts
        if result.artifacts:
            self.database.store_artifacts(child_program.id, result.artifacts)
```

### Island-Worker Mapping

Workers are assigned to islands in round-robin fashion:

```
Worker 0 → Island 0
Worker 1 → Island 1
Worker 2 → Island 2
Worker 3 → Island 3
Worker 4 → Island 0  (wraps around)
```

This ensures each island is actively evolving in parallel.

### Configuration

```yaml
evaluator:
  parallel_evaluations: 4  # Number of worker processes

database:
  num_islands: 5  # Should be >= parallel_evaluations for efficiency
```

**Best practice**: Set `num_islands >= parallel_evaluations` so each worker has its own island.

---

## Configuration and Customization

OpenEvolve is highly configurable through YAML files. Here's a comprehensive guide.

### Basic Configuration

Minimal config (uses mostly defaults):

```yaml
llm:
  models:
    - name: "gpt-4"
      weight: 1.0

max_iterations: 100
```

### Complete Configuration Structure

```yaml
# === General Settings ===
max_iterations: 1000           # Total iterations to run
checkpoint_interval: 50        # Save checkpoint every N iterations
log_level: "INFO"              # DEBUG, INFO, WARNING, ERROR
random_seed: 42                # For reproducibility

# === Evolution Settings ===
diff_based_evolution: true     # true = diffs, false = full rewrites
max_code_length: 10000         # Reject programs longer than this
language: "python"             # python, rust, r, javascript
file_suffix: ".py"             # File extension for temporary files

# === LLM Configuration ===
llm:
  # Evolution models (generate code)
  models:
    - name: "gpt-4"
      weight: 0.7
      temperature: 0.7
      max_tokens: 4096
      timeout: 120
      retries: 3
      api_base: "https://api.openai.com/v1"
      api_key: ${OPENAI_API_KEY}  # From environment

    - name: "claude-3-opus"
      weight: 0.3
      temperature: 0.7
      api_base: "https://api.anthropic.com/v1"
      api_key: ${ANTHROPIC_API_KEY}

  # Evaluator models (optional LLM feedback)
  evaluator_models:
    - name: "gpt-3.5-turbo"
      weight: 1.0

# === Prompt Configuration ===
prompt:
  template_dir: null                    # null = use defaults, or path to custom templates
  num_top_programs: 3                   # Show N best programs to LLM
  num_diverse_programs: 2               # Show N diverse programs to LLM
  use_template_stochasticity: false     # Randomize phrasing for diversity
  include_artifacts: true               # Include execution outputs in prompt
  max_artifact_bytes: 20480             # 20KB max per artifact
  artifact_security_filter: true        # Redact API keys, passwords, etc.

# === Database Configuration (MAP-Elites + Islands) ===
database:
  in_memory: true                       # true = RAM only, false = persist to disk
  population_size: 1000                 # Max programs to keep
  archive_size: 100                     # Size of elite archive

  # Island settings
  num_islands: 5                        # Number of parallel populations
  migration_interval: 50                # Migrate every N generations
  migration_rate: 0.1                   # Migrate top 10%

  # Feature dimensions for MAP-Elites
  feature_dimensions:
    - "complexity"                      # Must be in evaluator metrics
    - "diversity"                       # Must be in evaluator metrics

  feature_bins: 10                      # 10x10 grid (or per-dimension below)
  # OR per-dimension:
  # feature_bins:
  #   complexity: 15
  #   diversity: 20

  # Selection ratios (must sum to 1.0)
  elite_selection_ratio: 0.1            # Archive = top 10%
  exploration_ratio: 0.2                # 20% random selection
  exploitation_ratio: 0.7               # 70% elite selection

  # Novelty detection
  embedding_model: "text-embedding-3-small"  # OpenAI embedding model
  similarity_threshold: 0.99            # Reject if >99% similar
  # OR:
  # novelty_llm: true                   # Use LLM for novelty checking

# === Evaluator Configuration ===
evaluator:
  timeout: 300                          # Seconds per stage
  max_retries: 3                        # Retry failed evaluations

  cascade_evaluation: true              # Enable 3-stage cascade
  cascade_thresholds: [0.5, 0.75, 0.9]  # Thresholds for stages 1, 2, 3

  parallel_evaluations: 4               # Number of worker processes

  use_llm_feedback: false               # Optional LLM code review
  llm_feedback_weight: 0.3              # Weight for LLM scores

# === Evolution Tracing (for analysis/RL training) ===
evolution_trace:
  enabled: false                        # Enable trace logging
  format: "jsonl"                       # jsonl, json, or hdf5
  output_path: "evolution_trace.jsonl" # Where to save
  include_code: false                   # Include full code (large files!)
  include_prompts: true                 # Include prompts sent to LLM
  buffer_size: 100                      # Buffer N traces before writing
  compress: false                       # gzip compression

# === Early Stopping ===
early_stopping_patience: null           # null = disabled, or N iterations
convergence_threshold: 0.001            # Stop if improvement < this
early_stopping_metric: "combined_score" # Metric to monitor
```

### Environment Variables

Use `${VAR_NAME}` syntax to reference environment variables:

```yaml
llm:
  api_key: ${OPENAI_API_KEY}
  api_base: ${OPENAI_API_BASE:-https://api.openai.com/v1}  # With default
```

### Custom Templates

Create a directory with custom prompt templates:

```
my_templates/
├── system_message.txt
├── diff_user.txt
└── fragments.json
```

Then reference it:
```yaml
prompt:
  template_dir: "./my_templates"
```

### Language-Specific Configuration

For Rust programs:
```yaml
language: "rust"
file_suffix: ".rs"
```

For R programs:
```yaml
language: "r"
file_suffix: ".R"
```

### Reproducibility

Set a random seed for fully deterministic runs:
```yaml
random_seed: 42
```

This seeds:
- Python's `random` module
- NumPy's random generator
- Database sampling
- LLM model selection
- OpenAI's seed parameter (where supported)

**Note**: LLM responses may still vary slightly due to model non-determinism.

---

## Practical Advice

### Starting Configuration

For your first run, start simple:

```yaml
llm:
  models:
    - name: "gpt-4"
      weight: 1.0

max_iterations: 100
checkpoint_interval: 10

database:
  num_islands: 3
  feature_dimensions: []  # No MAP-Elites initially

evaluator:
  parallel_evaluations: 2
  cascade_evaluation: false  # Start without cascade
```

Then add complexity:
1. Add feature dimensions and MAP-Elites
2. Enable cascade evaluation
3. Add more islands
4. Add model ensemble
5. Tune selection ratios

### Debugging

Enable debug logging:
```yaml
log_level: "DEBUG"
```

Reduce parallelism to 1 for easier debugging:
```yaml
evaluator:
  parallel_evaluations: 1
```

Enable evolution tracing:
```yaml
evolution_trace:
  enabled: true
  include_code: true
  include_prompts: true
```

### Performance Tuning

**More parallelism**:
```yaml
evaluator:
  parallel_evaluations: 8  # Use more cores

database:
  num_islands: 8  # Match or exceed parallel_evaluations
```

**Faster evaluation**:
```yaml
evaluator:
  cascade_evaluation: true
  cascade_thresholds: [0.3, 0.6, 0.9]  # Stricter early filtering
```

**Better exploration**:
```yaml
database:
  exploration_ratio: 0.3  # More random exploration
  exploitation_ratio: 0.6
  num_islands: 10  # More parallel populations
```

---

## Summary

OpenEvolve combines several sophisticated techniques:

1. **Rich Prompt Engineering**: Guides the LLM with comprehensive context
2. **LLM Ensemble**: Uses multiple models with weighted selection
3. **MAP-Elites + Islands**: Maintains diverse populations across feature dimensions
4. **Cascade Evaluation**: Efficiently filters bad programs early
5. **Process Parallelism**: True parallel execution across CPU cores
6. **Novelty Detection**: Avoids wasting time on duplicates
7. **Checkpointing**: Robust resume after interruptions

The result is a system that can evolve code effectively for complex optimization problems, discovering creative solutions that might not be found by traditional optimization approaches.

For more details, see the source code and examples in the repository.
