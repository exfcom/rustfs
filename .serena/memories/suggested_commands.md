# Suggested Development Commands

## Quick Validation
```bash
# Fast validation without building binaries
cargo check --all-targets
```

## Building
```bash
# Release build
cargo build --release

# Or use Make (pipeline-aligned)
make build

# Development mode build
./build-rustfs.sh --dev

# Cross-platform compilation
./build-rustfs.sh --platform <target>
# Examples:
# ./build-rustfs.sh --platform x86_64-unknown-linux-musl
# ./build-rustfs.sh --platform aarch64-unknown-linux-musl
```

## Code Quality (MUST run before committing)
```bash
# Format code
cargo fmt --all

# Check formatting
cargo fmt --all --check

# Run Clippy
cargo clippy --all-targets --all-features -- -D warnings

# Run all checks in one command (RECOMMENDED)
make pre-commit
```

## Testing
```bash
# Run unit and integration tests (excluding e2e)
cargo test --workspace --exclude e2e_test

# Use nextest (if installed, faster)
cargo nextest run --all --exclude e2e_test

# Run all tests including documentation tests
cargo test --all
cargo test --all --doc

# Run e2e test server
make e2e-server

# KMS e2e tests (requires special environment variables)
NO_PROXY=127.0.0.1,localhost HTTP_PROXY= HTTPS_PROXY= cargo test --package rustfs-kms
```

## Running RustFS
```bash
# Using script (automatically downloads console)
./scripts/run.sh

# Using Docker
docker run -d -p 9000:9000 -p 9001:9001 \
  -v $(pwd)/data:/data -v $(pwd)/logs:/logs \
  rustfs/rustfs:latest

# Using Docker Compose with observability stack
docker compose --profile observability up -d
```

## Docker Building
```bash
# Build multi-architecture images
./docker-buildx.sh --build-arg RELEASE=latest

# Build and push
./docker-buildx.sh --push

# Using Make
make docker-buildx        # Build locally
make docker-buildx-push   # Build and push
```

## Common Utilities
```bash
# System commands (Linux)
ls, cd, grep, find, git

# Git workflow
git checkout -b feat/your-feature
git add .
git commit -m "feat: your commit message"
git push origin feat/your-feature
```

## Environment Variables Example (see scripts/run.sh)
```bash
export RUST_LOG="rustfs=debug,ecstore=info,s3s=debug,iam=info"
export RUST_BACKTRACE=1
export RUSTFS_VOLUMES="./target/volume/test{1...4}"
export RUSTFS_ADDRESS=":9000"
export RUSTFS_CONSOLE_ENABLE=true
export RUSTFS_CONSOLE_ADDRESS=":9001"
```

## Specific Crate Testing
```bash
cargo test -p rustfs-iam
cargo test -p rustfs-kms
cargo test -p rustfs-ecstore
```
