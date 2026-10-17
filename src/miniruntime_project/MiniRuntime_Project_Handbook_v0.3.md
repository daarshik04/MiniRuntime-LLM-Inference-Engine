# MiniRuntime — Project Handbook

**Version:** 0.3  
**Updated:** 16 October 2026  
**Project type:** Educational LLM inference runtime + personal AI platform  
**Primary development environment:** Windows + PowerShell  
**Hardware constraint:** Personal PC with 16 GB RAM

---

## 1. Project vision

MiniRuntime is both:

1. **An inference and systems-learning project** for understanding how an LLM is loaded, how text becomes tokens, how generation works, how memory is used, and how CPU execution can be measured and improved.
2. **The foundation of a personal AI platform** that the developer controls, with persistent conversations, searchable memory, project continuity, and eventually tools for planning and carrying out software work.

The goal is not to outperform mature inference engines. The goal is to understand the important mechanisms, implement selected parts ourselves, use established components where sensible, and create a useful system whose behavior can be inspected.

### Design principles

- Learn by implementing, measuring, testing, and explaining—not by copying a complete codebase blindly.
- Use **both Python and C++**. Python provides fast experimentation and application orchestration; C++ provides hands-on experience with native execution, memory layout, allocation, threading, and performance.
- Separate our own code from functionality delegated to third-party libraries.
- Keep original conversations and project records outside the model weights.
- Make memory and context observable: show what is stored, what is retrieved, and what is actually sent to the model.
- Build incrementally, with a working, tested milestone at each phase.
- Prefer local processing for privacy when hardware and model quality allow it.
- Use safe file and command permissions for any future agent capabilities.

## 2. Target architecture

```text
User
  |
  v
Chat UI / CLI (initially a CLI)
  |
  v
Python application / orchestration layer
  |--------------------|--------------------|
  v                    v                    v
Context engine      Memory engine       Agent/tool layer (later)
  |                    |
  |                 SQLite + search
  |                    |
  +----------+---------+
             v
       Inference API
             |
       Python/C++ boundary
             |
       C++ learning/runtime components
             |
       Model backend
       (Transformers first; llama.cpp later for comparison)
             |
             v
        CPU / RAM / storage
```

This is a target architecture, not a requirement to implement every box immediately. The first milestone will be a small CLI program with an inspectable model call. We will add the surrounding platform one capability at a time.

### Main subsystems

1. **Inference subsystem:** model and tokenizer loading, input preparation, token generation, stopping rules, runtime measurements, and backend integration.
2. **C++ systems subsystem:** small native components that teach memory ownership, data layout, allocation, interfaces, profiling, and eventually token-processing or sampling utilities. We will not reimplement all tensor operations before we understand the full pipeline.
3. **Context engine:** chooses system instructions, recent messages, retrieved memories, project state, and the current request while respecting the model's context budget.
4. **Memory engine:** stores original messages and structured project state; supports full-text search first and embeddings/vector search when justified.
5. **Agent/tool engine:** later plans tasks, accesses approved files, runs commands and tests, checks results, and persists progress.
6. **Interface:** CLI first; web UI and voice later.
7. **Privacy and control:** local storage, explicit permissions, inspectable retrieval, and memory correction/export/deletion features.

## 3. Core concepts to keep distinct

- **Model weights:** learned parameters produced by training or fine-tuning. They are not the user's personal conversation archive.
- **Context:** the information supplied to the model for one inference request.
- **Conversation history:** original user and assistant messages persisted outside the model.
- **Persistent memory:** selected facts or summaries that can be retrieved in future sessions.
- **KV cache:** intermediate attention states used to accelerate token generation. It is not long-term memory.
- **Project state:** structured goals, requirements, decisions, task status, test results, blockers, and next steps.
- **Inference backend:** software that executes the model. A model file alone is not an API server.
- **API/orchestration layer:** application code that prepares requests, calls the backend, and coordinates memory and tools.

A model can only use old information if the application includes that information in the current context or provides it through a tool. A larger context window helps but does not replace a well-designed memory and retrieval system.

## 4. Language and tooling strategy: Python + C++

### Python responsibilities

Python is the main experimental and orchestration language because it is quick to change and has a mature AI ecosystem.

