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
