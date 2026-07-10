<img width="400" height="400" alt="Nouveau projet" src="https://github.com/user-attachments/assets/7f42062d-b600-4098-9509-95695eda3928" />

**This project is currently under active development.**

# Python Compiler

A native, ahead-of-time (AOT) Python compiler written completely from scratch. 

This tool compiles pure Python code directly into standalone, native machine code (binary) for x86 and x64 architectures. Unlike existing solutions, it completely bypasses any intermediate C generation or heavy interpreter packaging.

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
## Speed ​​test comparing uncompiled and compiled Python

sur l'image est le code python qui est compiler. 
<img width="1114" height="763" alt="image" src="https://github.com/user-attachments/assets/44cc75be-8be9-4fb6-afc2-64d118266e1d" />