- Hugging Face Transformers and PyTorch for early model inspection and experiments.
- Tokenizer/config inspection, prompt construction, and generation experiments.
- SQLite persistence and retrieval logic.
- Context assembly, token-budget accounting, evaluation, and logging.
- A local API later, such as FastAPI.
- Test harnesses and benchmarks that compare implementations.

### C++ responsibilities

C++ is a first-class learning track, not an optional afterthought. We will use it in small, clearly scoped components and expand when we can measure the result.

- Learn compilation, linking, headers, translation units, interfaces, RAII, and error handling.
- Implement and test simple token-buffer/data structures and a sampler utility where useful.
- Study contiguous memory, allocation patterns, ownership, cache locality, and copying.
- Measure CPU time and allocations; compare a Python implementation with a C++ implementation of the same bounded task.
- Later explore multithreading, SIMD, and a native inference backend or bindings.
- If useful, connect Python and C++ with pybind11 or a small local process/API boundary. Choose the boundary based on the experiment rather than adding a binding layer prematurely.

**Important distinction:** writing some C++ does not mean we have implemented the model's matrix multiplications or the complete transformer. Initially, Transformers/PyTorch executes tensor operations. We will label every component as either our implementation or a library/backend responsibility.

### Development tools

- Windows and PowerShell.
- Python and `uv` for Python environment and dependency management, if already installed.
- A C++ compiler/toolchain: verify the available setup first (for example, Visual Studio Build Tools/MSVC or a suitable LLVM/MinGW toolchain). Do not assume one is installed.
- Git for version control.
- PyTorch + Transformers for the first direct-model experiments.
- SQLite for durable conversations and project records.
- Ollama may remain useful as a comparison point, but it should not hide the inference path we are trying to learn.
- Later: llama.cpp for quantized local inference comparisons; FastAPI and a UI once the core behavior is tested.

## 5. Model strategy and hardware constraints

### Primary educational track

Use **Qwen/Qwen3-1.7B** with Transformers/PyTorch as the initial model candidate. It provides a modern decoder-only instruction-tuned model whose tokenizer, configuration, input preparation, and generation can be inspected through Python. The first task is to validate actual compatibility, memory use, speed, and output quality on the available PC.

Reference: https://huggingface.co/Qwen/Qwen3-1.7B

### Later deployment comparison

Try a suitable quantized **Qwen3-4B** model through **llama.cpp** after the direct Transformers experiments. This is a practical deployment track using a mature inference engine; it is not our own transformer execution implementation.

References:
- https://huggingface.co/Qwen/Qwen3-4B
- https://github.com/ggml-org/llama.cpp

### Hardware discipline

The development PC has 16 GB RAM. Do not infer memory requirements from a model's advertised file size alone. Runtime usage can include model weights, temporary tensors, KV cache, tokenizer/runtime overhead, Python, the operating system, and memory-mapped files. Some systems can use disk-backed or partially mapped weights, but that may reduce speed and does not mean the full workload fits comfortably in physical RAM.

For every model experiment, record:

- Exact model ID and revision/quantization if applicable.
- Backend and library versions.
- Context length and generation settings.
- Model load time.
- Peak and steady-state RAM if measurable.
- Prompt length, output length, latency, and tokens per second.
- Whether CPU, GPU, or both are used.
- Errors, limitations, and reproducibility notes.

Never promise that a model will fit or run quickly before measuring it.

## 6. Suggested repository structure

Start small and expand only when a module exists. A reasonable target structure is:

```text
MiniRuntime/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── decisions.md
│   ├── experiments.md
│   └── project-state.md
├── pyproject.toml
├── uv.lock                    # created by uv when used
├── src/
│   └── miniruntime/
│       ├── __init__.py
│       ├── model_backend.py
│       ├── generation.py
│       ├── context.py
│       ├── memory.py
│       ├── storage.py
│       └── cli.py
├── cpp/
│   ├── CMakeLists.txt
│   ├── include/miniruntime/
│   ├── src/
│   └── tests/
├── tests/
│   ├── test_context.py
│   └── test_storage.py
├── data/                       # local data; review before committing
├── models/                     # optional; normally don't commit weights
├── benchmarks/
└── .gitignore
```

This is a planned structure, not a command to create every file at once. Begin with the minimum files needed for the current milestone. Keep model weights, virtual environments, private conversations, credentials, and large local databases out of Git unless there is a deliberate reason to version them.

