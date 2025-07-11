# AI File Tree Generator

[![GitHub Repo](https://img.shields.io/badge/GitHub-Repo-blue?logo=github)](https://github.com/yourusername/ai-file-tree-generator)  
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)  
[![Bash Version](https://img.shields.io/badge/Bash-4%2B-green)](https://www.gnu.org/software/bash/)

## Overview

**ai-file-tree-generator** is a versatile Bash script designed to generate a text-based representation of a directory structure (file tree) for code projects. It is particularly useful for developers, AI assistants (like Grok), and teams who need to share or analyze project structures without exposing sensitive or irrelevant files. The script supports customizable exclusions via a `.treeignore` file, multiple output formats (e.g., ASCII art, JSON, YAML), and options for compact or human-readable outputs.

This tool was inspired by needs in collaborative development workflows, such as preparing project overviews for AI analysis or documentation. It handles common exclusions (e.g., `.git`, `node_modules`) intelligently and provides fallbacks if the `tree` command is unavailable.

Key use cases:
- Generating project trees for AI prompts (e.g., code reviews or debugging).
- Creating structured outputs for documentation, reports, or integration with tools like Graphviz.
- Filtering out build artifacts, dependencies, or sensitive files.

## Features

- **Custom Exclusions with `.treeignore`**:
  - Supports ignoring files/directories (e.g., `.git`, `*.log`).
  - "contents:" prefix to show a directory but hide its contents (e.g., `contents:.venv`).
  - Validates patterns, warns on duplicates or issues, and deduplicates automatically.
  - If no `.treeignore` exists, the script warns and offers to create a sample.

- **Output Formats**:
  - `grok-ascii` (default): ASCII art tree (e.g., using `├──` branches).
  - `json`: Nested JSON object.
  - `yaml`: Indented YAML structure.
  - `xml`: Tagged XML hierarchy.
  - `markdown`: Bullet-point list for easy rendering in docs.
  - `dot`: Graphviz DOT format for visual diagrams.
  - `csv` or `excel`: Flat CSV (path,type) importable to spreadsheets.

- **Compact Mode (--pack)**:
  - Strips whitespace for smaller files (compatible with JSON, YAML, XML, Markdown, DOT).
  - Defaults to human-readable (indented) output.

- **Fallback Support**:
  - Uses the `tree` command if available for optimal output.
  - Falls back to `find` if `tree` is missing, with a warning and installation advice.

- **Command-Line Options**:
  - `--help`: Usage info.
  - `--version`: Script version (e.g., 0.03a).
  - `--format <format>`: Choose output format.
  - `--pack`: Compact output.
  - Defaults: Directory to `.` (current), output file based on format (e.g., `tree.json`).

- **Interactive Warnings**:
  - Prompts to create sample `.treeignore` if missing.
  - Suggests installing `tree` on popular platforms if using fallback.

## Installation

### Prerequisites

- **Bash**: Version 3+ (compatible with macOS default; 4+ recommended for full features). Most systems have this pre-installed.
- **tree Command** (Recommended): For best ASCII output. If missing, the script uses `find` but warns you.
- **Python** (Optional, for project-specific features): If your project includes Python (e.g., for AWS CDK stacks as in the example tree), minimum version is Python 3.13. The script itself is pure Bash and doesn't require Python.

#### Installing `tree` on Popular Platforms

The `tree` command provides superior ASCII tree rendering. Install it as follows:

- **Debian/Ubuntu (Linux)**:
  ```
  sudo apt update
  sudo apt install tree
  ```

- **CentOS/RHEL/Fedora (Linux)**:
  ```
  sudo yum install tree
  # Or for dnf-based systems:
  sudo dnf install tree
  ```

- **macOS (via Homebrew)**:
  First, install Homebrew if not present:
  ```
  /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
  ```
  Then:
  ```
  brew install tree
  ```

- **Windows (via Chocolatey)**:
  Install Chocolatey if not present:
  ```
  Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
  ```
  Then, in PowerShell:
  ```
  choco install tree
  ```
  Alternatively, use WSL (Windows Subsystem for Linux) and install via apt/yum as above.

- **Arch Linux**:
  ```
  sudo pacman -S tree
  ```

After installation, verify with `tree --version`.

#### Installing Python (If Needed for Your Project)

If your project uses Python (e.g., for virtual environments or scripts like AWS CDK), install Python 3.13+:

- **Debian/Ubuntu**:
  ```
  sudo apt update
  sudo apt install python3.13 python3.13-venv
  ```

- **CentOS/RHEL**:
  Enable EPEL and build from source or use:
  ```
  sudo yum install python3
  ```
  For 3.13 specifically, you may need to compile from source: Download from [python.org](https://www.python.org/downloads/release/python-3130/), then:
  ```
  ./configure --enable-optimizations
  make -j $(nproc)
  sudo make altinstall
  ```

- **macOS (via Homebrew)**:
  ```
  brew install python@3.13
  ```

- **Windows**:
  Download from [python.org](https://www.python.org/downloads/release/python-3130/) and run the installer. Ensure "Add Python to PATH" is checked.

Verify with `python3 --version` (or `python --version` on Windows).

To set up a virtual environment (as in the example project):
```
python3 -m venv .venv
source .venv/bin/activate  # On Unix/macOS
.venv\Scripts\activate     # On Windows
```

## Usage

Run the script from your terminal:

```
./ai-file-tree-generator.sh [options] [<directory>] [<output_file>]
```

### Examples

1. **Default (grok-ascii on current directory)**:
   ```
   ./ai-file-tree-generator.sh
   ```
   Outputs to `tree.txt`.

2. **JSON format with compact output**:
   ```
   ./ai-file-tree-generator.sh --format json --pack ./my-project
   ```
   Outputs compacted JSON to `tree.json`.

3. **Markdown for a specific directory**:
   ```
   ./ai-file-tree-generator.sh --format markdown /path/to/project custom.md
   ```

4. **CSV with fallback (if tree missing)**:
   ```
   ./ai-file-tree-generator.sh --format csv
   ```

## .treeignore File

Place a `.treeignore` in the root of your directory to customize exclusions. Format:

```
# Comments start with #
.git  # Ignore .git directory
node_modules  # Ignore npm dependencies
*.log  # Ignore log files
contents:.venv  # Show .venv but ignore its contents
```

The script validates this file and handles globs, leading dots, etc.

## Contributing

Contributions welcome! Fork the repo, make changes, and submit a PR. Ensure tests pass (if added) and follow Bash best practices.

## License

MIT License. See [LICENSE](LICENSE) for details.

## Acknowledgments

- Inspired by tools like `tree`, `.gitignore`, and AI-assisted development workflows.
- Thanks to xAI for Grok integration ideas.

For issues or suggestions, open a GitHub issue.