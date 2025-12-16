# Development Workflow

## Starting New Feature Development

### 1. Sync Main Branch
```bash
git checkout main
git pull origin main
```

### 2. Create Feature Branch
```bash
git checkout -b feat/your-feature-name
```
Branch naming conventions:
- `feat/`: New features
- `fix/`: Bug fixes
- `docs/`: Documentation updates
- `refactor/`: Code refactoring
- `test/`: Test-related changes
- `chore/`: Build/tooling updates

### 3. Development Preparation
```bash
# Set up Git hooks (first time only)
make setup-hooks

# Quick environment validation
cargo check --all-targets
```

## Iterative Development

### Development Loop
1. **Write Code**
   - Follow coding style and conventions
   - Add necessary tests
   - Update relevant documentation

2. **Quick Validation**
   ```bash
   cargo check --all-targets
   ```

3. **Run Tests**
   ```bash
   cargo test --workspace --exclude e2e_test
   # Or use nextest
   cargo nextest run --all --exclude e2e_test
   ```

4. **Run Locally**
   ```bash
   ./scripts/run.sh
   # Access http://localhost:9000 (S3 API)
   # Access http://localhost:9001 (Console)
   ```

## Committing Code

### 1. Run Pre-Commit Checks
```bash
make pre-commit
```
This automatically runs:
- Code formatting
- Clippy checks
- Compilation checks
- Unit tests

### 2. Stage Changes
```bash
git add <files>
```

### 3. Commit
```bash
git commit -m "feat: add feature description"
```
Commit message format:
- Use Conventional Commits
- Under 72 characters
- Clear description of changes

### 4. Push to Remote
```bash
git push origin feat/your-feature-name
```

## Creating Pull Requests

### PR Checklist
- [ ] Clear title and description
- [ ] Linked related issues
- [ ] Testing methodology explained
- [ ] Documentation updated (if needed)
- [ ] All CI checks passing
- [ ] Code review requested

### PR Description Template
```markdown
## Change Description
Brief description of this PR's purpose

## Changes Made
- Item 1
- Item 2

## Testing
Explain how to test these changes

## Related Issues
Closes #123
```

## Code Review

### As Author
- Respond to review comments promptly
- Explain design decisions
- Make necessary modifications
- Keep commits clean and focused

### As Reviewer
- Check code quality
- Verify test coverage
- Confirm documentation completeness
- Focus on security issues
- Provide constructive feedback

## After Merge
```bash
# Delete local branch
git branch -d feat/your-feature-name

# Delete remote branch
git push origin --delete feat/your-feature-name

# Update main branch
git checkout main
git pull origin main
```

## Common Scenarios

### Fixing Urgent Bugs
```bash
git checkout -b fix/urgent-bug
# Fix the code
make pre-commit
git commit -m "fix: urgent bug description"
git push origin fix/urgent-bug
# Create PR, mark as urgent
```

### Updating Dependencies
```bash
cargo update
cargo test --all
# Check for breaking changes
git commit -m "chore: update dependencies"
```

### Running Specific Crate Tests
```bash
cargo test -p rustfs-iam
cargo test -p rustfs-kms
cargo test -p rustfs-ecstore --features full
```

### Debugging Build Issues
```bash
# Clean build artifacts
cargo clean

# Rebuild with verbose output
cargo build -vv

# Check specific target
cargo check -p rustfs --target x86_64-unknown-linux-gnu
```

## Best Practices
- Commit early and often
- Keep commits atomic and focused
- Write descriptive commit messages
- Test before pushing
- Rebase instead of merge for cleaner history
- Use feature flags for experimental features
