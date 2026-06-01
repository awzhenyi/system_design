# Active Recall Command

This command turns a technical link into a structured Markdown revision page for active recall.

## Instructions for AI Agent

When this command is executed:

1. **Identify the Source Link**
   - Use the URL explicitly provided by the user.
   - If no URL is provided, ask the user to provide the technical article, documentation page, video transcript, or source link.
   - If multiple URLs are provided, process each one separately unless the user asks for a combined page.
   - Prefer authoritative source content over search snippets.

2. **Read the Source Content**
   - Fetch and read the linked content before writing notes.
   - If the link cannot be fetched directly:
     - Try to locate an accessible version using the page title, domain, or visible metadata.
     - If the content is still unavailable, ask the user to paste the content or provide another accessible link.
   - Preserve the technical meaning of the source.
   - Do not invent details that are not present in the source unless clearly marked as background context.

3. **Determine the Relevant Destination Folder**
   - Create the Markdown page in the most relevant subfolder of the repository.
   - Use the existing documentation structure under the workspace, especially:
     - `website/web/highleveldesign/` for system design concepts, distributed systems, architecture, infrastructure, scalability, databases, caching, queues, APIs, and design examples.
     - `website/web/lowleveldesign/` for object-oriented design, design patterns, class modeling, and low-level design examples.
     - `website/web/trivia/` for concise computer science fundamentals, networking, operating systems, security, and database trivia.
   - Match existing topic folders and naming conventions when possible.
   - If the best destination is ambiguous, choose the closest existing folder and mention the choice in the final response.
   - Use a clear title-case filename, such as `Consistent Hashing.md` or `Redis Streams.md`.

4. **Create an Active Recall Page**
   - The page should help the user quickly revise and reconstruct the full topic from memory.
   - Focus on recall cues, key ideas, tradeoffs, mechanisms, and interview-relevant reasoning.
   - Keep only the recall cue visible.
   - Put the answer and detailed explanation inside expandable Markdown details blocks.
   - Use the recall question or cue as the `<summary>` text so the answer is hidden until expanded.
   - Do not place the answer in a normal bullet before the `<details>` block.
   - Prefer standalone `<details>` blocks over nested details inside list items because some Markdown renderers expose nested content.
   - Use plain Markdown that works in Docusaurus.

5. **Use This Markdown Structure**
   - Add frontmatter when appropriate for nearby Docusaurus pages.
   - Use this general structure:

     ```markdown
     ---
     sidebar_position: [number if appropriate]
     ---

     # [Topic Title]

     ## Source

     - [Source title](https://example.com)

     ## Active Recall

     <details>
     <summary><strong>Question or cue?</strong></summary>

     Detailed explanation, examples, tradeoffs, or reasoning.

     </details>

     <details>
     <summary><strong>Another key point to recall</strong></summary>

     Expanded explanation.

     </details>

     ## Quick Revision Checklist

     - [ ] Recall the main problem this concept solves.
     - [ ] Explain the core mechanism without looking.
     - [ ] Compare the main tradeoffs.
     - [ ] Describe one practical example or interview scenario.
     ```

6. **Write High-Quality Recall Points**
   - Use recall-friendly prompts instead of passive summaries.
   - Prefer questions, incomplete statements, or strong cue phrases:
     - `What problem does X solve?`
     - `How does X work internally?`
     - `When would you choose X over Y?`
     - `What are the main bottlenecks or failure modes?`
     - `What tradeoff is being made?`
   - Keep each `<summary>` short enough to scan during revision.
   - Put the full explanation, example, and nuance inside the matching `<details>` block.
   - Include diagrams only when the source or topic clearly benefits from them, using Mermaid if supported by the surrounding docs.
   - Include formulas, pseudocode, or code snippets only when they are central to recall.

7. **Organize by Learning Flow**
   - Start with the problem, motivation, or mental model.
   - Then cover the core mechanism.
   - Then cover design tradeoffs, scaling behavior, consistency, reliability, failure modes, or performance.
   - End with interview framing, examples, or common pitfalls when useful.
   - Use sections such as:
     - `## Active Recall`
     - `## Core Mechanism`
     - `## Tradeoffs`
     - `## Failure Modes`
     - `## Interview Notes`
     - `## Quick Revision Checklist`

8. **Respect Existing Repository Style**
   - Keep Markdown well structured with appropriate headers, concise bullets, and nested details only where useful.
   - Follow nearby files for title style, frontmatter, and folder placement.
   - Do not modify unrelated files.
   - Do not overwrite an existing page unless the user explicitly asks to update that page.

9. **Report Completion**
   - Tell the user where the new Markdown page was created.
   - Briefly summarize the topic and how the notes are organized.
   - Mention if any part of the source could not be accessed or had to be inferred from available content.

## Quality Checklist

- The source link was read before writing the notes.
- The page is in the most relevant existing documentation folder.
- The visible `<summary>` lines work as active recall prompts.
- Detailed explanations are hidden inside `<details>` blocks.
- The page has a clear title and logical sections.
- Tradeoffs, edge cases, and interview-relevant points are included when applicable.
- The final Markdown is concise enough for revision but detailed enough when expanded.

## Error Handling

- **No link provided:** Ask the user to provide the link.
- **Link inaccessible:** Ask the user to paste the content or provide another source.
- **Destination unclear:** Choose the closest existing folder and explain the choice.
- **Existing file conflict:** Ask before overwriting, or create a clearly disambiguated filename.
