# MiniRuntime --- Project Handbook

**Version:** 0.1\
**Last updated:** 16 October 2026\
**Purpose:** A single source of truth for the project goal,
architecture, technology stack, development roadmap, setup, tasks,
tests, experiments, and resume procedure.

------------------------------------------------------------------------

## 1. Project overview

### What is MiniRuntime?

MiniRuntime is an educational, CPU-first LLM inference runtime. The goal
is to understand how a pretrained language model runs---not to
outperform mature runtimes such as llama.cpp, Ollama, or vLLM.

The project will progressively cover:

-   How text becomes token IDs and how a transformer produces logits.
-   How autoregressive generation selects one token at a time.
-   The difference between prompt prefill and token-by-token decoding.
-   KV caching and its effects on computation, memory, and latency.
-   CPU, RAM, memory bandwidth, threading, and process-level
    measurements.
-   Separating an inference runtime from an HTTP serving layer.
-   Integrating a native C++ component with Python and benchmarking it.
-   Testing, profiling, documentation, and reproducible engineering
    experiments.

**Guiding principle:** every milestone should leave behind working
software, tests, measurements, and an explanation of what we learned.

### Project boundaries

The initial version will not:

-   Train a language model from scratch.
-   Support every transformer architecture.
-   Compete with production inference engines.
-   Reimplement every tensor kernel before we have a working baseline.
-   Claim that a library-managed feature is our own implementation.

We will use existing libraries where useful, while progressively taking
ownership of the generation loop, runtime orchestration,
instrumentation, and selected lower-level components.

### Initial target

Start with **GPT-2 Small**, a relatively small decoder-only model. Use
CPU execution first. Model weights will be downloaded separately from
Python dependencies.

The initial model library will still perform the transformer forward
pass. Later, we can implement more of the forward pass ourselves and
compare the result against the reference implementation.

------------------------------------------------------------------------

## 2. Definition of success

The first useful version is complete when:

-   A fresh environment can install the project and run its tests.
-   GPT-2 loads and runs on CPU.
-   Our own generation loop works without calling `model.generate()`.
-   The code and documentation explain token IDs, logits, greedy
    selection, and stopping conditions.
-   Cached and uncached generation can be compared for correctness.
-   A benchmark records prompt length, generated token count, elapsed
    time, tokens/second, and process memory.
-   A FastAPI service can expose the runtime without containing the core
    generation logic.
-   At least one small C++ component can be called from Python and
    tested.
-   The README documents setup, architecture, limitations, and
    repeatable experiments.

These are learning and correctness goals, not performance guarantees.

------------------------------------------------------------------------

## 3. Technology stack

  -----------------------------------------------------------------------
  Technology              Role                    When introduced
  ----------------------- ----------------------- -----------------------
  Python 3.12             Main implementation and Phase 0
                          orchestration language  

  `uv`                    Python version,         Phase 0
                          dependency, and         
                          environment management  

  PyTorch                 Tensor operations and   Phase 0
                          initial model execution 

  Hugging Face            Initial GPT-2 loading   Phase 0
  Transformers            and reference behavior  

  Hugging Face Tokenizers Text/token conversion   Phase 0
  (through Transformers)                          

  `safetensors`           Support for a common    Phase 0
                          model-weight format     

  `psutil`                Process-level CPU and   Phase 0
                          memory observations     

  `pytest`                Automated tests         Phase 0

  Ruff                    Linting and basic code  Phase 0
                          quality                 

  FastAPI                 HTTP serving layer      Phase 4

  Uvicorn                 Run the API server      Phase 4

  C++ compiler/toolchain  Native experiments and  Phase 5
                          performance-sensitive   
                          code                    

  `pybind11`              Bridge native C++ code  Phase 5
                          to Python               

  Git                     Version control and     Phase 0
                          milestone history       
  -----------------------------------------------------------------------

### Language strategy

**Python first, then C++.**

Python helps us move quickly while learning model behavior, generation,
caching, tests, metrics, and API design. C++ is introduced once we have
a correct, measurable baseline and a clear reason to investigate
lower-level execution.

Using Python does not prevent OS and computer-architecture learning. We
can observe process memory, CPU utilization, thread count, timing, and
performance changes. C++ later gives us more direct control over
allocations, memory layout, native loops, and threading. Neither
language automatically explains CPU caches or operating-system behavior;
we must design experiments and interpret measurements.

