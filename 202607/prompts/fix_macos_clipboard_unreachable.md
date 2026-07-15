---
plan: 202607/fix_macos_clipboard_unreachable.md
---
 Why am I seeing this warning (see the output below)? Can you help me fix this without breaking anything? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.

```
bob-cli on  master is 📦 v0.1.0 via 🦀 v1.95.0
❯ cargo install --path . --locked --force
  Installing bob-cli v0.1.0 (/Users/bbugyi/projects/github/bbugyi200/bob-cli)
    Updating crates.io index
   Compiling bob-cli v0.1.0 (/Users/bbugyi/projects/github/bbugyi200/bob-cli)
warning: unreachable statement
   --> src/native/capture_clip.rs:186:5
    |
156 |           return run_required_command("pbpaste", &[], "pbpaste");
    |           ------------------------------------------------------ any code following this expression is unreachable
...
186 | /     if env::var_os("TMUX").is_some() {
187 | |         return run_required_command(
188 | |             "tmux",
189 | |             &["show-buffer"],
190 | |             "tmux show-buffer",
191 | |         );
192 | |     }
    | |_____^ unreachable statement
    |
    = note: `#[warn(unreachable_code)]` (part of `#[warn(unused)]`) on by default

    Building [======================>  ] 125/131: bob-cli
```