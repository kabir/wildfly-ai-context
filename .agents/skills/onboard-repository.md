---
name: onboard-repository
description: Onboard a new satellite repository into the WildFly AI Context Hub by fetching its llms.txt and updating all hub files
---

## Input

The user provides the URL of the new repository's root `llms.txt` file.

## Process

Work through each step below. Create a task checklist to track progress.

### Step 1: Fetch and analyze the remote llms.txt

- Use WebFetch to retrieve the `llms.txt` from the provided URL.
- If the fetch fails or the URL returns a 404, the repository likely hasn't been prepared yet. Suggest the user run the `prepare-repository` skill first to create the `llms.txt` and `AGENTS.md` in the target repository, then push to the appropriate branch before retrying.
- Parse it to understand:
  - Repository name and purpose
  - Components and subsystems it provides
  - Key directories and file pointers
  - Dependencies on other ecosystem repositories
  - Any SPI contracts it defines or consumes
- Summarize what you found to the user before proceeding.

### Step 2: Determine repository classification

Read the current hub `llms.txt` to understand the existing sections.

- Decide which section the repository belongs in:
  - **Core & Runtime Repositories** — part of the server runtime
  - **Ecosystem Tooling** — build-time, developer, or operational tooling
  - A new section if neither fits
- If the repository was already listed as a placeholder in "Ecosystem Tooling (Placeholders/Future)", it will be moved out of that section.

### Step 3: Update hub `llms.txt`

- Add a properly formatted entry to the appropriate section:
  - Repository name as the link text
  - URL to its `llms.txt` as the link target
  - A concise description (1-2 sentences) covering what the repository provides and its ecosystem role
- If replacing a placeholder, remove the old placeholder text.
- If the "Ecosystem Tooling (Placeholders/Future)" section becomes empty after removing a placeholder, remove the section or rename it.

### Step 4: Update `architecture/ecosystem-topology.md`

- Add or update a subsection for the new repository describing its components and role.
- Update the component hierarchy ASCII diagram if the repository introduces a new architectural layer or belongs to an existing layer that isn't yet represented accurately.
- Document any build/provisioning flow interactions.
- Document any runtime interactions.
- Keep the style and level of detail consistent with existing entries.

### Step 5: Update `architecture/repository-boundaries.md`

- Add an ownership subsection under "Ownership Rules" defining what belongs in the new repository. Include a one-line **Test** heuristic like the existing entries have.
- Add SPI contract documentation if the repository defines or consumes contracts with other ecosystem repositories.
- Update the dependency direction diagram if the new repository introduces new dependency edges.
- Add any "violations to watch for" specific to this repository.

### Step 6: Update `README.md`

- Update the satellite repositories table: change the repository's status from "Planned" to "Indexed", add a link to its GitHub repository, and verify the description is accurate.
- If the repository is not already in the table, add a new row.

### Step 7: Review and summarize

- Re-read all modified files to verify internal consistency:
  - Links in `llms.txt` are well-formed
  - Repository names are consistent across all files
  - The dependency direction diagram in `repository-boundaries.md` is acyclic
  - Cross-references between files are correct
- Summarize all changes to the user.
- Flag any ambiguities or areas where the remote `llms.txt` didn't provide enough information to fill in a section confidently — the user may need to provide additional context.