### Initial dependencies

Runtime dependencies:

-   `torch`
-   `transformers`
-   `safetensors`
-   `psutil`

Development dependencies:

-   `pytest`
-   `ruff`

Later, when building the API, add `fastapi` and `uvicorn`. Add
native-build dependencies only when Phase 5 begins.

------------------------------------------------------------------------

## 4. High-level architecture

``` text
Client / CLI
    |
    +------------------------------+
    |                              |
    v                              v
CLI entry point              FastAPI serving layer (later)
    |                              |
    +--------------+---------------+
                   |
                   v
             Inference Runtime
     - generation lifecycle
     - sampling and stopping
     - cache lifecycle
     - metrics and limits
                   |
                   v
             Model Execution
     - tokenizer
     - model configuration and weights
     - transformer forward pass
     - logits
                   |
                   v
             Execution Hardware
     - CPU
     - RAM
     - CPU caches
     - memory bandwidth
     - threads
```

Later, selected operations may use a native backend:

``` text
Python runtime
    |
    v
Python/C++ interface (e.g. pybind11)
    |
    v
Native C++ component
    |
    v
CPU execution
```

### Layer responsibilities

-   **CLI:** simplest interface for experiments; build this before an
    API.
-   **Serving layer:** translates HTTP requests into runtime calls. It
    should not contain model-generation logic.
-   **Runtime:** coordinates generation, sampling, stopping rules, cache
    lifecycle, and metrics.
-   **Model execution:** initially mostly handled by
    PyTorch/Transformers; progressively inspect or replace parts.
-   **Native backend:** a later experiment for selected operations, not
    a rewrite of the whole system by default.

### What happens during inference?

1.  The tokenizer converts text into integer token IDs.
2.  The transformer processes the input IDs.
3.  The final output contains logits---scores for possible next tokens.
4.  The runtime selects a token (initially by greedy `argmax`).
5.  The new token is appended and generation repeats until a stop
    condition or token limit.
6.  The tokenizer converts generated token IDs back into text.

**Prefill** processes the initial prompt. **Decode** produces subsequent
tokens one at a time. We will measure these separately where practical.

------------------------------------------------------------------------

## 5. Planned repository structure

Create files as needed; do not create empty modules just to match the
diagram.

``` text
MiniRuntime/
├── pyproject.toml
├── uv.lock
├── .python-version
├── .gitignore
├── README.md
├── src/
│   └── mini_runtime/
│       ├── __init__.py
│       ├── model_loader.py
│       ├── tokenizer.py
│       ├── generation.py
│       ├── sampling.py
│       ├── cache.py
│       ├── runtime.py
│       ├── metrics.py
│       └── api.py                 # introduced in Phase 4
├── tests/
│   ├── test_model_loader.py
│   ├── test_generation.py
│   ├── test_cache.py
│   └── test_metrics.py
├── experiments/
│   ├── inspect_model.py
│   └── benchmark.py
├── native/
│   └── README.md                  # introduced in Phase 5
└── docs/
    ├── architecture.md
    ├── experiments.md
    └── progress-log.md
```

### Module responsibilities

  Module              Responsibility
  ------------------- -------------------------------------------------------
  `model_loader.py`   Load the model and configuration
  `tokenizer.py`      Convert text and token IDs
  `generation.py`     Own the token-by-token generation loop
  `sampling.py`       Greedy selection first; optional sampling later
  `cache.py`          Cache-related interfaces, lifecycle, and measurements
  `runtime.py`        Coordinate model, generation, limits, and metrics
  `metrics.py`        Latency, token throughput, and memory measurements
  `api.py`            HTTP endpoints, added only after the runtime works

At first, `cache.py` will document and use the model library's cache
behavior. Do not describe that as a fully self-implemented KV cache.
Later milestones can take ownership of more of the cache implementation.

------------------------------------------------------------------------

## 6. Development environment setup (Windows / PowerShell)

Assumption: development happens on a Windows PC. Use a new project
directory and do not reuse another project's virtual environment.

### 6.1 Check prerequisites

``` powershell
python --version
uv --version
git --version
```

Target Python version: **3.12**. If `uv` is not installed or any command
fails, resolve that before proceeding.

### 6.2 Create the project

If `D:\MiniRuntime` does not already contain project files:

