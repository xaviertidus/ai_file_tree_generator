# README.md for Windows C:\ .treeignore (User Files Focus)

## Overview

This `.treeignore` file is tailored for scanning the C:\ drive on Windows systems using tools like the `ai-file-tree-generator` script (or similar tree-generating utilities). Its primary goal is to exclude non-user-related files and directories, focusing the output on "user files" such as personal documents, downloads, and profile data under `C:\Users`. This helps in generating clean directory trees for backups, audits, sharing with AI assistants, or documentation without clutter from system files, programs, or caches.

By using this ignore file, the tree output will primarily highlight user-generated content while hiding or partially showing system-heavy areas. It's especially useful for privacy (e.g., avoiding exposure of system configs) or efficiency (e.g., reducing output size).

**Note**: Windows file systems can be sensitive—run scripts with appropriate permissions (e.g., as admin if needed). Always back up before automating scans.

## Features and Design

- **Focus on User Files**: Prioritizes paths like `C:\Users\<username>\Documents`, `Downloads`, `Pictures`, etc., by excluding system roots.
- **Partial Exclusions with `contents:`**: For directories like `AppData` (which contains user caches but is often not purely "user content"), it shows the folder but ignores its contents.
- **Common Windows Patterns**: Handles system files (e.g., `pagefile.sys`), hidden recycle bins, program installations, and temp areas.
- **Wildcards and Globs**: Uses patterns like `Program*` to catch variations (e.g., `Program Files`, `Program Files (x86)`).
- **Assumptions**:
  - Assumes a standard Windows installation (e.g., Windows 10/11) with default drive layout.
  - "User files" are defined as non-system, personal data; if you consider installed programs as "user," adjust exclusions.
  - Ignores all of `Windows` and `Program Files`—customize if you need partial inclusion (e.g., show but hide contents).
  - Does not account for custom partitions or non-standard setups (e.g., relocated Users folder).
  - Compatible with tools supporting `.gitignore`-like patterns, but optimized for `ai-file-tree-generator`'s features (e.g., `contents:` prefix).
  - No handling for encrypted/protected folders (e.g., BitLocker); access issues may occur.

If your definition of "user files" differs (e.g., include certain Program Files subdirs), edit the file accordingly.

## Usage

1. **Placement**: Copy the `.treeignore` file to the root of the drive you're scanning (e.g., `C:\.treeignore`). If using a subdirectory, place it there and adjust relative paths.

2. **With ai-file-tree-generator Script**:
   - Ensure the script is in your PATH or run it directly.
   - Example command to generate a tree:
     ```
     ./ai-file-tree-generator.sh --format grok-ascii C:\
     ```
     This outputs to `tree.txt`, focusing on user-relevant parts.
   - For JSON output:
     ```
     ./ai-file-tree-generator.sh --format json --pack C:\
     ```
   - If no `.treeignore` exists, the script will prompt to create a sample— you can replace it with this one.

3. **Customization**:
   - Add patterns for specific exclusions (e.g., `OneDrive` if synced).
   - Use `contents:` for folders you want to acknowledge but not detail (e.g., `contents:Users/*/Desktop` to hide desktop contents).
   - Test with a dry run: Run the script and review output before final use.

4. **Limitations**:
   - Windows paths are case-insensitive, but patterns here are literal—works fine in most tools.
   - Large drives may take time; consider scanning subdirs like `C:\Users` instead.
   - Hidden/system files require admin rights to scan fully.
   - Not for non-Windows drives; adapt for D:\ or external volumes.

## Sample .treeignore Content

For reference, here's the content of the `.treeignore` file (copy-paste into your file):

<details>
<summary>Click to expand: .treeignore</summary>

```
# Ignore core Windows system directories
Windows
PerfLogs
System Volume Information
$RECYCLE.BIN
$SysReset
Recovery

# Ignore program installation directories
Program Files
Program Files (x86)
ProgramData

# Ignore hidden system files (paging, hibernation, etc.)
pagefile.sys
hiberfil.sys
swapfile.sys

# Ignore temporary and cache directories
Temp
tmp
*.tmp
*.log

# Ignore boot and config files
bootmgr
BOOTNXT
bootsect.bak

# For user profiles: Show C:\Users, but hide contents of AppData (caches, temps)
contents:Users/*/AppData

# Ignore other common non-user areas
Intel
AMD
NVIDIA
Drivers

# Ignore IDE/editor files if present at root (unlikely, but common in user dirs)
.vscode
.idea

# Ignore macOS-like files if cross-platform (e.g., .DS_Store)
*.DS_Store

# Note: Add custom patterns below as needed
# Example: IntelOptics  # Vendor-specific caches
```

</details>

## Installation and Dependencies

- **ai-file-tree-generator Script**: Download from its repo (if using). No install needed—it's a standalone Bash script. On Windows, run via Git Bash, WSL, or Cygwin.
- **tree Command** (Optional but Recommended): For better ASCII output.
  - On Windows (via Chocolatey): `choco install tree`
  - Via Scoop: `scoop install tree`
  - If using WSL: `sudo apt install tree`
- **Permissions**: Run as admin for full C:\ access: Right-click Command Prompt/Git Bash > Run as administrator.

## Contributing or Feedback

If you have suggestions (e.g., more patterns for enterprise Windows setups), feel free to adapt or share improvements. This is a community-inspired config—test on your system!

## License

This .treeignore and README are provided under the MIT License. Use freely!