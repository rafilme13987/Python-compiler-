<img width="400" height="400" alt="Nouveau projet" src="https://github.com/user-attachments/assets/7f42062d-b600-4098-9509-95695eda3928" />

**This project is currently under active development.**

A beta version is coming out; I'm updating the GitHub.

# Python Compiler

A native, ahead-of-time (AOT) Python compiler written completely from scratch. 

This tool compiles pure Python code directly into standalone, native machine code (binary) for x86 and x64 architectures. Unlike existing solutions, it completely bypasses any intermediate C generation or heavy interpreter packaging.

---

## How It Works — Compilation Pipeline

The compiler transforms a `.py` source file into a native `.exe` in **6 stages**. Each stage is a distinct module with a clear responsibility.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         COMPILATION PIPELINE                                │
│                                                                             │
│  source.py                                                                  │
│      │                                                                      │
│      ▼                                                                      │
│  ┌─────────────┐                                                            │
│  │   LEXER     │  python_lexer.py                                           │
│  │             │  Splits raw text into a stream of tokens.                  │
│  │ "x = 1 + 2" │  e.g. [NOM 'x'] [OP '='] [INT '1'] [OP '+'] [INT '2']      │
│  └──────┬──────┘                                                            │
│         │  Token stream                                                     │
│         ▼                                                                   │
│  ┌─────────────┐                                                            │
│  │   PARSER    │  python_parser.py                                          │
│  │             │  Builds an Abstract Syntax Tree (AST) from the tokens.     │
│  │  Token[] →  │  e.g.  Affectation(cible='x',                              │
│  │    AST      │          valeur=OpBin('+', EntierLit(1), EntierLit(2)))    │
│  └──────┬──────┘                                                            │
│         │  AST (tree of Noeud objects)                                      │
│         ▼                                                                   │
│  ┌─────────────┐                                                            │
│  │  CODEGEN    │  python_codegen_x64.py                                     │
│  │             │  Walks the AST and emits x64 machine instructions.         │
│  │  AST →      │  e.g.  mov rax, 0x09   ; LORTE_VAL(1)                      │
│  │  x64 bytes  │        add rax, 0x11   ; LORTE_VAL(2)                      │
│  └──────┬──────┘        mov [rbp-8], rax ; store x                          │
│         │  Raw instruction bytes                                            │
│         ▼                                                                   │
│  ┌─────────────┐                                                            │
│  │   RUNTIME   │  python_runtime_x64.py                                     │
│  │             │  Provides built-in functions compiled into the binary:     │
│  │  lorte_*    │    lorte_print, lorte_list_new, lorte_dict_set,            │
│  │  functions  │    lorte_str_concat, lorte_alloc, print_flush, ...         │
│  └──────┬──────┘                                                            │
│         │  .text section bytes (code + runtime merged)                      │
│         ▼                                                                   │
│  ┌─────────────┐                                                            │
│  │ PE BUILDER  │  python_pe_x64.py  +  pe_builder.py                        │
│  │             │  Wraps everything into a valid Windows PE32+ executable:   │
│  │  bytes →    │    MZ header, PE header, .text, .data, .idata sections,    │
│  │  .exe file  │    Import Table (kernel32.dll, ws2_32.dll), relocations.   │
│  └─────────────┘                                                            │
│                                                                             │
│  output.exe    (single standalone binary, no dependencies)                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## The LORTE_VAL Type System

All Python values at runtime are represented as a single 64-bit integer called a **LORTE_VAL**. The 3 lowest bits are a **tag** that identifies the type. The remaining 61 bits carry the payload.

