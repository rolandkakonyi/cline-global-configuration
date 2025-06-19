# Consolidated Learnings

## Tool Usage: `replace_in_file`

### Prepending Content
- **Pattern:** When prepending content to a file using `replace_in_file`, the `SEARCH` block must accurately reflect the *current* first few lines of the file.
- **Empty Files:** If the target file is empty, the `SEARCH` block should also be empty.

### Adding YAML Frontmatter
- **Pattern:** Ensure the `---` delimiters for YAML frontmatter are placed at the very beginning of the file.
- **Validation:** The content between the `---` delimiters must be valid YAML.

## Process Improvements

### Verifying Tool Actions
- **Guideline:** Always double-check the `final_file_content` provided in the tool result after a file modification. This confirms the change was applied as expected.
- **Timing:** This check is especially crucial before moving to the next file or concluding a multi-step task.
- **Error Handling:** If a discrepancy is found between the expected and actual `final_file_content`, address it immediately. This usually involves adjusting the `SEARCH` block for the next attempt if using `replace_in_file`.

## Development Guide Best Practices

### Client-Server Interaction Testing
- **Pattern:** When documenting client-server interactions, suggesting a minimal, local server (like an echo server) for initial client-side testing is a good practice to include in development guides.
- **Rationale:** This helps isolate client logic from server complexities and provides a controllable environment for debugging basic client functionality (connection, tool listing, invocation, response handling).

### Referencing Official Specifications
- **Pattern:** When creating or updating technical documentation that refers to a standard or protocol (e.g., MCP), always ensure a direct link to the latest official specification document is included.
- **Rationale:** This provides users with a canonical source for detailed information and ensures they have access to the most up-to-date and authoritative version of the specification.
