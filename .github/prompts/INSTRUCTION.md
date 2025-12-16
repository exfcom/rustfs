# RustFS Prompts Directory

## Overview

This directory contains technology-specific prompts for the RustFS multi-agent system. Each prompt provides specialized knowledge and coding patterns for different aspects of the project.

## Prompt Files

| Prompt File | Domain | Description |
|-------------|--------|-------------|
| `rust-core.prompt.md` | Rust Core | Core Rust patterns, idioms, and best practices |
| `rust-async.prompt.md` | Async Rust | Tokio, async/await, concurrent programming |
| `rust-storage.prompt.md` | Storage Systems | Erasure coding, file I/O, metadata management |
| `rust-s3api.prompt.md` | S3 API | S3-compatible API implementation patterns |
| `rust-security.prompt.md` | Security | IAM, KMS, encryption, authentication |

## Usage

Prompts are automatically loaded based on task context. They provide:
- Language-specific coding patterns
- Framework best practices  
- Domain-specific knowledge
- Code templates and examples

## Prompt Structure

Each prompt follows this format:
```yaml
---
domain: "Domain name"
language: "rust"
frameworks: ["tokio", "axum", "serde"]
---

# Domain-Specific Content
...
```
