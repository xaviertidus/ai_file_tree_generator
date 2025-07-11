<details>
<summary>Click to expand: ai-file-tree-generator.sh</summary>

```bash
#!/bin/bash

# Script to generate a tree-like text representation of a directory structure
# Reads exclusion patterns from .treeignore, supporting # comments and contents: prefix
# Validates .treeignore before building tree, warns on duplicates and leading dots
# Deduplicates patterns before processing
# Handles trailing slashes and leading dots in patterns
# Supports regex patterns in .treeignore for advanced matching
# Usage: ./ai-file-tree-generator.sh [options] [<directory_path>] [<output_file>]
# Options:
#   --help     Display this help message
#   --version  Display the script version
#   --format FORMAT  Output format: grok-ascii (default), json, yaml, xml, markdown, dot, csv, excel
#                    Note: excel outputs CSV format importable to Excel.
#   --pack     Compact output by stripping whitespace (for compatible formats; defaults to human-readable)
#   --ignore-file FILE  Path to custom ignore file (overrides .treeignore in directory)
# If <output_file> is not provided, defaults based on format (e.g., tree.txt for grok-ascii, tree.json for json)
# If <directory_path> is not provided, defaults to current directory "."

VERSION="0.04a"

# Improved option parsing to handle options anywhere, not just before positionals
FORMAT="grok-ascii"
PACK=false
IGNORE_FILE=""
POSITIONALS=()
while [[ $# -gt 0 ]]; do
    case "$1" in
        --help)
            echo "Usage: $0 [options] [<directory_path>] [<output_file>]"
            echo "Options:"
            echo "  --help     Display this help message"
            echo "  --version  Display the script version"
            echo "  --format FORMAT  Output format: grok-ascii (default), json, yaml, xml, markdown, dot, csv, excel"
            echo "                   Note: excel outputs CSV format importable to Excel."
            echo "  --pack     Compact output by stripping whitespace (for compatible formats; defaults to human-readable)"
            echo "  --ignore-file FILE  Path to custom ignore file (overrides .treeignore in directory)"
            echo "Defaults:"
            echo "  <directory_path> defaults to '.' if not provided"
            echo "  <output_file> defaults based on format (e.g., tree.txt for grok-ascii)"
            exit 0
            ;;
        --version)
            echo "$VERSION"
            exit 0
            ;;
        --format)
            FORMAT="$2"
            shift
            ;;
        --pack)
            PACK=true
            ;;
        --ignore-file)
            IGNORE_FILE="$2"
            shift
            ;;
        *)
            POSITIONALS+=("$1")
            ;;
    esac
    shift
done

# Assign positionals
DIRECTORY="${POSITIONALS[0]:-.}"
OUTPUT_FILE="${POSITIONALS[1]:-}"

# Set default output file based on format if not provided
if [ -z "$OUTPUT_FILE" ]; then
    case "$FORMAT" in
        grok-ascii) OUTPUT_FILE="tree.txt" ;;
        json) OUTPUT_FILE="tree.json" ;;
        yaml) OUTPUT_FILE="tree.yaml" ;;
        xml) OUTPUT_FILE="tree.xml" ;;
        markdown) OUTPUT_FILE="tree.md" ;;
        dot) OUTPUT_FILE="tree.dot" ;;
        csv|excel) OUTPUT_FILE="tree.csv" ;;
        *)
            echo "Error: Unknown format '$FORMAT'"
            exit 1
            ;;
    esac
fi

# Validate directory exists
if [ ! -d "$DIRECTORY" ]; then
    echo "Error: Directory '$DIRECTORY' does not exist."
    exit 1
fi

# Changed section: Use --ignore-file if provided, else default to .treeignore in DIRECTORY
TREEIGNORE_FILE="$DIRECTORY/.treeignore"
if [ -n "$IGNORE_FILE" ]; then
    if [ ! -f "$IGNORE_FILE" ]; then
        echo "Warning: Specified ignore file '$IGNORE_FILE' does not exist. Proceeding without exclusions."
        TREEIGNORE_FILE=""
    else
        TREEIGNORE_FILE="$IGNORE_FILE"
    fi
else
    # Warn and offer sample only if no .treeignore in DIRECTORY and no --ignore-file
    if [ ! -f "$TREEIGNORE_FILE" ]; then
        echo "Warning: No .treeignore file found in '$DIRECTORY'. It's rare to want to include all directories (e.g., .git, .DS_Store) in the tree."
        echo "Would you like to create a sample .treeignore file? (y/n)"
        read -r response
        if [[ "$response" =~ ^[Yy]$ ]]; then
            cat > "$TREEIGNORE_FILE" << EOL
# Sample .treeignore file with common exclusions
.git
.DS_Store
node_modules
__pycache__
*.log
*.gitignore
EOL
            echo "Sample .treeignore created in '$DIRECTORY'."
        fi
    fi
fi

# Validate .treeignore file and collect unique patterns (if file exists)
unique_patterns=()
if [ -f "$TREEIGNORE_FILE" ]; then
    line_number=0
    while IFS= read -r line; do
        ((line_number++))
        # Skip full comment lines
        if [[ ! "$line" =~ ^\s*# ]]; then
            # Strip inline comments
            clean_line=$(echo "$line" | sed 's/\s*#.*$//')
            # Split line into patterns (space-separated)
            IFS=' ' read -ra patterns <<< "$clean_line"
            for pattern in "${patterns[@]}"; do
                # Check for empty pattern
                if [ -z "$pattern" ] && [ -n "$line" ]; then
                    echo "Error in $TREEIGNORE_FILE at line $line_number: Empty pattern after stripping comment"
                    exit 1
                fi
                # Process non-empty patterns
                if [ -n "$pattern" ]; then
                    # Check for invalid characters (e.g., '|')
                    if [[ "$pattern" =~ \| ]]; then
                        echo "Error in $TREEIGNORE_FILE at line $line_number: Pattern '$pattern' contains invalid character '|'"
                        exit 1
                    fi
                    # Check for trailing slashes (warn, as they’re cleaned later)
                    if [[ "$pattern" =~ /+$ && ! "$pattern" =~ ^contents: ]]; then
                        echo "Warning in $TREEIGNORE_FILE at line $line_number: Pattern '$pattern' contains trailing slash, which will be ignored"
                    fi
                    # Check for leading dots (warn, as they may cause issues with find
                    if [[ "$pattern" =~ ^\..* && ! "$pattern" =~ ^contents: && ! "$pattern" =~ ^\*\. ]]; then
                        echo "Warning in $TREEIGNORE_FILE at line $line_number: Pattern '$pattern' starts with a dot, which may cause issues with find (install tree for better support)"
                    fi
                    # Check for malformed contents: patterns
                    if [[ "$pattern" =~ ^contents: ]]; then
                        dir=$(echo "$pattern" | sed 's/^contents://' | sed 's:/*$::' | sed 's:^/*::')
                        if [ -z "$dir" ]; then
                            echo "Error in $TREEIGNORE_FILE at line $line_number: Malformed 'contents:' pattern, no directory specified"
                            exit 1
                        fi
                    fi
                    # Check for duplicate patterns using array loop (warn only)
                    clean_pattern=$(echo "$pattern" | sed 's:/*$::')
                    is_duplicate=false
                    for existing in "${unique_patterns[@]}"; do
                        if [ "$existing" = "$clean_pattern" ]; then
                            is_duplicate=true
                            break
                        fi
                    done
                    if $is_duplicate; then
                        echo "Warning in $TREEIGNORE_FILE at line $line_number: Duplicate pattern '$clean_pattern'"
                    else
                        # Add to unique list
                        unique_patterns+=("$clean_pattern")
                    fi
                fi
            done
        fi
    done < "$TREEIGNORE_FILE"
fi

# Separate regular exclusions and contents exclusions
declare -a reg_excl cont_excl
for pat in "${unique_patterns[@]}"; do
    if [[ "$pat" =~ ^contents: ]]; then
        dir="${pat#contents:}"
        cont_excl+=("$dir")
    else
        reg_excl+=("$pat")
    fi
done

# Changed section: Support regex matching for exclusions
# Function to check if a name matches any regex in array
matches_regex() {
    local name="$1"
    shift
    for pat in "$@"; do
        if [[ $name =~ $pat ]]; then
            return 0
        fi
    done
    return 1
}

# Function to check if a name is regularly excluded (using regex)
is_excluded() {
    matches_regex "$1" "${reg_excl[@]}"
}

# Function to check if a name is contents excluded (show dir, hide contents) (using regex)
is_contents_excluded() {
    matches_regex "$1" "${cont_excl[@]}"
}

# For grok-ascii format, use tree or find fallback with exclusions
if [[ "$FORMAT" == "grok-ascii" ]]; then
    REGULAR_EXCLUSIONS=$(IFS='|'; echo "${reg_excl[*]}")
    CONTENTS_EXCLUSIONS=$(IFS='|'; echo "${cont_excl[*]}/*")
    # Using tree if available
    if command -v tree >/dev/null 2>&1; then
        if [ -n "$CONTENTS_EXCLUSIONS" ]; then
            temp_file=$(mktemp)
            tree -a -I "$REGULAR_EXCLUSIONS" --noreport "$DIRECTORY" > "$temp_file"
            awk -v excl="$CONTENTS_EXCLUSIONS" '
            BEGIN {
                skipping_level = -1;
                split(excl, raw_patterns, "|");
                for (i in raw_patterns) {
                    p = raw_patterns[i];
                    if (length(p) > 2) {
                        dir = substr(p, 1, length(p)-2);
                        exclude[dir] = 1;
                    }
                }
            }
            NR == 1 {
                print;
                next;
            }
            {
                copy = $0;
                level = gsub(/│/, "", copy);
                name = $0;
                sub(/^.*[├└]── /, "", name);
                show = 1;
                if (skipping_level >= 0) {
                    if (level > skipping_level) {
                        show = 0;
                    } else {
                        skipping_level = -1;
                    }
                }
                if (show && (name in exclude)) {
                    skipping_level = level;
                }
                if (show) print;
            }' "$temp_file" > "$OUTPUT_FILE"
            rm "$temp_file"
        else
            tree -a -I "$REGULAR_EXCLUSIONS" --noreport "$DIRECTORY" > "$OUTPUT_FILE"
        fi
    else
        # Warn and provide installation instructions when falling back to find
        echo "tree command not found. Using find command (output may differ slightly)."
        echo "To install tree:"
        echo "  - On Debian/Ubuntu: sudo apt install tree"
        echo "  - On macOS: brew install tree"
        echo "  - On CentOS/RHEL: sudo yum install tree"
        basename_dir=$(basename "$DIRECTORY")
        echo "$basename_dir" > "$OUTPUT_FILE"
        FIND_CMD="find \"$DIRECTORY\" -type f -o -type d"
        for pattern in "${reg_excl[@]}"; do
            clean_pattern=$(echo "$pattern" | sed 's:/*$::' | sed 's:^/*::')
            if [[ "$clean_pattern" =~ ^\*\. ]]; then
                glob_pattern=$(echo "$clean_pattern" | sed 's/^\*\./*./')
                FIND_CMD="$FIND_CMD -not -name \"$glob_pattern\""
            elif [[ "$clean_pattern" =~ ^asset\.\* ]]; then
                FIND_CMD="$FIND_CMD -not -path \"*/asset.*/*\" -not -name \"asset.*\""
            elif [[ "$clean_pattern" =~ ^\. ]]; then
                FIND_CMD="$FIND_CMD -not -path \"*/$clean_pattern/*\" -not -name \"$clean_pattern\""
            else
                FIND_CMD="$FIND_CMD -not -path \"*/$clean_pattern/*\" -not -name \"$clean_pattern\""
            fi
        done
        if [ -n "$CONTENTS_EXCLUSIONS" ]; then
            while IFS='|' read -r pattern; do
                if [ -n "$pattern" ]; then
                    dir=$(echo "$pattern" | sed 's:/*$::' | sed 's:^/*::' | cut -d'/' -f1)
                    FIND_CMD="$FIND_CMD -not -path \"*/$dir/*\""
                    [ -d "$DIRECTORY/$dir" ] && echo "├── $dir" >> "$OUTPUT_FILE"
                fi
            done <<< "$CONTENTS_EXCLUSIONS"
        fi
        eval "$FIND_CMD -printf \"├── %p\\n\" | sed \"s|$DIRECTORY/||\" | sort -u" >> "$OUTPUT_FILE"
    fi
else
    # For other formats, use recursive bash functions to build the tree
    # Adjust indentation/spacing based on $PACK
    INDENT="  "
    SUBINDENT="    "
    if $PACK; then
        INDENT=""
        SUBINDENT=""
    fi
    case "$FORMAT" in
        json)
            # Recursive function to build JSON tree
            build_json() {
                local dir="$1"
                local is_root="$2"
                local output=()
                local name=$(basename "$dir")
                if [ "$is_root" = "true" ]; then
                    output+=("{")
                else
                    output+=("\"$name\":{")
                fi
                output+=("${INDENT}\"type\":\"directory\",")
                output+=("${INDENT}\"children\":{")
                local first=true
                for item in "$dir"/{.,}*; do
                    if [[ "$item" == "$dir/." || "$item" == "$dir/.." ]]; then continue; fi
                    local iname=$(basename "$item")
                    if is_excluded "$iname"; then continue; fi
                    if ! $first; then output+=(","); fi
                    first=false
                    if [ -d "$item" ]; then
                        if is_contents_excluded "$iname"; then
                            output+=("${SUBINDENT}\"$iname\":{\"type\":\"directory\",\"children\":{}}")
                        else
                            local sub_output=($(build_json "$item" "false"))
                            output+=("${sub_output[@]}")
                        fi
                    else
                        output+=("${SUBINDENT}\"$iname\":{\"type\":\"file\"}")
                    fi
                done
                output+=("${INDENT}}")
                output+=("}")
                printf '%s\n' "${output[@]}"
            }
            build_json "$DIRECTORY" "true" | tr -d '\n ' > "$OUTPUT_FILE"  # Further compact if --pack
            if ! $PACK; then
                # If not pack, we already have indents, but for pack we remove all whitespace
                sed -i '' 's/[{,]/&\n/g' "$OUTPUT_FILE"  # But since pack removes, for non-pack keep as is
            fi
            ;;
        yaml)
            # Recursive function to build YAML tree
            build_yaml() {
                local dir="$1"
                local indent="$2"
                local name=$(basename "$dir")
                printf "%s%s:\n" "$indent" "$name"
                printf "%s%stype: directory\n" "$indent" "$INDENT"
                printf "%s%schildren:\n" "$indent" "$INDENT"
                local subindent="$indent$SUBINDENT"
                for item in "$dir"/{.,}*; do
                    if [[ "$item" == "$dir/." || "$item" == "$dir/.." ]]; then continue; fi
                    local iname=$(basename "$item")
                    if is_excluded "$iname"; then continue; fi
                    if [ -d "$item" ]; then
                        if is_contents_excluded "$iname"; then
                            printf "%s%s:\n" "$subindent" "$iname"
                            printf "%s%stype: directory\n" "$subindent" "$INDENT"
                            printf "%s%schildren: {}\n" "$subindent" "$INDENT"
                        else
                            build_yaml "$item" "$subindent"
                        fi
                    else
                        printf "%s%s:\n" "$subindent" "$iname"
                        printf "%s%stype: file\n" "$subindent" "$INDENT"
                    fi
                done
            }
            if $PACK; then
                build_yaml "$DIRECTORY" "" | sed 's/^[ \t]*//' > "$OUTPUT_FILE"
            else
                build_yaml "$DIRECTORY" "" > "$OUTPUT_FILE"
            fi
            ;;
        xml)
            # Recursive function to build XML tree
            build_xml() {
                local dir="$1"
                local indent="$2"
                local name=$(basename "$dir")
                printf "%s<directory name=\"%s\">\n" "$indent" "$name"
                local subindent="$indent$INDENT"
                for item in "$dir"/{.,}*; do
                    if [[ "$item" == "$dir/." || "$item" == "$dir/.." ]]; then continue; fi
                    local iname=$(basename "$item")
                    if is_excluded "$iname"; then continue; fi
                    if [ -d "$item" ]; then
                        if is_contents_excluded "$iname"; then
                            printf "%s<directory name=\"%s\"/>\n" "$subindent" "$iname"
                        else
                            build_xml "$item" "$subindent"
                        fi
                    else
                        printf "%s<file name=\"%s\"/>\n" "$subindent" "$iname"
                    fi
                done
                printf "%s</directory>\n" "$indent"
            }
            if $PACK; then
                build_xml "$DIRECTORY" "" | tr -d ' \t\n' > "$OUTPUT_FILE"
            else
                build_xml "$DIRECTORY" "" > "$OUTPUT_FILE"
            fi
            ;;
        markdown)
            # Recursive function to build Markdown tree
            build_markdown() {
                local dir="$1"
                local indent="$2"
                local name=$(basename "$dir")
                if [ -n "$indent" ]; then
                    printf "%s- %s\n" "$indent" "$name"
                else
                    printf "%s\n" "$name"
                fi
                local subindent="$indent$INDENT"
                for item in "$dir"/{.,}*; do
                    if [[ "$item" == "$dir/." || "$item" == "$dir/.." ]]; then continue; fi
                    local iname=$(basename "$item")
                    if is_excluded "$iname"; then continue; fi
                    if [ -d "$item" ]; then
                        if is_contents_excluded "$iname"; then
                            printf "%s- %s\n" "$subindent" "$iname"
                        else
                            build_markdown "$item" "$subindent"
                        fi
                    else
                        printf "%s- %s\n" "$subindent" "$iname"
                    fi
                done
            }
            if $PACK; then
                build_markdown "$DIRECTORY" "" | sed 's/^[ \t]*//' > "$OUTPUT_FILE"
            else
                build_markdown "$DIRECTORY" "" > "$OUTPUT_FILE"
            fi
            ;;
        dot)
            # Recursive function to build DOT (Graphviz) tree
            build_dot() {
                local dir="$1"
                local parent="$2"
                local root_name=$(basename "$DIRECTORY")
                if [ -z "$parent" ]; then
                    echo "digraph tree{"
                    parent="$root_name"
                    echo "${INDENT}\"$parent\"[label=\"$root_name\" shape=\"folder\"];"
                fi
                for item in "$dir"/{.,}*; do
                    if [[ "$item" == "$dir/." || "$item" == "$dir/.." ]]; then continue; fi
                    local iname=$(basename "$item")
                    if is_excluded "$iname"; then continue; fi
                    local child="$parent/$iname"
                    if [ -d "$item" ]; then
                        echo "${INDENT}\"$child\"[label=\"$iname\" shape=\"folder\"];"
                        echo "${INDENT}\"$parent\"->\"$child\";"
                        if ! is_contents_excluded "$iname"; then
                            build_dot "$item" "$child"
                        fi
                    else
                        echo "${INDENT}\"$child\"[label=\"$iname\"];"
                        echo "${INDENT}\"$parent\"->\"$child\";"
                    fi
                done
                if [ -z "$2" ]; then
                    echo "}"
                fi
            }
            if $PACK; then
                build_dot "$DIRECTORY" "" | tr -d ' \t\n' > "$OUTPUT_FILE"
            else
                build_dot "$DIRECTORY" "" > "$OUTPUT_FILE"
            fi
            ;;
        csv|excel)
            # Use recursive function for CSV to properly handle contents exclusions
            echo "path,type" > "$OUTPUT_FILE"
            build_csv() {
                local dir="$1"
                local prefix="$2"
                for item in "$dir"/{.,}*; do
                    if [[ "$item" == "$dir/." || "$item" == "$dir/.." ]]; then continue; fi
                    local iname=$(basename "$item")
                    if is_excluded "$iname"; then continue; fi
                    local relpath="$prefix$iname"
                    if [ -d "$item" ]; then
                        echo "\"$relpath\",directory" >> "$OUTPUT_FILE"
                        if ! is_contents_excluded "$iname"; then
                            build_csv "$item" "$relpath/"
                        fi
                    else
                        echo "\"$relpath\",file" >> "$OUTPUT_FILE"
                    fi
                done
            }
            build_csv "$DIRECTORY" ""
            ;;
    esac
fi

# Notify user of success
echo "Directory tree has been written to $OUTPUT_FILE in $FORMAT format"
```

