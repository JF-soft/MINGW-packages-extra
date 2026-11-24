# python-msgspec — MinGW build notes

Why this recipe carries patches that upstream does not have, what root
causes they fix, and how to reproduce the findings.

Target: `mingw-w64-ucrt-x86_64` (UCRT64), CPython 3.14, GCC 16.2.0,
msgspec 0.22.0. Both findings were verified empirically; see
[Verification](#verification).

## Patch 1 — `0001-mingw_skip_msvc_c11_flags.patch`

### Symptom

`build()` fails while compiling `src/msgspec/_core.c`:

```
gcc: error: /std:c11: linker input file not found: No such file or directory
gcc: error: /experimental:c11atomics: linker input file not found: No such file or directory
error: Command '['gcc', ..., '/std:c11', '/experimental:c11atomics']' returned non-zero exit status 1.
```

### Root cause

`setup.py` assumes every Windows build uses MSVC:

```python
if sys.platform == "win32":
    extra_compile_args.extend(["/std:c11", "/experimental:c11atomics"])
```

Those are MSVC flags. On MinGW (`sys.platform == "win32"` too) the
compiler is GCC, which rejects them. GCC supports C11 atomics by
default, so no equivalent flag is needed.

### Fix

Gate the flags on the compiler reported by `sysconfig`:

```python
cc = sysconfig.get_config_var("CC") or ""
if sys.platform == "win32" and "gcc" not in cc and "clang" not in cc:
```

(`CC` is `'gcc'` in MSYS2 environments.)

The official MSYS2 package removes these flags unconditionally
(`remove_msvc_arguments.patch`); our variant keeps them for real MSVC
builds and is the version proposed upstream.

## Patch 2 — `0001-mingw_generic_getattr_iat.patch`

### Symptom

Encoding a dataclass that has a `ClassVar` field leaks the ClassVar
into the JSON/MessagePack output:

```python
from __future__ import annotations
from dataclasses import dataclass
from typing import ClassVar

@dataclass
class Ex:
    a: int
    b: ClassVar[int] = 2

msgspec.json.encode(Ex(a=1))
# expected: b'{"a":1}'
# got:      b'{"a":1,"b":2}'        <- ClassVar default leaks
```

Affects only `json` and `msgpack` (the C encoder). `toml` and `yaml`
go through the pure-Python `to_builtins` path, which filters
`ClassVar` correctly via `dataclasses.Field._field_type`.

### Root cause

The C encoder iterates `__dataclass_fields__` (which *does* include
ClassVar entries). To iterate safely it checks:

```c
/* src/msgspec/_core.c, dataclass_iter_setup() */
iter->standard_getattr = (
    Py_TYPE(obj)->tp_getattro == PyObject_GenericGetAttr
);
```

If the check fails, the fallback calls `getattr(obj, name)` per field
name, which happily resolves **class-level attributes** — including the
ClassVar default `b = 2` — and emits them as fields.

On MinGW the check can never succeed:

1. CPython's `exports.h` declares, for extension modules on
   `__MINGW32__` (outside `Py_BUILD_CORE`):

   ```c
   #define PyAPI_FUNC(RTYPE) RTYPE      /* no __declspec(dllimport) */
   ```

2. Without `dllimport`, GCC/ld resolve `&PyObject_GenericGetAttr`
   inside the extension to a **linker-generated import thunk**
   (`jmp [__imp_PyObject_GenericGetAttr]`) living in the `.pyd`, not to
   the function in `libpython3.14.dll`.

3. Measured at runtime (instrumented build, same process):

   ```
   [SETUP] type=Ex tp_getattro=0x7ff9de6da140   (libpython3.14.dll + 0x18a140)
                       GenericGetAttr=0x7ffa04ee3ed0 (inside _core.pyd) equal=0
   ```

   The two pointers will never be equal, so the fast path is disabled
   on every MinGW-built extension using this pattern, and the leaky
   fallback always runs.

MSVC builds are unaffected (`PyAPI_FUNC` uses `__declspec(dllimport)`,
so `&func` reads the IAT and yields the real address), and so are
ELF/Linux builds (no import thunks). Upstream CI therefore stays green
while every MinGW consumer silently encodes extra keys.

### Fix

When compiling with MinGW, resolve the real address through the import
address table instead of using the thunk address:

```c
#ifdef __MINGW32__
extern void *__imp_PyObject_GenericGetAttr;
#  define MSGSPEC_IS_GENERIC_GETATTR(type) \
    ((type)->tp_getattro == (getattrofunc)__imp_PyObject_GenericGetAttr)
#else
#  define MSGSPEC_IS_GENERIC_GETATTR(type) \
    ((type)->tp_getattro == PyObject_GenericGetAttr)
#endif
```

The linker symbol `__imp_PyObject_GenericGetAttr` is the IAT slot; its
value is the real function address inside `libpython3.14.dll`.

With the fix, the fast-path check behaves exactly as on MSVC/ELF: the
instance dict is used for lookups, missing keys fall back to a
descriptor check that skips plain class attributes, and ClassVar
defaults are no longer emitted. Upstream tests
`test_encode_dataclass_classvars_ignored[json]` and `[msgpack]` pass
without any skip.

## Verification

| Build | `encode(Ex(a=1))` (string ann.) | upstream classvars tests |
| --- | --- | --- |
| 0.22.0 unpatched (MinGW) | `{"a":1,"b":2}` (leak) | FAIL (json, msgpack) |
| 0.22.0 + patches (MinGW) | `{"a":1}` | PASS |
| 0.22.0 (official MSYS2 pkg) | same as unpatched (no IAT fix in its PKGBUILD) | n/a |

- Full suite after the fix: `6489 passed, 113 skipped, 0 failed`
  (`makepkg-mingw --cleanbuild --syncdeps --force --noconfirm`, UCRT64).
- The `from __future__ import annotations` variant and the real-`ClassVar`
  variant both leak before the fix and both pass after it.
- Diagnosis was done with an instrumented `_core.c` build
  (`fprintf` in `dataclass_iter_setup`/`dataclass_iter_next`) plus
  `ctypes` slot inspection (`PyTypeObject.tp_getattro` at offset 144)
  and `objdump -p` import-table checks.

## Reporting upstream

Both issues should be reported to <https://github.com/msgspec/msgspec>:

1. `setup.py` MSVC flags — blocks any MinGW build.
2. `_core.c` getattro pointer check — data-correctness bug for MinGW
   consumers (extra keys in encoded output).

Both patches are inert outside MinGW (`#ifdef __MINGW32__` / compiler
check), so they cannot affect the project's own CI.

A runnable reproducer that builds upstream `main` and fails `check()`
with the classvars test lives in `../python-msgspec-git/`
(`makepkg-mingw` in a UCRT64 shell; the check failure is the bug
demonstration, documented in its PKGBUILD). See also the MSYS2
package, which carries a variant of patch 1 and has the patch 2 bug.
