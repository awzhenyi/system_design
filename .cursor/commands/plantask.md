# Plan Task Command

This command breaks down a planning task into phases and tasks, then creates a numbered task markdown file with checkboxes.

## Instructions for AI Agent

When this command is executed in plan mode:

1. **Understand the Planning Context**
   - Analyze the user's request or the current task/context
   - Review any open files or recent context that might inform the planning
   - Identify the main objective and scope of the planning task

2. **Break Down into Phases and Tasks**
   - Organize the work into logical phases (e.g., Phase 1: Setup, Phase 2: Implementation, Phase 3: Testing)
   - Within each phase, break down into specific, actionable tasks
   - Ensure tasks are:
     - Specific and clear
     - Actionable (can be completed)
     - Ordered logically (dependencies considered)
     - Measurable (can be marked as complete)

3. **Determine Next Task Number**
   - Search for existing task files matching pattern `task*.md` in the workspace root
   - Find the highest numbered task file (e.g., if task001.md and task003.md exist, next is task004.md)
   - If no task files exist, start with task001.md
   - Use zero-padded 3-digit format (task001.md, task002.md, ..., task010.md, etc.)

4. **Create Task File**
   - Create a new markdown file with the determined task number (e.g., `task001.md`)
   - Include the following structure:
     - Title: `# Task [NUMBER]: [Brief Description]`
     - Date created (optional but helpful)
     - Overview/Objective section
     - Phases section with subsections for each phase
     - Each task should be a checkbox: `- [ ] Task description`
     - Use proper markdown hierarchy (## for phases, ### for sub-phases if needed, - [ ] for tasks)
   - Format example:
     ```markdown
     # Task 001: [Project Name/Description]

     **Created:** [Date]
     
     ## Overview
     [Brief description of what this task planning covers]
     
     ## Phase 1: [Phase Name]
     - [ ] Task 1.1: [Description]
     - [ ] Task 1.2: [Description]
     - [ ] Task 1.3: [Description]
     
     ## Phase 2: [Phase Name]
     - [ ] Task 2.1: [Description]
     - [ ] Task 2.2: [Description]
     
     ## Phase 3: [Phase Name]
     - [ ] Task 3.1: [Description]
     - [ ] Task 3.2: [Description]
     ```

5. **Execution Steps**
   - First, use `glob_file_search` to find existing task files: `glob_file_search(glob_pattern="task*.md")` under CursorTasks folder.
   - Determine the next task number from the results
   - Analyze the planning context from user input and open files
   - Create a structured breakdown with phases and tasks
   - Use `write` tool to create the new task file with proper formatting
   - Confirm the task file creation to the user

## Example Output Structure

The generated task file should follow this structure:

```markdown
# Task 001: [Project Title]

**Created:** [Current Date]

## Overview
[1-2 paragraph description of the planning scope and objectives]

## Phase 1: [Phase Name]
- [ ] Task description 1
- [ ] Task description 2
- [ ] Task description 3

## Phase 2: [Phase Name]
- [ ] Task description 1
- [ ] Task description 2

## Phase 3: [Phase Name]
- [ ] Task description 1
- [ ] Task description 2
- [ ] Task description 3
```

## Notes
- Tasks should be granular enough to be completable but not so granular as to be trivial
- Consider dependencies between tasks and phases
- Use clear, action-oriented language for task descriptions
- The checkboxes allow users to track progress by marking tasks as complete: `- [x]` for completed tasks