```
 63                                        3   2   1   0
 ┌──────────────────────────────────────────┬───────────┐
 │              PAYLOAD (61 bits)           │  TAG (3)  │
 └──────────────────────────────────────────┴───────────┘

  TAG  │  Type    │  Payload
 ──────┼──────────┼──────────────────────────────────────────────────────
   0   │  None    │  always 0
   1   │  int     │  actual integer value  (payload << 3 | 1)
   2   │  bool    │  0 = False, 1 = True   (payload << 3 | 2)
   3   │  str     │  pointer to heap struct { len: u64, data: utf-8... }
   4   │  list    │  pointer to heap struct { len, cap, items: ptr[] }
   5   │  object  │  pointer to heap struct { class_id, nattr, attrs[] }
   6   │  float   │  pointer to heap struct { value: f64 }
   7   │  dict    │  pointer to heap struct { len, cap, entries[] }

 Examples:
   Python 42       →  LORTE_VAL = (42 << 3) | 1  =  0x151
   Python True     →  LORTE_VAL = (1  << 3) | 2  =  0x0A
   Python "hello"  →  LORTE_VAL = (heap_ptr)  | 3
   Python None     →  LORTE_VAL = 0
```

This encoding means type checks are a single `AND rax, 7` instruction, and integer arithmetic only requires a shift. No boxing overhead for integers.

---

## Calling Convention

The compiler uses the **Windows x64 ABI** (Microsoft calling convention) throughout — both for calls into the runtime and for user-defined functions.

```
  User function:  def f(a, b, c):          Compiled user method:  Class__method(self, a, b)
                      │                                                │
          Arguments ──┤                              Arguments ────────┤
           a → RCX    │                               self → RCX       │
           b → RDX    │                               a    → RDX       │
           c → R8     │                               b    → R8        │
        return → RAX  │                            return  → RAX       │

  All LORTE_VAL values (tagged 64-bit integers).
  Stack must be 16-byte aligned before each CALL.
  32-byte shadow space reserved above return address.
```

---

## Multi-File Bundling (project.json)

When a project has multiple source files, the build system **bundles** them before compilation:

```
  compilateur.json
       │
       ▼
  build.py  →  bundliser_sources()
                    │
                    ├── binaire.py          ─┐
                    ├── python_lexer.py      │  ast.NodeTransformer removes:
                    ├── python_parser.py     │   • inter-module imports
                    ├── python_codegen.py    │   • try/except import blocks
                    ├── python_runtime.py    │   • if __name__ == "__main__"
                    └── python_natif.py    ──┘
                              │
                              ▼
                    _compiler_bundle.py   (single flat file, )
                              │
                              ▼
                    compiler_python_natif()
                              │
                              ▼
                    compiler.exe  ✓
```

---

## Concrete Example: `print("hello")`

Here is what happens at each stage for the single instruction `print("hello")`:

```
  SOURCE:   print("hello")

  ── LEXER ──────────────────────────────────────────────────────────────────
  [NOM 'print'] [OP '('] [CHAINE 'hello'] [OP ')'] [NEWLINE]

  ── PARSER ─────────────────────────────────────────────────────────────────
  ExprInstr(
    valeur = Appel(
      func = Nom(id='print'),
      args = [ ChaineLit(valeur='hello') ]
    )
  )

  ── CODEGEN ────────────────────────────────────────────────────────────────
  ; Evaluate ChaineLit("hello") → pointer to string stored in .data section
  lea  rax, [rip + offset_hello]      ; rax = ptr to "hello\0" in .data
  or   rax, 3                         ; tag as LORTE_STR
  mov  rcx, rax                       ; arg0 for lorte_print
  call lorte_print                    ; runtime function

  ── RUNTIME (lorte_print) ──────────────────────────────────────────────────
  ; Check tag (AND rax,7 → 3 = STR)
  ; Get ptr to utf-8 bytes and length from the LORTE_STR struct
  mov  rcx, str_data_ptr
  mov  rdx, str_len
  call print_raw                      ; buffered write to stdout
  ; Write newline '\n'
  call print_raw

  ── PRINT_RAW ──────────────────────────────────────────────────────────────
  ; Append to 8KB internal buffer (printbuf)
  ; When buffer is full → WriteFile(GetStdHandle(-11), printbuf, len, ...)
  ; Flushed at program exit (print_flush) or before sys.exit()

  ── PE OUTPUT (.data section) ──────────────────────────────────────────────
  offset_hello:  db 'h','e','l','l','o'   ; raw utf-8 bytes embedded in binary
                 dq 5                      ; length prefix in LORTE_STR struct
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

## Configuration File Example

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