## 7. Roadmap and acceptance criteria

### Phase 0 — Environment and project baseline

Tasks:
- Inspect the current project folder and existing handbook/code.
- Verify Python, uv, Git, compiler, CMake, available RAM, and current model tooling.
- Create or verify the repository and `.gitignore`.
- Record exact versions and the first architecture decisions.

Acceptance criteria:
- We know which tools are actually installed.
- A minimal Python script runs.
- A minimal C++ program compiles and runs using the chosen toolchain.
- The project has a reproducible setup note.

### Phase 1 — First model response

Tasks:
- Set up an isolated Python environment.
- Load the selected model and tokenizer through Transformers/PyTorch.
- Run one prompt and generate a response.
- Inspect tokenizer output, special tokens, model configuration, and input/output tensor shapes where practical.
- Record load time, RAM, and generation latency.

Acceptance criteria:
- A repeatable command generates a response.
- The model ID and dependency versions are recorded.
- A basic benchmark record exists.
- Limitations and resource usage are documented.

### Phase 2 — Understand the generation path

Tasks:
- Trace text → token IDs → model inputs → output logits → selected next token → decoded text.
- Inspect sampling controls such as temperature, top-k/top-p where supported, maximum new tokens, and stop conditions.
- Understand prefill versus decode and what the KV cache does.
- Separate code we wrote from operations handled by Transformers/PyTorch.

Acceptance criteria:
- We can explain the flow and show the relevant code/logs.
- Small experiments demonstrate how token count and generation settings affect latency/output.

### Phase 3 — C++ foundations and measured native components

Tasks:
- Create a minimal C++ build and test workflow.
- Learn headers, compilation, linking, RAII, standard containers, ownership, and errors through small exercises.
- Implement a bounded utility such as a token-ID buffer, stop-sequence checker, or sampling helper, where the design is meaningful.
- Test correctness and benchmark it against a simple Python equivalent.
- Inspect allocation/copying behavior and document what the benchmark does and does not prove.

Acceptance criteria:
- C++ code builds reproducibly and has tests.
- The utility's contract and edge cases are documented.
- The comparison uses the same inputs and correctness checks.
- We do not claim a speedup without measurements.

### Phase 4 — Runtime boundary and backend comparison

Tasks:
- Define a small backend interface in Python so application code is not tied to one model implementation.
- Keep the first backend on Transformers.
- Compare with Ollama or llama.cpp using equivalent prompts and settings where possible.
- Later decide whether C++ bindings, a subprocess, or an HTTP API is the best integration boundary.

Acceptance criteria:
- The application can call the chosen backend through a stable interface.
- Backend differences and delegated work are documented.

### Phase 5 — Persistent conversations

Tasks:
- Add SQLite tables for conversations and messages.
- Save messages with timestamps, role, content, and conversation ID.
- Add commands to list, open, and continue a conversation.
- Test shutdown/restart and error handling.

Acceptance criteria:
- Messages survive restart.
- Conversations can be retrieved without sending all history to the model.
- Basic database tests pass.

### Phase 6 — Context inspector and memory retrieval

Tasks:
- Assemble system instructions, recent messages, selected memories, project state, and current message.
- Track token counts and context budget; show truncation decisions.
- Add SQLite full-text search or a simple keyword baseline before adding embeddings.
- Retrieve a small number of relevant past messages or notes.
- Distinguish original records from summaries and derived memories.

Acceptance criteria:
- The UI/CLI can show the exact assembled context before inference.
- Token budget and truncation behavior are visible.
- Retrieval results include source references so the user can inspect them.
- Tests cover irrelevant retrieval, missing information, and conflicting/stale memories.

### Phase 7 — Durable project memory

Tasks:
- Maintain project goals, requirements, architecture decisions, tasks, status, test outcomes, blockers, and next actions.
- Keep a project-state file or structured database record that can be loaded in a fresh session.
- Import conversations only through user-provided exports or supported provider APIs; do not assume other services' private histories are automatically accessible.

Acceptance criteria:
- A new session can recover project status from stored records.
- The system can identify uncertainty and cite the source of a stored decision.
- Corrections and outdated information can be handled explicitly.

### Phase 8 — Agent tools and safe execution

