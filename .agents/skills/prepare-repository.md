---
name: prepare-repository
description: Prepare a satellite repository for onboarding into the WildFly AI Context Hub by creating its llms.txt and AGENTS.md with cross-repo routing
---

## Input

The user provides:
- The local filesystem path to the target repository checkout (e.g. `../wildfly-glow`)

## Process

This skill operates on files in a different repository than the one where this skill is defined. All file reads and writes target the provided repository path.

Work through each step below. Create a task checklist to track progress.

### Step 1: Explore the target repository

- Read the repository's top-level directory listing to understand its structure.
- Look for existing `AGENTS.md`, `CLAUDE.md`, `llms.txt`, `README.md`, and documentation directories.
- Identify what the repository does by reading its README or build files.
- Look for existing documentation — common locations include `docs/`, `src/main/asciidoc/`, `src/docs/`, or documentation embedded in source directories.

### Step 2: Locate documentation

If the repository has an obvious documentation directory, use it. If it's unclear, ask the user where the documentation lives or where `llms.txt` should be placed.

Consider:
- Some repositories have documentation in a dedicated `docs/` tree (like WildFly's `docs/src/main/asciidoc/`). However, it could be anywhere in the source tree.
- Some repositories have documentation at the root level (like WildFly Core's `llms.txt` at repo root)
- Some repositories may have no existing documentation — in that case, the root is the right place for `llms.txt`

### Step 3: Create `llms.txt`

Create a root `llms.txt` that indexes the repository's components. Follow this format:

```
# {Repository Name}

> {One-paragraph description of what this repository does and what this index covers.}

## {Section heading}

- [{Document or component title}]({relative-path-to-file}): {One-line description}
```

Guidelines:
- The opening blockquote should explain what the repository is and what the index covers.
- Group entries into logical sections.
- Link to actual documentation files, key source directories, or configuration files that an agent would need.
- If the repository has subdirectories with their own substantial documentation, create sub-level `llms.txt` files in those directories and link to them from the root `llms.txt` (following the pattern WildFly uses with `_admin-guide/llms.txt`, `_developer-guide/llms.txt`, etc.).
- Keep descriptions concise — one line per entry.

### Step 4: Create or update `AGENTS.md`

If `AGENTS.md` does not exist, create it. If it already exists, add the "Ecosystem Context & Cross-Repo Routing" section to it.

The file should include at minimum:
1. A header describing what the repository is and how agents should navigate it.
2. An "Ecosystem Context & Cross-Repo Routing" section (see below).

#### Ecosystem Context & Cross-Repo Routing

Read the hub's `llms.txt` (at the root of this repository — `wildfly-ai-context`) to determine which other satellite repositories exist and what they own. Then write the routing section with two parts:

**Local Tasks** — Describe what tasks are local to this repository and link to the local `llms.txt`. Use a raw GitHub URL on the `ai-index` branch for the link (matching the pattern used in the hub's `llms.txt`). Example:

```
- **Local Tasks:** For {description of what this repo owns}, consult the local [{Repo Name} Documentation Index]({raw-github-url-to-llms.txt}).
```

**Cross-Repository Tasks** — Link to the hub and list specific task triggers that route to other repositories. Tailor the triggers to be relevant from the perspective of someone working in *this* repository. Example:

```
- **Cross-Repository Tasks:** For changes involving upstream/downstream components, consult the [WildFly Central AI Hub](https://raw.githubusercontent.com/kabir/wildfly-ai-context/main/llms.txt) and look up the target project:
    - *{specific task description}* → Navigate to **{Target Repo Name}**.
    - *{specific task description}* → Navigate to **{Target Repo Name}**.
```

The task triggers should reflect the actual dependency relationships. For example:
- A tooling repo (like Glow or Maven Plugin) would route to WildFly/WildFly Core for runtime questions and to each other for tooling integration questions.
- A runtime repo would route to tooling repos for provisioning questions and to the other runtime repo for SPI boundary questions.

### Step 5: Create `CLAUDE.md` symlink

If `CLAUDE.md` does not exist, create it as a symlink to `AGENTS.md`:

```
ln -s AGENTS.md CLAUDE.md
```

If `CLAUDE.md` already exists and is not a symlink to `AGENTS.md`, ask the user how to handle it — they may want to keep the existing file, convert it to the symlink, or merge content.

### Step 6: Determine the `llms.txt` URL for the hub

The hub's `llms.txt` links to satellite repositories using raw GitHub URLs. Determine the correct URL for the new repository's root `llms.txt`.

Check the existing entries in the hub's `llms.txt` to see what org and branch pattern is currently in use (e.g. `kabir/{repo}/ai-index/` during development, or `wildfly/{repo}/main/` once upstreamed). Follow the same pattern for the new entry.

Ask the user to confirm the GitHub org/repo name and branch if it's not obvious from the checkout or the existing hub entries.

### Step 7: Review and summarize

- List all files created or modified in the target repository.
- Show the `llms.txt` URL that should be used when running the `onboard-repository` skill on the hub side.
- Flag any areas where you made assumptions the user should verify.
- Remind the user that these files should go on the `ai-index` branch in the target repository, and that they can now run the `onboard-repository` skill in this hub repo to complete the integration.
