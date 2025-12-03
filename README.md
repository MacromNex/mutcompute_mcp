# MutCompute MCP

MutCompute MCP server for protein modeling.

## Overview
This MutCompute MCP server support two tools: 1. Mutation recommendations for each site; 2. Likelihood calculation given variants.

## Installation

```bash
# Create and activate virtual environment
mamba env create -p ./env python=3.10 pip -y
mamba activate ./env
pip install -r requirements.txt 
pip install loguru sniffio biopython scipy numpy requests

pip install --ignore-installed fastmcp
```
Download the model parameters from aws.
```shell
# Install aws if not installed.
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Download mokdels
cd repo/MutCompute/models
./download_models.sh
```

## Local usage
### 1. Calculate Mutation Probability Matrix
```shell
python scripts/run_spired.py --fasta examples/test.fasta --output examples/spired --repo repo/MutCompute
```

### 2. Estimate MutCompute likelihoods
```shell
python scripts/run_spired_fitness.py --fasta examples/test.fasta --output examples/spired --repo repo/MutCompute
```

## MCP usage

### Install MCP Server
```shell
fastmcp install claude-code mcp-servers/MutCompute_mcp/src/MutCompute_mcp.py --python mcp-servers/MutCompute_mcp/env/bin/python
fastmcp install gemini-cli mcp-servers/MutCompute_mcp/src/MutCompute_mcp.py --python mcp-servers/MutCompute_mcp/env/bin/python
```

### Call MCP service
### 1. Calculate Mutation Probability Matrix
```markdown
Can you help train a ProtTrans model for data @examples/case2.1_subtilisin/ and save it to 
@examples/case2.1_subtilisin/prot-t5_fitness using the ProtTrans mcp server with ProtT5-XL model.

Please convert the relative path to absolution path before calling the MCP servers. 
```

### 2. Estimate MutCompute likelihoods
```markdown
```
