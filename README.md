<img width="400" height="400" alt="Nouveau projet" src="https://github.com/user-attachments/assets/7f42062d-b600-4098-9509-95695eda3928" />

**This project is currently under active development.**

A beta version is coming out; I'm updating the GitHub.

# <img src="https://cdn.jsdelivr.net/npm/simple-icons@v3/icons/python.svg" width="28" height="28" align="top" /> Python Compiler

<p align="center">
  <img width="800" alt="Compiler Overview" src="/home/lorte/.gemini/antigravity-ide/brain/2760af21-9409-4ba5-bcc4-6107372e62c6/compiler_overview_1784768879050.png" />
</p>

A native, ahead-of-time (AOT) Python compiler written completely from scratch. 

This tool compiles pure Python code directly into standalone, native machine code (binary) for x86 and x64 architectures. Unlike existing solutions, it completely bypasses any intermediate C generation or heavy interpreter packaging.

## <img src="https://cdn.jsdelivr.net/npm/simple-icons@v3/icons/apache.svg" width="24" height="24" align="top" /> Deep Dive: The Compiler Architecture & Philosophy

The compiler is engineered from the ground up to bridge the massive gap between Python's high-level, dynamic execution model and the raw, unyielding performance of native machine code. Most attempts at compiling Python rely on either packing a heavy interpreter with the scripts (like PyInstaller) or transpiling Python into C code as an intermediate step (like Nuitka). This project takes the hardest, but most performant route: **Direct Native Compilation**.

### 1. The Core Engineering Challenge
Python is dynamically typed; variables don't have types, only values do. A simple expression like `a + b` could mean integer addition, string concatenation, or a custom class operator. A standard interpreter determines this at runtime on every single execution, which is slow. Our compiler resolves this by generating specialized polymorphic machine code instructions. It creates native pathways that check the data types (using our ultra-fast `LORTE_VAL` system) in a few CPU cycles and instantly branches to the correct low-level logic (e.g., `ADD RAX, RBX`).

### 2. Zero-Dependency Standalone Binaries
When we say "no dependencies", we mean it. The compiler parses the abstract syntax tree (AST) of your Python code and maps every single Python instruction directly to x86/x64 assembly instructions. 
It then manually structures the Windows PE32/PE64 (Portable Executable) headers, allocates the `.text` (code), `.data` (initialized data like strings), and `.rdata` sections, and writes the raw binary bytes to the disk. The resulting `.exe` doesn't need `python39.dll`, it doesn't need a C++ redistributable, and it doesn't need the user to install Python. It is purely the CPU talking directly to the operating system's kernel.

### 3. The Embedded Micro-Runtime
To make Python work natively, the compiler injects a highly optimized, microscopic runtime written directly in assembly. This micro-runtime is bound into the final executable. It handles:
*   **Memory Allocation**: A custom, lightning-fast heap allocator that replaces Python's traditional `malloc` wrappers.
*   **Built-in Types**: Native implementations of lists, dictionaries, strings, and floats using raw memory blocks and pointers, bypassing all standard Python overhead.
*   **String Manipulation**: Direct UTF-8 byte manipulation in memory for hyper-fast string slicing and concatenation.

### 4. Bypassing the Abstract Syntax Tree (AST) Bottlenecks
During compilation, the Lexer and Parser read the `.py` files and generate an AST. However, instead of passing this AST to an evaluation loop, our Code Generation engine walks the AST exactly once at compile-time. It translates complex loops (`for`, `while`) into direct native jumps (`JMP`, `JNE`, `JZ`) and variable assignments into direct memory offsets relative to the base pointer (`RBP - 8`). This means the structural logic of your code is permanently baked into the CPU's execution path.

By merging these concepts, the compiler achieves maximum hardware execution speed, bypassing all intermediate interpretation steps.

---

## How It Works — Compilation Pipeline

The compiler transforms a `.py` source file into a native `.exe` in **6 stages**. Each stage is a distinct module with a clear responsibility.

