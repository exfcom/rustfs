# Coding Style and Conventions

## Rustfmt Configuration
The project uses the following rustfmt configuration (defined in `rustfmt.toml`):
- `max_width = 130`: Maximum line width
- `fn_call_width = 90`: Maximum function call width
- `single_line_let_else_max_width = 100`: Maximum width for let-else statements

## Naming Conventions
- **Functions, variables, modules**: `snake_case`
- **Types, traits, enums**: `PascalCase`
- **Constants**: `SCREAMING_SNAKE_CASE`
- **Lifetimes**: `'lowercase`

## Error Handling
- ⛔ **NEVER use** `unwrap()` or `expect()` in production code
- ✅ **MUST use** `Result` type to propagate errors
- ✅ Use `thiserror` to define crate-specific error types
- `unwrap()` and `expect()` are acceptable in test code only

## Async Programming
- Keep async code non-blocking
- Offload CPU-intensive work with `tokio::task::spawn_blocking`
- Prefer `async-trait` for defining async traits
- Use structured concurrency patterns

## Testing Conventions
- Co-locate unit tests with their modules
- Use behavior-driven test names like `handles_expired_token`
- Integration tests go in each crate's `tests/` directory
- End-to-end tests live in `crates/e2e_test/`
- Include regression tests when fixing bugs or adding features

## Documentation
- Public APIs MUST have doc comments `///`
- Module-level documentation uses `//!`
- Add comments for complex algorithms
- Examples in doc comments are encouraged

## Clippy Rules
- All clippy warnings MUST be fixed (`-D warnings`)
- Workspace-level lints:
  - `unsafe_code = "deny"`: Unsafe code is forbidden
  - `clippy::all = "warn"`: Enable all clippy warnings

## Commit Conventions
- Use Conventional Commits format
- Commit messages should be under 72 characters
- Format: `<type>: <description>`
  - feat: New feature
  - fix: Bug fix
  - docs: Documentation
  - style: Formatting
  - refactor: Code refactoring
  - test: Tests
  - chore: Build/tooling
- Each commit must compile, format correctly, and pass `make pre-commit`

## Code Organization
- Keep functions focused and concise
- Prefer composition over inheritance
- Use type-driven design
- Avoid premature optimization