</details>

The updated file is `ai-file-tree-generator.sh` and should replace the existing one in the project root directory.

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
  - Supports regex patterns for advanced matching (e.g., `^temp[0-9]+$` for directories like temp1, temp2).
  - "contents:" prefix to show a directory but hide its contents (e.g., `contents:.venv`).
  - Validates patterns, warns on duplicates or issues, and deduplicates automatically.
  - If no `.treeignore` exists, the script warns and offers to create a sample.

- **Custom Ignore File Path (--ignore-file)**:
  - Specify a custom ignore file location with `--ignore-file /path/to/custom.ignore`.
  - Overrides the default `.treeignore` in the target directory.
  - If provided, no warning for missing `.treeignore` in the directory.

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
  - `--version`: Script version (e.g., 0.04a).
  - `--format <format>`: Choose output format.
  - `--pack`: Compact output.
  - `--ignore-file <file>`: Custom ignore file path.
  - Defaults: Directory to `.` (current), output file based on format (e.g., `tree.json`).

- **Sample Ignore Files**:
  - For programming languages: Examples for Python, JavaScript, Java, C#, Go, Rust, Swift, Kotlin, PHP, C, C++, Perl, VBA, Visual Basic, PowerShell, Visual Studio.
  - For OS roots (user files focus): Examples for Windows C:\, macOS /, Ubuntu /.
  - See the [samples directory](samples/) in the repo for these files and their READMEs.

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

2. **JSON format with compact output and custom ignore**:
   ```
   ./ai-file-tree-generator.sh --format json --pack --ignore-file /path/to/custom.ignore ./my-project
   ```
   Outputs compacted JSON to `tree.json`.

3. **Markdown for a specific directory**:
   ```
   ./ai-file-tree-generator.sh --format markdown /path/to/project custom.md
   ```

## .treeignore File

Place a `.treeignore` in the root of your directory to customize exclusions. Format:

```
# Comments start with #
.git  # Ignore .git directory
node_modules  # Ignore npm dependencies
*.log  # Ignore log files (glob)
^temp[0-9]+$  # Regex: Ignore temp1, temp2, etc.
contents:.venv  # Show .venv but ignore its contents
```

The script validates this file and handles globs, leading dots, and now regex patterns (using Bash =~ matching).

## Contributing

Contributions welcome! Fork the repo, make changes, and submit a PR. Ensure tests pass (if added) and follow Bash best practices.

## License

MIT License. See [LICENSE](LICENSE) for details.

## Acknowledgments

- Inspired by tools like `tree`, `.gitignore`, and AI-assisted development workflows.
- Thanks to xAI for Grok integration ideas.

For issues or suggestions, open a GitHub issue.