```mermaid
flowchart TD
    A[source.py] --> B[LEXER<br>python_lexer.py]
    B -->|Token stream| C[PARSER<br>python_parser.py]
    C -->|AST tree| D[CODEGEN<br>python_codegen_x64.py]
    D -->|x64 raw instructions| E[RUNTIME<br>python_runtime_x64.py]
    E -->|.text section bytes| F[PE BUILDER<br>pe_builder.py]
    F -->|MZ/PE Headers + Sections| G([output.exe<br>Standalone Binary])

    classDef stage fill:#1e1e1e,stroke:#4a90e2,stroke-width:2px,color:#fff;
    class B,C,D,E,F stage;
```

---

## The LORTE_VAL Type System

All Python values at runtime are represented as a single 64-bit integer called a **LORTE_VAL**. The 3 lowest bits are a **tag** that identifies the type. The remaining 61 bits carry the payload.

```mermaid
flowchart LR
    LV[LORTE_VAL<br>64-bit Integer] --> P[Payload<br>61 bits]
    LV --> T[Tag<br>3 bits]
    
    T -->|Tag 0| T0[None]
    T -->|Tag 1| T1[int]
    T -->|Tag 2| T2[bool]
    T -->|Tag 3| T3[str pointer]
    T -->|Tag 4| T4[list pointer]
    T -->|Tag 5| T5[object pointer]
    T -->|Tag 6| T6[float pointer]
    T -->|Tag 7| T7[dict pointer]

    style LV fill:#2d2d2d,stroke:#e0a96d,stroke-width:2px,color:#fff
```

This encoding means type checks are a single `AND rax, 7` instruction, and integer arithmetic only requires a shift. No boxing overhead for integers.

---

## Calling Convention

The compiler uses the **Windows x64 ABI** (Microsoft calling convention) throughout — both for calls into the runtime and for user-defined functions.

```mermaid
flowchart LR
    subgraph User Function: def f_a_b_c
        direction LR
        A[a] -->|RCX| X[Function Body]
        B[b] -->|RDX| X
        C[c] -->|R8| X
        X -->|RAX| R[Return]
    end
    
    subgraph Compiled Method: Class__method
        direction LR
        SA[self] -->|RCX| Y[Method Body]
        SB[a] -->|RDX| Y
        SC[b] -->|R8| Y
        Y -->|RAX| SR[Return]
    end
```

---

## Multi-File Bundling (project.json)

When a project has multiple source files, the build system **bundles** them before compilation:

```mermaid
flowchart TD
    CFG[compilateur.json] --> B[build.py]
    B -->|bundliser_sources| S{Source Files}
    S --> F1[binaire.py]
    S --> F2[python_lexer.py]
    S --> F3[python_parser.py]
    S --> F4[...]
    
    F1 & F2 & F3 & F4 -->|ast.NodeTransformer cleans imports| FLAT[_compiler_bundle.py]
    FLAT -->|compiler_python_natif| EXE([compiler.exe])

    style EXE fill:#3a7e40,stroke:#fff,stroke-width:2px
```

---

## Concrete Example: `print("hello")`

Here is what happens at each stage for the single instruction `print("hello")`:

```mermaid
flowchart TD
    S[1. Source Code] -->|"print 'hello'"| L[2. Lexer]
    L -->|"[NOM 'print'] [OP '('] [CHAINE 'hello'] [OP ')']"| P[3. Parser]
    P -->|"ExprInstr Appel Nom args"| C[4. Codegen]
    C -->|"lea rax, hello_ptr<br/>call lorte_print"| R[5. Runtime]
    R -->|"print_raw buffered"| PE[6. PE Output]
    PE -->|".data section: 'h','e','l','l','o'"| EXE((Native Execution))

    classDef step fill:#1a1a2e,stroke:#e94560,stroke-width:2px,color:#fff;
    class S,L,P,C,R,PE step;
```

---

## Technical Approach

Most Python packagers or compilers take one of two routes:
* **Interpreter Bundling:** Packaging CPython and dependencies into an archive (e.g., PyInstaller).
* **C Transpilation:** Translating Python code into C source code before compiling it (e.g., Nuitka).

**Python Compiler takes a direct route:** It parses the Python abstract logic and writes raw machine code instructions (binary octets) directly into the executable file. 