``` powershell
mkdir D:\MiniRuntime
cd D:\MiniRuntime

uv python install 3.12
uv init --package mini-runtime
uv python pin 3.12
```

If the directory already exists, inspect its contents first rather than
blindly initializing over it.

### 6.3 Install initial dependencies

``` powershell
uv add torch transformers safetensors psutil
uv add --dev pytest ruff
```

The first PyTorch installation can take time and disk space. We start
with CPU inference; a GPU is not required.

### 6.4 Verify dependencies

``` powershell
uv run python -c "import torch, transformers, psutil; print('PyTorch:', torch.__version__); print('Transformers:', transformers.__version__); print('CUDA available:', torch.cuda.is_available())"

uv run pytest --version
uv run ruff --version
```

CUDA availability is informational; it is not a requirement.

### 6.5 Initialize Git

After the environment is verified:

``` powershell
git init
git add .
git commit -m "chore: initialize MiniRuntime project"
```

If Git requests an author identity, configure your Git name/email first.
Do not commit secrets, virtual environments, or downloaded model
weights.

### 6.6 Suggested `.gitignore`

``` gitignore
.venv/
__pycache__/
*.py[cod]
.pytest_cache/
.ruff_cache/
.env
.env.*
!.env.example
models/
*.safetensors
*.bin
*.pt
*.pth
.DS_Store
```

Do not ignore source code, test fixtures, documentation, or small
benchmark reports. Review the ignore rules before committing.

------------------------------------------------------------------------

## 7. Roadmap and timeline

Planning assumption: **5--7 hours per week**. The eight-week schedule is
a target, not a deadline. Extend a phase if the underlying concepts are
not clear yet.

### Phase 0 --- Environment and project skeleton

**Target:** Week 1, first session

Tasks: - \[ \] Verify Python, `uv`, and Git. - \[ \] Create the project
and pin Python 3.12. - \[ \] Install dependencies. - \[ \] Verify
imports and tools. - \[ \] Initialize Git and commit the baseline. - \[
\] Add a README with the project goal and setup commands.

Acceptance criteria: - \[ \] A clean terminal session can run the
verification commands. - \[ \] Dependencies are recorded in
`pyproject.toml` and `uv.lock`. - \[ \] Git tracks project files but not
the environment or model weights.

### Phase 1 --- Inspect a real model

**Target:** Week 1

Tasks: - \[ \] Create `experiments/inspect_model.py`. - \[ \] Load GPT-2
and its tokenizer. - \[ \] Run inference explicitly on CPU. - \[ \]
Print model type, layer count, attention heads, hidden size, vocabulary
size, and parameter count. - \[ \] Print prompt token IDs and token
count. - \[ \] Print logits shape and greedy next token. - \[ \] Record
what the model library still owns.

Acceptance criteria: - \[ \] The model loads and runs on CPU. - \[ \]
You can explain each printed value and the logits shape. - \[ \] The
script works again after the first model download without unnecessary
re-downloading.

### Phase 2 --- Implement our own generation loop

**Target:** Week 2

Tasks: - \[ \] Implement a simple `generate_greedy` function. - \[ \]
Call the model forward pass directly, not `model.generate()`. - \[ \]
Select the next token using `argmax`. - \[ \] Append the selected token
and repeat. - \[ \] Stop at a configured token limit or a valid EOS
condition. - \[ \] Decode generated tokens into text. - \[ \] Add
deterministic tests for stopping and token selection.

Acceptance criteria: - \[ \] The generation loop is owned by this
project. - \[ \] It returns generated token IDs and readable text. - \[
\] It respects the maximum-new-token limit. - \[ \] The code can explain
the distinction between prompt tokens and generated tokens.

### Phase 3 --- KV cache and correctness

**Target:** Week 3

Tasks: - \[ \] Understand why repeated full-prefix computation is
wasteful. - \[ \] Implement a cached path using the model API's
supported cache interface. - \[ \] Compare cached and uncached greedy
generation. - \[ \] Test token outputs and numerical differences where
applicable. - \[ \] Record cache shapes and memory cost where
accessible. - \[ \] Document which cache operations are still
implemented by Transformers/PyTorch.

Acceptance criteria: - \[ \] Cached and uncached paths produce matching
greedy choices for a controlled test, allowing for documented numerical
differences. - \[ \] Cache initialization and updates are understood. -
\[ \] Tests catch incorrect token positions, cache reuse, and stopping
behavior.

