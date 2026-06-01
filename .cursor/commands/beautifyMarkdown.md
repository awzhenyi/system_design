# Beautify Markdown Command

This command rewrites a target Markdown file so the same information is presented more clearly with better headings, sections, lists, spacing, and emphasis.

## Instructions for AI Agent

When this command is executed:

1. **Identify the Target Markdown File**
   - Use the file explicitly provided by the user, such as `@path/to/file.md`.
   - If no file is provided, use the currently open Markdown file when the context clearly identifies one.
   - If the target file is unclear, ask the user which Markdown file should be beautified.
   - Only operate on Markdown files with `.md` or `.mdx` extensions.

2. **Read and Understand the Content**
   - Read the full target file before editing.
   - Determine the intended topic, audience, and existing structure.
   - Preserve the original meaning, technical claims, examples, links, code blocks, and important terminology.
   - Do not invent new facts unless the user explicitly asks for content expansion.

3. **Improve the Presentation**
   - Add or improve a clear title using `#`.
   - Organize content into logical sections using `##` and `###`.
   - Convert dense paragraphs into concise bullet lists where it improves readability.
   - Use numbered lists for ordered processes, workflows, or step-by-step instructions.
   - Use tables only when they clearly improve comparison or reference readability.
   - Add short introductory paragraphs for major sections when helpful.
   - Use bold text sparingly to highlight key terms or important takeaways.
   - Ensure consistent spacing between headings, paragraphs, lists, tables, and code blocks.

4. **Clean Up Markdown Formatting**
   - Normalize heading hierarchy so levels do not jump unnecessarily.
   - Remove duplicate blank lines while keeping Markdown easy to scan.
   - Fix inconsistent list indentation and nested bullet formatting.
   - Keep code fences intact and preserve their language tags when present.
   - Preserve images, links, frontmatter, admonitions, MDX components, and Docusaurus-specific syntax.
   - Do not reformat code inside fenced code blocks unless the user explicitly requests code formatting.

5. **Respect the Existing Content**
   - Keep the edit focused on readability and structure.
   - Do not delete substantive content unless it is an obvious duplicate.
   - Do not rename files or move content to another file.
   - Do not make broad stylistic rewrites that change the author's voice more than necessary.
   - If the file appears to be generated, ask before editing.

6. **Apply Design Document Sidebar Position**
   - If the target file is under a high-level design or low-level design folder, ensure Docusaurus frontmatter has the correct `sidebar_position`.
   - Treat folder names case-insensitively and match common variants such as `highleveldesign`, `high level design`, `lowleveldesign`, and `low level design`.
   - If the filename is `requirements.md`, set `sidebar_position: 1`.
   - If the filename is `entities.md`, `core_entities.md`, `core-entities.md`, `core entities.md`, or a similar entity/core entity filename, set `sidebar_position: 2`.
   - If frontmatter already exists, update or add only the `sidebar_position` field while preserving other frontmatter fields.
   - If frontmatter does not exist, add it at the top of the file.
   - Do not apply this rule to non-design folders or unrelated filenames.

7. **Write the Beautified File**
   - Update the target file in place.
   - Prefer direct Markdown edits over automated global replacements.
   - Keep the final document well structured with headings, sections, concise bullets, and readable spacing.

8. **Report Completion**
   - Briefly summarize the main formatting improvements.
   - Mention any content that was intentionally left unchanged, such as code blocks, links, frontmatter, or MDX components.
   - If no changes were needed, tell the user the file was already well structured.

## Beautification Checklist

- The document has a clear title.
- Major ideas are grouped into logical sections.
- Heading levels are consistent.
- Lists are concise and consistently indented.
- Long paragraphs are split where helpful.
- Important terms are emphasized without overusing bold text.
- Design `requirements.md` and entity files have the expected `sidebar_position` when applicable.
- Code blocks, links, images, frontmatter, and MDX syntax are preserved.
- The final Markdown is easier to scan without changing the original meaning.

## Example Workflow

Given a Markdown file with unstructured notes:

```markdown
redis is an in-memory datastore. it supports strings hashes lists sets sorted sets. it is often used for cache sessions counters queues.
```

The command may rewrite it as:

```markdown
# Redis

Redis is an in-memory data store commonly used for low-latency workloads.

## Core Data Types

- Strings
- Hashes
- Lists
- Sets
- Sorted sets

## Common Use Cases

- Caching
- Session storage
- Counters
- Queues
```

## Error Handling

- **No Markdown file provided:** Ask the user to specify the file to beautify.
- **Target is not Markdown:** Inform the user that this command only applies to `.md` and `.mdx` files.
- **File cannot be read:** Report the issue and do not attempt edits.
- **Ambiguous target:** Ask for clarification instead of guessing.