Tasks:
- Add planning with tasks, dependencies, and acceptance criteria.
- Restrict file operations to an approved workspace.
- Add a command/test runner with timeouts and captured output.
- Verify changes with tests and report failures rather than claiming success.
- Require explicit confirmation for destructive operations, external communication, credential access, or other risky actions.

Acceptance criteria:
- The agent cannot silently access arbitrary files or run unrestricted commands.
- Task progress and tool results persist.
- Failed steps are reported accurately and can be resumed.

### Phase 9 — UI, voice, and optimization

Tasks:
- Add a web UI/API after the CLI and core layers are stable.
- Optionally integrate Piper TTS and a voice interface.
- Profile hotspots before optimizing.
- Explore C++ threading, SIMD, memory mapping, or deeper backend integration only when an experiment motivates it.

Acceptance criteria:
- UI and voice are replaceable frontends, not entangled with storage or inference logic.
- Performance changes are measured against a recorded baseline.

## 8. Personal memory model

A single database can support multiple logical kinds of memory; separate databases are not required at the beginning.

- **Episodic memory:** original conversations and events.
- **Semantic memory:** selected facts, preferences, goals, and stable information.
- **Project memory:** requirements, decisions, tasks, blockers, and test results.
- **Retrieval index:** searchable terms or embeddings that point back to original records.

Memory should be evaluated, not treated as magic. Track whether retrieved information is relevant, correct, current, and supported by a source. The user should be able to inspect and correct remembered information. Preserve original records separately from generated summaries so summaries can be regenerated if needed.

## 9. Testing and engineering workflow

For each feature:

1. State the problem and expected behavior.
2. Write a small design or interface contract.
3. Implement the smallest working version.
4. Add tests for normal behavior and edge cases.
5. Run tests and inspect the output.
6. Measure performance if performance is part of the goal.
7. Record the result, decision, limitations, and next step in `docs/`.
8. Commit a working checkpoint when using Git.

Do not add abstractions, dependencies, vector databases, agents, or native optimizations merely because they are common in AI projects. Add them when a requirement or experiment justifies them.

## 10. Privacy, reliability, and limitations

- Local inference can keep prompts on the machine when the selected backend and application are configured for local-only execution. Verify network behavior rather than assuming every dependency is offline.
- Downloading a model requires storage and usually internet access at least once. Model licenses and usage restrictions still apply.
- Local models remain limited by hardware, model quality, context length, and software correctness.
- Persistent memory can preserve mistakes. Store provenance and allow corrections.
- Retrieval can fail or return irrelevant records. Show sources and allow inspection.
- Tool-enabled agents can damage data if unrestricted. Use an isolated workspace, backups, command allowlists where practical, timeouts, and confirmation gates.
- Do not store API keys or credentials in source control or plain project notes.
- Import external chat histories only through supported exports or APIs and respect the source service's terms and privacy controls.

## 11. Decision log

Initial decisions for v0.3:

- MiniRuntime serves two purposes: learn inference/systems and build the foundation of a personal AI platform.
- Use both Python and C++; Python leads experimentation/orchestration and C++ is used for deliberate native systems exercises.
- Start with a CLI before a full UI.
- Use Transformers/PyTorch to inspect the early model path; compare mature inference backends later.
- Start persistent storage with SQLite; begin retrieval with full-text/keyword search before adopting vector search.
- Make context inspection an early feature.
- Keep project state separate from model weights and temporary KV cache.
- Validate toolchain and model fit on the actual PC before locking down implementation details.

## 12. Immediate next action

**Do not implement the entire roadmap yet. Start with Phase 0.**

In PowerShell, inspect the environment and the actual project state. Run these commands individually and note any errors:

```powershell
Get-Location
Get-ChildItem
python --version
py --version
uv --version
git --version
cmake --version
```

Some commands may not be installed; that is useful information, not a failure. Check for a C++ compiler using whichever toolchain is present—for example, `cl` from a Visual Studio Developer PowerShell, or `clang++`/`g++` if installed. Do not install several toolchains before we establish what is already available.

Then inspect the existing MiniRuntime folder and handbook. If no code exists, create the smallest baseline: one Python script that runs, one C++ program that compiles and runs, and a README that records how to reproduce both. After that, proceed to model loading.

**First milestone definition of done:** verified environment, working Python baseline, working C++ baseline, recorded versions, and a clean Git checkpoint. Only then begin model inference.
