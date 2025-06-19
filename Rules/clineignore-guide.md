---
description: Explains the purpose and usage of the .clineignore file for excluding files and directories from Cline's analysis.
author: Cline
version: 1.0
globs: [".clineignore"]
tags: ["configuration", "clineignore", "context-management", "best-practices"]
---

## .clineignore File Guide

### Overview

The `.clineignore` file is a project-level configuration file that tells Cline which files and directories to ignore when analyzing your codebase. Similar to `.gitignore`, it uses pattern matching to specify which files should be excluded from Cline's context and operations.

### Purpose

* **Reduce Noise**: Exclude auto-generated files, build artifacts, and other non-essential content
* **Improve Performance**: Limit the amount of code Cline needs to process
* **Focus Attention**: Direct Cline to relevant parts of your codebase
* **Protect Sensitive Data**: Prevent Cline from accessing sensitive configuration files

### Example .clineignore File

```
# Dependencies
node_modules/
**/node_modules/
.pnp
.pnp.js

# Build outputs
/build/
/dist/
/.next/
/out/

# Testing
/coverage/

# Environment variables
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# Large data files
*.csv
*.xlsx
```
