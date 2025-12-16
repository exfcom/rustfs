# Task Completion Checklist

When completing a coding task, **MUST** execute the following checks in order:

## 1. Format Code ✨
```bash
cargo fmt --all
```

## 2. Verify Formatting ✅
```bash
cargo fmt --all --check
```
- If this fails, re-run the formatting command

## 3. Clippy Checks 🔍
```bash
cargo clippy --all-targets --all-features -- -D warnings
```
- All warnings MUST be fixed
- No clippy warnings are allowed

## 4. Compilation Check 🔨
```bash
cargo check --all-targets
```
- Ensure code compiles successfully

## 5. Run Tests 🧪
```bash
# Basic tests (excluding e2e)
cargo test --workspace --exclude e2e_test

# Or use nextest (recommended, faster)
cargo nextest run --all --exclude e2e_test

# Documentation tests
cargo test --all --doc

# Full tests (including e2e, before PR)
cargo test --all
```

## 6. One-Command Check (RECOMMENDED) 🚀
```bash
make pre-commit
```
This runs in sequence: fmt → clippy → check → test

## 7. Pre-Commit Checklist 📝
- [ ] No debug code left (e.g., `println!`, `dbg!`)
- [ ] Documentation updated if needed
- [ ] Tests updated if needed
- [ ] Git staging area contains only relevant changes
- [ ] Commit message follows Conventional Commits

## 8. Security Checklist 🔒
- [ ] No sensitive information committed (keys, passwords)
- [ ] IAM/KMS-related changes require second reviewer
- [ ] Environment variables configured correctly
- [ ] No credentials in code or config files

## Special Considerations
- For IAM or KMS-related code changes, perform additional security review
- KMS e2e tests require special proxy settings:
  ```bash
  NO_PROXY=127.0.0.1,localhost HTTP_PROXY= HTTPS_PROXY=
  ```
- Run `make setup-hooks` to set up Git hooks for automatic checks before commits

## After PR Merge
- Delete feature branch locally and remotely
- Update main branch
- Verify CI/CD pipeline success
