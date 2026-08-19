---
layout: post
title: CrystalPotato - A GodPotato in Crystal
categories: [Red Team, Malware Development]
excerpt_separator: <!--more-->
---

CrystalPotato is a Crystal port of GodPotato, a local privilege escalation tool that abuses the DCOM OXID Resolver and named pipe impersonation to escalate from service accounts with `SeImpersonatePrivilege` to `NT AUTHORITY\SYSTEM`. This post covers why Crystal is an interesting language for offensive tooling and the OPSEC hardening applied to this implementation.

<!--more-->

<p align="center"><img src="https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/CrystalPotato/Screenshot_1.png" alt="RangerZone logo" width="420"></p>

<br>

---

## Background

[GodPotato](https://github.com/BeichenDream/GodPotato) is a well-known privilege escalation technique that works across a wide range of Windows versions (8 through 11, Server 2012 through 2022). It targets any process running with `SeImpersonatePrivilege`, which includes IIS application pools, MSSQL service accounts, Network Service and Local Service. The original tool is written in C# and, while effective, carries the usual .NET trade-offs: it requires the CLR, is easy to decompile, and its strings and class names are trivially visible in the assembly metadata.

I ported it to Crystal to produce a binary with a smaller forensic footprint.


<br>

---

## Why Crystal

[Crystal](https://crystal-lang.org/) is a statically typed, compiled language with Ruby-like syntax. It compiles to native code via LLVM, producing standalone executables with no runtime dependency. Several properties make it interesting for offensive development:

- **Native binaries**: Crystal compiles to a single PE on Windows with no interpreter, VM or runtime to deploy alongside it.

- **Compile-time macros**: Crystal's macro system executes arbitrary logic at compile time, which can be used for string obfuscation, code generation and other transformations that happen before the binary is produced.

- **Direct C interop**: Crystal can call C functions and Windows API functions directly through its `lib` FFI bindings, with no wrapper libraries needed.

- **Low analyst familiarity**: Crystal is rarely seen in malware samples, which means analysts and automated tooling are less likely to have signatures or heuristics tuned for Crystal-compiled binaries.

- **No reflection metadata**: Unlike .NET assemblies, Crystal binaries do not embed rich type metadata that can be trivially decompiled back to source code.


<br>

---

## The Technique

The privilege escalation chain used by CrystalPotato (and the original GodPotato) works as follows:

1. **RPC hook**: The tool locates the OXID Resolver's RPC dispatch table inside `combase.dll` by searching for the well-known interface GUID. It then patches the first entry in the dispatch table to redirect `UseProtseq` calls to a custom hook function.

2. **Named pipe server**: A named pipe is created with a permissive DACL. The hook function injects the pipe's endpoint address into the RPC binding, causing the OXID Resolver (running as SYSTEM) to connect to our pipe.

3. **DCOM trigger**: A COM object is marshaled and then unmarshaled with a crafted `OBJREF` structure pointing to `127.0.0.1` over TCP. This forces the OXID Resolver to resolve the object reference, which triggers the hooked `UseProtseq` function.

4. **Token capture**: When SYSTEM connects to the named pipe, `ImpersonateNamedPipeClient` captures its security context. The tool then enumerates all system handles to find a SYSTEM token with `Impersonation` level and sufficient integrity, duplicates it, and uses it to create a new process via `CreateProcessWithTokenW`.

The entire chain runs in-process with no child process spawned until the final command execution.


<br>

---

## OPSEC Hardening

The main contribution of this port beyond the language change is a set of OPSEC improvements designed to reduce the binary's signature surface. Running `strings` against the compiled executable should reveal very little about the tool's purpose.


<br>

### Compile-Time String Obfuscation

All strings that reveal the tool's behavior are XOR-obfuscated at compile time using a Crystal macro. The `obf()` macro takes a string literal, XORs each byte with a fixed key at compile time, and emits the encoded bytes as a constant array. At runtime, the bytes are decoded back to a string. This means the plaintext strings never appear in the binary's `.rdata` section.

The obfuscated strings include: DLL names (`combase.dll`), named pipe paths, protocol identifiers (`ncacn_ip_tcp`, `ncacn_np`), SDDL security descriptors, well-known SIDs (`S-1-5-18`), IP addresses (`127.0.0.1`), all debug and error messages, and all exception strings.


Usage is transparent:

```crystal
combase_name = obf("combase.dll").to_utf16
```

<br>

### RTTI Name Sanitization

Crystal embeds struct and class names as RTTI metadata in the compiled binary. Names like `RpcServerInterface`, `RpcDispatchTable`, `MidlServerInfo` or `SystemHandleTableEntryInfoEx` immediately tell an analyst what the tool is doing. All structs with revealing names have been renamed to short, meaningless identifiers (`Ri`, `Rd`, `Mi`, `He`, etc.) that carry no semantic information.

<br>

### Silent by Default

By default, the tool produces no output other than the executed command's stdout. All diagnostic messages (hook status, pipe events, token search progress, DCOM trigger details) are suppressed unless the `-d` / `--debug` flag is passed. This means a successful execution leaves no console artifacts beyond the command output itself.

<br>

---

## Build

CrystalPotato compiles to a single file with no dependencies. Cross-compilation from Linux to Windows is supported by Crystal's toolchain:

```
crystal build CrystalPotato.cr -o CrystalPotato.exe --release
```

<br>

---

## Usage

```
CrystalPotato.exe -c <COMMAND>
```

| Flag | Description |
|---|---|
| `-c CMD` | Command to execute as SYSTEM (required) |
| `-p NAME` | Custom pipe name (default: `Crystal`) |
| `-d` | Verbose debug output |
| `-h` | Show help |

<img src="https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/CrystalPotato/Screenshot_2.png" alt="CrystalPotato" width="720">
