# Fixing Frequent Full Rust Builds With Cargo Fingerprints

Rust compile times can be very long and there are a ton of ways to [make it faster](https://corrode.dev/blog/tips-for-faster-rust-compile-times/). But sometimes the issue might be related to environment variables.

Recently, I encountered an issue in a large project where many crates were re-building frequently. Running 2 after 1 would rebuild 40+ crates even though there was no change in the code. This was very very sus 🕵️.

1. `make build/test` using a Makefile build script to call `cargo build/test`
2. `rust-analyzer` the lsp running `cargo check` in the background

Turns out some crates depend on environment variables and rebuild if they change. If this crate happens to be core dependency, all dependent crates are also rebuilt. But, how do I figure this out??? Using the below command that uses cargo instrumentation to inspect its guts aka the build log.

```
export CARGO_LOG="cargo::core::compiler::fingerprint=info"
export RUST_LOG=trace
cargo build -vv
```

Here's the offending log that shows the `VIRTUAL_ENV` path for the python interpreter triggered a rebuild because `rust-analyzer` was running in a separate process which did not have the python environment activated.

```
   0.359245532s  INFO prepare_target{force=false package_id=pyo3-build-config v0.24.1 target="build-script-build"}: cargo::core::compiler::fingerprint:     dirty: EnvVarChanged { name: "VIRTUAL_ENV", old_value: None, new_value: Some("/home/twitu/Code/nautilus_trader/.venv") }

       Dirty pyo3-build-config v0.24.1: the env variable VIRTUAL_ENV changed
[pyo3-build-config 0.24.1] cargo:rerun-if-env-changed=VIRTUAL_ENV
```

Ensuring that the build environments were consistent on important environment variables drastically [cut down on rebuilds](https://github.com/nautechsystems/nautilus_trader/pull/2524). Same applies to feature flags as well. For rust analyzer you can configure the environment variables and feature flags like so. For vscode it goes in the `.vscode/settings.json` by default.

```
{
  # Runs cargo check
  "rust-analyzer.check.extraEnv": {
    "CC": "clang",
    "CXX": "clang++",
    "VIRTUAL_ENV": "/home/twitu/Code/nautilus_trader/.venv"
  },
  # Runs cargo test
  "rust-analyzer.runnables.extraEnv": {
    "CC": "clang",
    "CXX": "clang++",
    "VIRTUAL_ENV": "/home/twitu/Code/nautilus_trader/.venv"
  },
  "rust-analyzer.cargo.features": "all",
  "rust-analyzer.check.features": "all"
}
```