For example, when the compiler encounters a `print("hello world")` statement, it does not convert it to a C `printf`. Instead, it immediately maps the statement to the exact processor-level system calls required to output the text to the screen.

---

## Key Features

* **Direct Binary Generation:** Bypasses C generation to output raw machine code.
* **Zero Dependencies:** Outputs a single, lightweight, standalone executable.
* **Project Configuration:** Supports multi-file compilation using a structured JSON configuration file.
* **Architecture Targeting:** Capability to target x86 (32-bit) and x64 (64-bit) architectures.

---

## Project Status

**Warning:** This project is currently under active development. 

Because it targets machine code directly, building complete coverage for the entire Python ecosystem is a monumental task. As of now, heavy third-party libraries (such as PyQt, NumPy, etc.) are not yet integrated or supported. The focus is currently on stabilizing core Python syntax, built-in types, control flows, and file structures.

---

## <img src="https://cdn.jsdelivr.net/npm/simple-icons@v3/icons/codeigniter.svg" width="24" height="24" align="top" /> Debug Mode

The compiler includes a powerful built-in **Debug Mode** that allows developers to inspect the state of their program at runtime, right at the machine code level but with high-level Python context.

<p align="center">
  <img width="800" alt="Debug Mode Illustration" src="/home/lorte/.gemini/antigravity-ide/brain/2760af21-9409-4ba5-bcc4-6107372e62c6/debug_mode_illustration_1784768871433.png" />
</p>

When compiling with the debug flag, the compiler injects specialized tracking instructions alongside the standard machine code. This creates a continuous bridge between the executing binary and the original Python source.

### Debug Architecture Schema

```mermaid
graph TD
    A[Source Code .py] -->|Compile with --debug| B(Compiler)
    B --> C[Injected Debug Symbols]
    B --> D[Native Machine Code]
    C --> E{Debug Runtime}
    D --> E
    E -->|Breakpoints Hit| F[State Inspection]
    F --> G[Variables View]
    F --> H[Memory Dump]
    F --> I[Call Stack]
```

**Key Debug Features:**
*   **Variable Tracking**: Inspect local and global variables mapped from memory addresses back to Python objects using the `LORTE_VAL` type system.
*   **Call Stack Analysis**: Trace function calls with exact line numbers corresponding to the original `.py` source.
*   **Memory Inspection**: Directly view the heap structure of complex objects like lists and dictionaries to identify memory leaks or corruption.
*   **Breakpoint Handling**: Set breakpoints directly on Python source lines, which are mapped to the corresponding `INT 3` (0xCC) software breakpoints in the resulting binary.

---

## <img src="https://cdn.jsdelivr.net/npm/simple-icons@v3/icons/json.svg" width="24" height="24" align="top" /> Configuration File Example

For multi-file projects, you can manage the build architecture and source files using a `compiler.json` configuration file:

```json
{
  "project_name": "MyProject",
  "version": "1.0.0",
  "main_entry": "main.py",
  "source_files": [
    "modules/logic.py",
    "modules/utils.py"
  ],
  "target_architecture": "x64",
  "output_directory": "build/"
}
```
---

## Speed Test: Compiled vs Uncompiled Python

Below is a benchmark performance test highlighting the massive speed difference between standard interpreted Python and the binary executable generated by this compiler.

### 1. Compiled Execution (Native Binary)
The following image shows the execution time of the code after being compiled into a native binary:

<img width="1114" height="763" alt="Compiled Python Speed Test" src="https://github.com/user-attachments/assets/44cc75be-8be9-4fb6-afc2-64d118266e1d" />

---

### 2. Uncompiled Execution (Standard Python)
The following image shows the execution time of the exact same script running through the standard standard CPython interpreter:

<img width="1122" height="624" alt="Uncompiled Python Speed Test" src="https://github.com/user-attachments/assets/6ed1d83f-7a8f-44b7-af12-9a7ba4ca284b" />

---

### Benchmark Analysis

The results show a massive difference of **14.5844588186646 seconds** in favor of the compiled version. 

This performance leap is achieved because the compiler does not waste time loading an interpreter or evaluating bytecodes line by line at runtime. By generating direct machine code, the processor executes the logic natively at maximum hardware capability.

