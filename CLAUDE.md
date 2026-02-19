# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MutCompute MCP is a Model Context Protocol server for structure-based protein mutation analysis. It wraps a 3D convolutional neural network (MutCompute) that predicts amino acid substitution probabilities from protein structures, and exposes it as MCP tools for use with Claude Code and other MCP clients.

## Dual Environment Architecture

This project requires **two separate Python environments** due to MutCompute's legacy dependencies:

- **`env/` (Python 3.10)**: Runs the MCP server (`fastmcp`, `loguru`, `pandas`, etc.)
- **`env_py36/` (Python 3.6)**: Runs the neural network inference (`theano==1.0.4`, `biopython==1.70`, Cython box_builder)

The MCP server in `src/tools/mutcompute_predict.py` calls the legacy environment via **subprocess** (`env_py36/bin/python repo/mutcompute/run.py`). The log-likelihood tool (`mutcompute_llh.py`) runs entirely in the Python 3.10 environment.

## Key Paths

All tool modules resolve paths relative to `PROJECT_DIR = SRC_DIR.parent` (the repo root):
- `PROJECT_DIR / "env_py36"` — legacy Python 3.6 environment
- `PROJECT_DIR / "repo" / "mutcompute"` — cloned MutCompute repo (contains model weights, pdb2pqr, source)
- `PROJECT_DIR / "repo" / "mutcompute" / "run.py"` — neural network entry point

In Docker, these are symlinks: `/app/env_py36` → `/opt/env_py36`, `/app/repo/mutcompute` → `/opt/mutcompute`.

## Build & Run Commands

```bash
# Setup (creates both conda environments)
bash quick_setup.sh

# Run MCP server
./env/bin/python src/server.py

# Run with fastmcp dev mode
./env/bin/fastmcp dev src/server.py

# Register with Claude Code
claude mcp add mutcompute -- ./env/bin/python src/server.py

# Docker build
docker build -t mutcompute-mcp .
docker run mutcompute-mcp

# Run tests
./env/bin/python -m pytest tests/test_server.py
./env/bin/python tests/test_sync_tools.py
./env/bin/python tests/run_integration_tests.py
```

## MCP Server Architecture

`src/server.py` creates a root `FastMCP(name="mutcompute")` and mounts two sub-servers:

1. **`mutcompute_predict_mcp`** (`src/tools/mutcompute_predict.py`)
   - Tool: `mutcompute_predict` — runs 3D-CNN ensemble on a PDB file via subprocess to `env_py36`
   - Returns per-residue mutation probabilities as CSV

2. **`mutcompute_llh_mcp`** (`src/tools/mutcompute_llh.py`)
   - Tool: `mutcompute_calculate_llh` — calculates log-likelihood ratios from MutCompute probabilities
   - Compares variant sequences to wild-type, computes Spearman/Pearson correlations with fitness data
   - Runs entirely in Python 3.10 (no subprocess)

Both tools use `multiprocessing.set_start_method('spawn', force=True)` set in `server.py`.

## System Dependencies

The neural network pipeline requires system tools not installable via pip:
- **python2** — required by `pdb2pqr-2.1` (PDB→PQR conversion)
- **freesasa** — solvent accessible surface area calculation (built from source in Docker)
- **CUDA** (optional) — GPU acceleration for Theano; CPU fallback works via `--cpu` flag or `THEANO_FLAGS=device=cpu,floatX=float32`

## Docker

Multi-stage Dockerfile:
- **Stage 1 (legacy)**: Clones MutCompute repo, builds conda Python 3.6 env, compiles Cython extensions and freesasa
- **Stage 2 (mcp)**: Copies legacy artifacts, creates Python 3.10 env, installs MCP deps, sets up symlinks

Model weights (~144MB) are baked into the image to avoid runtime downloads. The MutCompute repo is cloned from `github.com/charlesxu90/MutCompute.git` during build (configurable via `MUTCOMPUTE_REPO` and `MUTCOMPUTE_BRANCH` build args).

## Example Data

`examples/data/` contains test files: `1y4a_BPN.pdb` (bacterial serine protease), pre-computed `1y4a_BPN_mutcompute.csv`, and `wt.fasta`.

## Performance

- GPU inference: 30–180 seconds per protein structure
- CPU inference: 10–30 minutes per protein structure
- Log-likelihood calculation: ~2 seconds
- Memory: ~4–8GB RAM per protein