### Phase 4 --- Measurement and OS behavior

**Target:** Week 4

Tasks: - \[ \] Measure prompt processing and generation time separately
where practical. - \[ \] Record prompt length, generated token count,
elapsed time, and tokens/second. - \[ \] Record process RSS memory and
CPU utilization with `psutil`. - \[ \] Repeat experiments instead of
trusting one run. - \[ \] Vary prompt length and generation length
independently. - \[ \] Observe the process in Windows Task Manager or
Resource Monitor. - \[ \] Document warm-up, caching, measurement
overhead, and limitations.

Acceptance criteria: - \[ \] A benchmark produces reproducible, clearly
labelled measurements. - \[ \] Results distinguish time-to-first-token
or prompt processing from later decoding where measurement supports
it. - \[ \] Conclusions are based on measurements, not assumptions.

### Phase 5 --- Runtime design and HTTP API

**Target:** Week 5

Tasks: - \[ \] Separate CLI/runtime code from API code. - \[ \] Create a
runtime interface that accepts a prompt and generation settings. - \[ \]
Add `fastapi` and `uvicorn`. - \[ \] Implement `/health` and a basic
generation endpoint. - \[ \] Add request validation, maximum token
limits, and useful error responses. - \[ \] Add a metrics endpoint. - \[
\] Add streaming only after ordinary requests work correctly. - \[ \]
Test API routes and runtime logic separately.

Acceptance criteria: - \[ \] CLI and API use the same runtime. - \[ \]
The API can generate text locally. - \[ \] Invalid input fails safely
and predictably. - \[ \] Core generation logic does not live inside
route handlers.

### Phase 6 --- C++ foundations and Python integration

**Target:** Week 6

Tasks: - \[ \] Set up a C++ compiler and build toolchain. - \[ \] Learn
pointers/references, object lifetime, RAII, vectors, compilation, and
linking. - \[ \] Write and test a small native component. - \[ \]
Benchmark it independently. - \[ \] Connect it to Python using
`pybind11` or another justified interface. - \[ \] Document data
conversions and memory ownership.

Acceptance criteria: - \[ \] The native component builds reproducibly. -
\[ \] Python can call it. - \[ \] Correctness is tested against a Python
reference implementation. - \[ \] Benchmark methodology and limitations
are documented.

### Phase 7 --- CPU experiments and optimization

**Target:** Week 7

Tasks: - \[ \] Profile before optimizing. - \[ \] Compare alternative
implementations of one selected operation. - \[ \] Investigate thread
count, memory allocation, data layout, and CPU utilization. - \[ \]
Record baseline and modified results under the same conditions. - \[ \]
Investigate SIMD or parallelism only when measurements justify it.

Acceptance criteria: - \[ \] At least one controlled experiment has a
written hypothesis, method, result, and conclusion. - \[ \] Any claimed
speedup is supported by repeatable measurements. - \[ \] Correctness
remains covered by tests.

### Phase 8 --- Integration and MiniRuntime v0.1

**Target:** Week 8

Tasks: - \[ \] Run tests and linting. - \[ \] Re-run setup instructions
from a clean environment. - \[ \] Update architecture and experiment
docs. - \[ \] Document limitations and known issues. - \[ \] Record
benchmark hardware and software configuration. - \[ \] Tag the milestone
in Git.

Acceptance criteria: - \[ \] A new checkout can be set up from the
documentation. - \[ \] Tests pass. - \[ \] The project has a clear
explanation of what is implemented by us versus libraries. - \[ \]
Benchmark results can be reproduced.

------------------------------------------------------------------------

## 8. First experiment: inspect GPT-2

Create `experiments/inspect_model.py`:

