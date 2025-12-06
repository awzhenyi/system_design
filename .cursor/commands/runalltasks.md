# Run Task Command

This command finds the latest numbered task file, locates the next incomplete task, executes it, and marks it as completed.

## Instructions for AI Agent

When this command is executed:

1. **Find the Latest Task File**
   - Use `glob_file_search` to find all task files matching pattern `task*.md` in the workspace root
   - Parse the task numbers from filenames (e.g., task001.md → 001, task010.md → 010)
   - Identify the highest numbered task file (latest task file)
   - If no task files exist, inform the user that no task files are available

2. **Read the Latest Task File**
   - Use `read_file` to read the contents of the latest task file
   - Parse the markdown structure to understand phases and tasks

3. **Find the Next Incomplete Task**
   - Scan through the file from top to bottom (maintaining phase order)
   - Look for checkbox patterns:
     - Incomplete task: `- [ ]` or `- [ ]` (with any indentation)
     - Complete task: `- [x]` or `- [X]` (with any indentation)
   - Find the **first incomplete task** encountered in the file (top-to-bottom, phase-by-phase order)
   - Extract the full task description for context
   - Note the exact line number and surrounding context for later editing

4. **Execute the Task**
   - Analyze the task description to understand what needs to be done
   - Review any relevant context:
     - The task file's overview section
     - The phase the task belongs to
     - Any previous completed tasks that might provide context
     - Open files or project structure that might be relevant
   - Execute the task using appropriate tools:
     - Create/edit files as needed
     - Run commands if required
     - Make code changes
     - Perform any other actions specified in the task
   - Verify the task has been completed successfully

5. **Mark Task as Completed**
   - After successfully executing the task, update the task file
   - Use `search_replace` to change the incomplete checkbox to completed:
     - Find the exact line with `- [ ]` for the task that was just completed
     - Replace `- [ ]` with `- [x]` for that specific task
     - Include enough context (surrounding lines) to uniquely identify the task
   - Preserve all formatting and indentation

6. **Report Completion**
   - Inform the user which task was executed and completed
   - Provide a brief summary of what was accomplished
   - Indicate if there are more incomplete tasks remaining

## Execution Steps (Detailed)

1. **Search for task files:**
   ```
   glob_file_search(glob_pattern="task*.md")
   ```

2. **Parse and find latest:**
   - Extract numbers from filenames (e.g., "task001.md" → 1, "task042.md" → 42)
   - Sort numerically and select the highest number
   - If multiple files exist, the highest number is the "latest"

3. **Read the task file:**
   ```
   read_file(target_file="task[NUMBER].md")
   ```

4. **Parse for incomplete tasks:**
   - Use regex or line-by-line parsing to find `- [ ]` patterns
   - Track line numbers and context
   - Select the first incomplete task found (top-to-bottom order)

5. **Execute the task:**
   - Understand what the task requires
   - Use appropriate tools (codebase_search, read_file, write, search_replace, run_terminal_cmd, etc.)
   - Complete the work specified in the task

6. **Update the task file:**
   ```
   search_replace(
     file_path="task[NUMBER].md",
     old_string="- [ ] [Task description]",
     new_string="- [x] [Task description]"
   )
   ```
   - Include sufficient context lines before and after to make the replacement unique

 7. **Execute the next Incomplete tasks**
    - While the task.md file still has incomplete tasks, repeat steps 3-6 until all tasks are marked as completed.
## Example Workflow

Given a task file `task001.md`:
```markdown
# Task 001: Example Project

## Phase 1: Setup
- [x] Task 1.1: Create project structure
- [ ] Task 1.2: Install dependencies
- [ ] Task 1.3: Configure build tools

## Phase 2: Implementation
- [ ] Task 2.1: Implement core feature
```

The command would:
1. Find `task001.md` as the latest (or only) task file
2. Read the file
3. Identify "Task 1.2: Install dependencies" as the next incomplete task
4. Execute the task (e.g., run `npm install` or similar)
5. Update the file to mark Task 1.2 as complete:
   ```markdown
   - [x] Task 1.2: Install dependencies
   ```

## Error Handling

- **No task files found:** Inform user and suggest running `/plantask` first
- **All tasks completed:** Inform user that all tasks in the latest task file are complete
- **Task execution fails:** Report the error, do not mark the task as complete, and suggest next steps
- **Multiple task files with same number:** Use the one with the most recent modification time

## Notes

- Tasks are processed in the order they appear in the file (top-to-bottom, maintaining phase order)
- Only one task is executed per command invocation
- The task must be successfully completed before marking it as done
- Preserve all formatting, indentation, and structure when updating checkboxes
- If a task description spans multiple lines, include all lines in the search_replace context

