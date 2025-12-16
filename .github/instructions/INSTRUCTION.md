# RustFS Instructions Directory

## Overview

This directory contains orchestration instructions for different multi-agent coordination models. Each instruction file defines how agents collaborate to accomplish complex tasks.

## Instruction Files

| File | Model | Topology | Best For |
|------|-------|----------|----------|
| `hive_hierarchical.instruction.md` | HIVE | Hierarchical | Complex projects with clear delegation |
| `team_mesh.instruction.md` | TEAM | Mesh | Collaborative tasks, brainstorming |
| `pipeline_ring.instruction.md` | PIPELINE | Ring | Sequential workflows, CI/CD |

## Model Descriptions

### HIVE (Hierarchical)
- Central orchestrator coordinates all agents
- Clear delegation and escalation paths
- Best for structured, enterprise projects

### TEAM (Mesh)
- All agents can communicate directly
- Collaborative decision making
- Best for exploratory or creative tasks

### PIPELINE (Ring)
- Sequential processing through agents
- Each agent transforms and passes work
- Best for well-defined workflows

## Usage

Instructions are automatically applied based on task type. The orchestrator selects the appropriate model and coordinates agents accordingly.
