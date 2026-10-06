# Go/Rust hints

Go: find `runtime.main` / `main.main` first; recover via pclntab.  
Rust: collect `src/` path strings and `Option`/`Result` handling blocks first.  
Both: string-driven first; avoid getting lost in the runtime library.