``` python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer


MODEL_NAME = "openai-community/gpt2"
PROMPT = "The future of artificial intelligence"


def main():
    print(f"Loading model: {MODEL_NAME}")

    tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME)
    model = AutoModelForCausalLM.from_pretrained(MODEL_NAME)
    model.to("cpu")
    model.eval()

    prompt_tokens = tokenizer(PROMPT, return_tensors="pt")

    print("\n--- Model information ---")
    print("Architecture:", model.config.model_type)
    print("Layers:", model.config.n_layer)
    print("Attention heads:", model.config.n_head)
    print("Hidden size:", model.config.n_embd)
    print("Vocabulary size:", model.config.vocab_size)
    print("Parameters:", f"{model.num_parameters():,}")

    print("\n--- Tokenizer output ---")
    print("Prompt:", PROMPT)
    print("Token IDs:", prompt_tokens["input_ids"].tolist())
    print("Token count:", prompt_tokens["input_ids"].shape[1])

    with torch.inference_mode():
        output = model(**prompt_tokens)

    print("\n--- Model output ---")
    print("Logits shape:", tuple(output.logits.shape))

    next_token_logits = output.logits[:, -1, :]
    next_token_id = next_token_logits.argmax(dim=-1).item()
    next_token = tokenizer.decode([next_token_id])

    print("Next token ID:", next_token_id)
    print("Greedy next token:", repr(next_token))


if __name__ == "__main__":
    main()
```

Run it from the project root:

``` powershell
uv run python experiments/inspect_model.py
```

The first run needs an internet connection to download model files. If
it fails, capture the full error message; don't guess at a fix.

### What to understand

The logits tensor has shape:

``` text
[batch_size, sequence_length, vocabulary_size]
```

For a single prompt, batch size is usually 1. The final sequence
position's logits score possible next tokens. `argmax` selects the
highest-scoring token; that is greedy selection, not the only possible
decoding strategy.

At this stage, Transformers still performs the transformer's forward
computation. We are inspecting the model, not claiming to have
implemented it ourselves.

------------------------------------------------------------------------

## 9. Benchmark plan

Create `experiments/benchmark.py` after the generation function is
stable.

Each experiment should record:

  Field                      Meaning
  -------------------------- ---------------------------------------------
  Model and model revision   Exact model used
  Device                     CPU for initial work
  Prompt token count         Input sequence length
  Generated token count      Number of new tokens
  Warm-up policy             Whether initial runs are excluded
  Total elapsed time         Wall-clock duration
  Time to first token        If measured consistently
  Decode tokens/second       Generated tokens divided by decode duration
  Process RSS                Resident process memory
  CPU utilization            Measurement method and interval
  Cache mode                 Cached or uncached
  Thread configuration       Relevant environment/settings
  Software/hardware          Python, PyTorch, OS, CPU model, RAM

Avoid mixing cold-start model loading with steady-state generation
latency. Run repeated trials and report the measurement method. CPU
utilization readings can be noisy, so don't over-interpret a single
sample.

### Experiments to run in order

1.  Different prompt lengths, with a fixed generation length.
2.  Different generation lengths, with a fixed prompt.
3.  Cached versus uncached generation.
4.  Warm versus first-run behavior.
5.  Different thread counts, when controllable.
6.  A native implementation versus a Python reference, once the native
    component exists.

For each experiment, write: - **Question:** What are we trying to
learn? - **Hypothesis:** What do we expect and why? - **Method:** What
is held constant and what changes? - **Results:** Raw measurements and
summary. - **Conclusion:** What the results do and do not prove.

------------------------------------------------------------------------

## 10. Core concepts checklist

Use this as a learning checklist alongside implementation.

### Model and inference

-   [ ] Tokenization and token IDs
-   [ ] Embeddings and positional information
-   [ ] Transformer layers and attention
-   [ ] Query, key, and value tensors
-   [ ] Feed-forward networks, residual connections, and normalization
-   [ ] Logits and next-token selection
-   [ ] Autoregressive decoding
-   [ ] Prefill versus decode
-   [ ] EOS and stopping conditions
-   [ ] KV cache and context length

### Systems and performance

-   [ ] Model weights versus runtime memory
-   [ ] Process memory / RSS versus total system memory
-   [ ] Virtual memory and page faults
-   [ ] CPU utilization versus throughput
-   [ ] CPU caches and memory bandwidth
-   [ ] Threading and synchronization
-   [ ] Data layout and memory allocation
-   [ ] Profiling versus guessing
-   [ ] Numerical precision and quantization (later)

### Software engineering

-   [ ] Separation of concerns
-   [ ] Unit tests and integration tests
-   [ ] Reproducible environments
-   [ ] Git commits and milestones
-   [ ] API request validation and error handling
-   [ ] Benchmark reproducibility
-   [ ] Python/C++ boundary and memory ownership

