# Obsidian Knowledge Vault - Agent Guidelines

This document provides guidelines for agents working with this Obsidian vault to ensure consistency with the user's organizational practices.

## Vault Overview

This is a personal knowledge vault focused on:
- Book notes and summaries (primary focus)
- Blog/article insights and takeaways
- Personal ideas and project concepts
- Technical and non-technical reflections

## Folder Structure Guidelines

### Primary Folders
1. **`/books`** - Book-related content
    - Each book gets its own subfolder
    - Subfolder naming: `book-title-in-kebab-case` or `author-book-title`
    - Required subfolders within each book folder:
        - `/source/` - Canonical source files, usually the original PDF or source material
        - `/chapter/` - Contains individual chapter notes
        - `/attachments/` or `/attatchments/` - Stores PDFs, images, and other files
        - `/etc/` - Contains supplementary markdown files explaining topics
    - Human-written content belongs in the book root, alongside `chapter/`, `etc/`, and `attatchments/` when used.
    - Do not create a separate `/ai/` directory.
    - Append AI-generated output as a `TL;DR (AI Summary)` section at the end of the relevant `chapter/` or `etc/` note.
    - Store the canonical source PDF in `/source/` and reference that same file from notes that need it.
    - Root level files:
        - `INIT.md` - Brief description of the book with links to chapters (table of contents)
    - Example: `/books/designing-data-intensive-application/`
    
2. **`/blog`** - Blog and article insights
    - Organized by topic, publication, or author
    - Subfolder examples: `/blog/martin-fowler/`, `/blog/distributed-systems/`
    - Each insight gets its own note

3. **`/personal-things`** - Personal reflections, goals, life notes
    - Non-technical personal content
    - Goals, travel, hobbies, etc.

4. **`/personal-web`** - Web development ideas, snippets, experiments
    - Code snippets, web concepts, frontend/backend ideas
    - May include small projects or prototypes

### Folder Creation Rules
- Use lowercase letters, numbers, and hyphens only
- Use descriptive but concise names
- Avoid special characters and spaces
- Create subfolders only when meaningful grouping exists

## Tagging System Guidelines

Tags are used for cross-referencing and should follow these conventions:

### Topic Tags (Primary categorization)
- Format: `#topic-name` (kebab-case)
- Examples: `#distributed-systems`, `#software-architecture`, `#productivity`
- Use existing tags when applicable, create new ones for distinct topics

### Content Type Tags
- Format: `#content-type`
- Examples: `#summary`, `#idea`, `#question`, `#quote`, `#highlight`
- Helps quickly identify note purpose

### Status Tags
- Format: `#status`
- Examples: `#in-progress`, `#completed`, `#to-read`, `#reviewed`
- Tracks reading/consumption progress

### Format Tags (Optional)
- Format: `#format-type`
- Examples: `#pdf`, `#article`, `#video`, `#podcast`
- Indicates source material type

### Tag Usage Rules
- Apply 2-4 tags per note minimum (topic + content type + status)
- Use existing tags from the vault's tag graph when possible
- Create new tags only for genuinely new categories
- Tag consistently - if you use `#distributed-systems` for one note, use it for similar topics

## Note Creation Guidelines

### Book Notes
For each book:
1. Create a folder under `/books/` using kebab-case naming
2. Create an `INIT.md` in the folder (not README.md) with:
    - Book title and author
    - Publication year
    - Brief description/why you're reading it
    - Overall rating/reflection (optional)
    - Table of contents linking to all chapter notes using [[]] syntax
3. Create chapter notes in the `/chapter/` subfolder:
    - Naming: `CHAPTER-X-[Topic].md` (use hyphens, not underscores)
    - Focus on key takeaways, quotes, and brief reflections
    - Store detailed explanations in `/etc/` if needed
4. Store attachments (PDFs, images, diagrams) in the `/attatchments/` subfolder (note spelling with double 't')
5. Create supplementary explanations in the `/etc/` subfolder for complex topics that would clutter chapter notes
6. Tag appropriately:
    - At least one topic tag (e.g., `#distributed-systems`)
    - `#summary` or `#notes`
    - Reading status (e.g., `#in-progress` or `#completed`)

### Blog/Article Notes
1. Place in appropriate `/blog/` subfolder or create new one
2. Note should include:
    - Original title and author/publication
    - Date read
    - Main thesis/argument (2-3 sentences)
    - Key takeaways (bullet points)
    - Personal assessment/value rating
    - Link to original source
3. Tagging:
    - Topic tag(s)
    - `#summary` or `#insight`
    - `#article` or `#blog`
    - Status tag if relevant

### Idea Notes
1. Location depends on nature:
    - Technical ideas: `/personal-web/` or relevant topic folder
    - Personal/project ideas: `/personal-things/`
    - Book-related ideas: in the relevant book folder
2. Note should include:
    - Clear title describing the idea
    - Description of the concept
    - Potential applications or use cases
    - Related existing ideas/notes (with links)
    - Next steps or actions (if actionable)
3. Tagging:
    - `#idea` as primary content type
    - Relevant topic tags
    - Status tags (e.g., `#to-explore`, `#in-progress`)

### Linking Practices
- Use double brackets `[[Note Title]]` to link to related notes
- Link liberally - create connections between concepts
- When mentioning a concept that has its own note, link to it
- Use [[]] links even for notes that don't exist yet (creates placeholder)
- Consider using [[]] links for books, authors, concepts mentioned

## Maintenance Practices

### Regular Reviews
- Periodically review `#in-progress` tags to update status
- Check for orphaned notes (no links) and integrate them
- Review and consolidate duplicate or similar tags
- Update book folders with final reflections upon completion

### Vault Hygiene
- Keep attachment folders organized
- Remove temporary or test notes regularly
- Ensure consistent naming conventions
- Back up vault regularly (user handles this externally)

## Agent-Specific Instructions

When working in this vault:
1. Always check existing folder structure before creating new folders
2. Review existing tags before creating new ones
3. Follow established naming conventions
4. When adding content to existing folders, match the established format
5. Link to related content generously using [[]] syntax
6. Preserve the hybrid folder/tag approach - don't abandon folders for tags-only
7. Focus on book notes as primary content type when unsure
8. Maintain the balance between structure (folders) and discoverability (tags)
9. When in doubt, follow the examples in existing notes
10. Respect the personal nature - don't reorganize unless explicitly asked

Last Updated: April 2026
