# QuickJS fuzz target

This directory vendors **quickjs-ng** and contains all patches required to make the
`qjs` shell work as a Fuzzilli REPRL worker. The sources already embed:

- SanitizerCoverage hooks (`-fsanitize-coverage=trace-pc-guard`)
- REPRL data pump (`--reprl` / `-r` flag)
- `globalThis.fuzzilli()` helper for `FUZZILLI_CRASH` / `FUZZILLI_PRINT`

## Building the instrumented shell

```bash
cd /home/user/work/fuzzilli/Targets/quickjs
make qjs          # configures CMake under ./build and builds ./build/qjs
```

> ℹ️  The wrapper auto-detects `clang` for `trace-pc-guard` support.  
> Override with `FUZZILLI_CLANG=/path/to/clang` if needed.

The top-level `Makefile` injects the sanitizer-coverage flags through CMake cache
variables. You can customise the flags if needed:

```bash
FUZZILLI_EXTRA_CFLAGS="-fsanitize=address -fsanitize-coverage=trace-pc-guard" \
FUZZILLI_EXTRA_LDFLAGS="-fsanitize=address -fsanitize-coverage=trace-pc-guard" \
make qjs
```

Extra CMake arguments can be passed via `CMAKE_EXTRA_ARGS="..."`.

## Using the worker with Fuzzilli

- Worker binary: `Targets/quickjs/build/qjs`
- Required flag: `--reprl` (or `-r`)
- Optional: `--std` if the profile expects `std/os` modules

Example runner snippet:

```bash
FUZZILLI_HOME=/home/user/work/fuzzilli
TARGET=$FUZZILLI_HOME/Targets/quickjs/build/qjs
$FUZZILLI_HOME/Fuzzilli \
  --profile=/path/to/quickjs_profile.json \
  --storagePath=/tmp/fuzzilli-qjs \
  --worker-binary "$TARGET" \
  --worker-args="--reprl --std"
```

During execution Fuzzilli will talk to the REPRL pipes (FDs 100-103) and the
JS environment can communicate back via:

```js
fuzzilli('FUZZILLI_PRINT', 'hello from quickjs');
fuzzilli('FUZZILLI_CRASH', 0);   // deterministic crash
```

## Manual smoke tests

1. Build once with `make qjs`
2. Run `./build/qjs -e 'fuzzilli("FUZZILLI_PRINT", "ready")'`  
   You should see the string on stdout (falls back when REPRL pipes are absent)
3. Run `./build/qjs -e 'fuzzilli("FUZZILLI_CRASH", 0)'` to confirm crash handling

## Relationship to `Targets/QJS`

The older `Targets/QJS` directory keeps the historical patches against Fabrice
Bellard's repository. The current fuzzing workflow should go through this
`Targets/quickjs` tree instead—no manual patching is required and the build is
fully automated behind `make qjs`.