We do not need to master every concept before writing code. Learn the
concept when the next task gives it a concrete purpose.

------------------------------------------------------------------------

## 11. Engineering rules and common pitfalls

1.  **Do not call `model.generate()` for our own generation path.** Call
    the model forward pass and implement token selection and stopping
    ourselves.
2.  **Do not confuse a generation loop with a complete inference
    engine.** The initial transformer forward pass is still
    library-provided.
3.  **Do not optimize before measuring.** Keep a baseline and change one
    variable at a time.
4.  **Do not assume a larger CPU utilization value means faster
    inference.** Measure throughput and latency.
5.  **Do not compare runs with different prompts or settings and call
    the result a speedup.**
6.  **Do not assume cache memory equals model-weight memory.** Measure
    and document both where possible.
7.  **Do not implement C++ prematurely.** Establish a correct Python
    baseline first.
8.  **Do not build the API first.** The CLI and runtime should work
    independently.
9.  **Keep model files outside Git.** Record the model ID and revision
    instead.
10. **Record what each dependency does.** The project is educational, so
    transparency matters.

------------------------------------------------------------------------

## 12. Progress log and resume protocol

Use `docs/progress-log.md` to record progress after every work session.
Copy this template:

``` markdown
## Session YYYY-MM-DD

### Goal
What I planned to accomplish.

### Completed
- ...

### Commands run
- ...

### Tests and results
- ...

### Concepts understood
- ...

### Questions / blockers
- ...

### Git commit
- ...

### Next exact action
One concrete task to start with next session.
```

### Current status at project start

-   [ ] Prerequisites verified
-   [ ] Project initialized
-   [ ] Dependencies installed
-   [ ] Git baseline committed
-   [ ] GPT-2 inspection script created and run
-   [ ] Own generation loop implemented
-   [ ] KV cache path tested
-   [ ] Benchmarks recorded
-   [ ] FastAPI serving layer added
-   [ ] C++ component integrated
-   [ ] v0.1 documentation completed

### How to resume after a break

1.  Open this handbook and `docs/progress-log.md`.

2.  Check the current Git state:

    ``` powershell
    git status
    git log -5 --oneline
    ```

3.  Activate the project context by opening PowerShell at the repository
    root:

    ``` powershell
    cd D:\MiniRuntime
    ```

4.  Read the last session's **Next exact action**.

5.  Run the relevant tests before changing code:

    ``` powershell
    uv run pytest
    ```

6.  Make one small change, test it, record the result, and commit when
    the milestone is coherent.

7.  Update the progress log before stopping.

When returning to ChatGPT, share the latest progress-log entry, the
current task, and any error output. That is enough to continue from the
actual state rather than relying on memory or assumptions.

------------------------------------------------------------------------

## 13. First-session checklist

Do these in order and stop if one fails:

1.  [ ] Open PowerShell.
2.  [ ] Run `python --version`, `uv --version`, and `git --version`.
3.  [ ] Create `D:\MiniRuntime` if it does not already contain project
    files.
4.  [ ] Initialize the package with `uv`.
5.  [ ] Install the initial dependencies.
6.  [ ] Verify imports and test tools.
7.  [ ] Initialize Git and commit the clean baseline.
8.  [ ] Create and run `experiments/inspect_model.py`.
9.  [ ] Explain the printed model configuration, token IDs, and logits
    shape in your own words.
10. [ ] Update `docs/progress-log.md`.

**The immediate next action is Step 2: verify the installed tools.** Do
not begin the C++ phase, API phase, or optimization work yet. Establish
a working baseline first.

------------------------------------------------------------------------

## 14. Useful official references

-   PyTorch: https://pytorch.org/docs/stable/
-   Transformers documentation:
    https://huggingface.co/docs/transformers/
-   GPT-2 model page: https://huggingface.co/openai-community/gpt2
-   `uv` documentation: https://docs.astral.sh/uv/
-   FastAPI: https://fastapi.tiangolo.com/
-   pytest: https://docs.pytest.org/
-   pybind11: https://pybind11.readthedocs.io/
-   Python documentation: https://docs.python.org/3.12/
-   Git documentation: https://git-scm.com/doc

Check official documentation if a command or API changes; record the
version used when a change affects the project.

------------------------------------------------------------------------

**Project mantra:** understand → implement → test → measure → explain →
document.
