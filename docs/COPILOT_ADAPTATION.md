# GitHub Copilot CLI Adaptation Guide (ARIS Workflows)

> Use ARIS research workflows in GitHub Copilot CLI with native `SKILL.md` skills, repo instructions, MCP setup, and file-based recovery.

GitHub Copilot CLI is a natural fit for ARIS because it already supports the same building blocks ARIS depends on: Markdown `SKILL.md` skills, repository instructions, MCP servers, `@file` references, and resumable CLI sessions.

## 1. Key Differences: Claude Code vs GitHub Copilot CLI

| Concept | Claude Code | GitHub Copilot CLI |
|---------|-------------|--------------------|
| Skill invocation | `/skill-name "args"` | Write a prompt such as `Use the /skill-name skill for ...` after the skill is loaded; `@skills/.../SKILL.md` is the fallback |
| Skill storage | `~/.claude/skills/skill-name/SKILL.md` | `~/.copilot/skills/skill-name/SKILL.md`, `.github/skills/skill-name/SKILL.md`, or any added skill location via `/skills add` |
| Project instructions | `CLAUDE.md` in project root | `.github/copilot-instructions.md`, `.github/instructions/**/*.instructions.md`, `AGENTS.md` |
| MCP servers | `claude mcp add ...` | `/mcp add` (stored in `~/.copilot/mcp-config.json` by default) |
| File references | Auto-read from project | `@path/to/file` |
| Session recovery | Persistent CLI session with auto-compact | `/resume` plus ARIS artifact/state files |
| Permissions | Tool approvals inside Claude Code | Trusted-folder prompt plus per-tool approvals in Copilot CLI |

## 2. Setup

### 2.1 Install GitHub Copilot CLI

Install Copilot CLI using the method you prefer. One option:

```bash
npm install -g @github/copilot
copilot
```

If prompted, run `/login` and authenticate with your GitHub account.

### 2.2 Clone ARIS

```bash
git clone https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep.git
cd Auto-claude-code-research-in-sleep
```

When Copilot asks whether you trust the folder, choose the option that matches your environment. ARIS workflows are tool-heavy, so Copilot will need permission to read files, edit files, and run commands in the repo.

### 2.3 Register ARIS skills (recommended)

ARIS keeps its canonical skills in the top-level `skills/` directory. In Copilot CLI, the cleanest path is to register that directory instead of copying or forking the skills.

1. Start Copilot CLI in the ARIS repo root:
   ```bash
   copilot
   ```
2. Add the skills directory:
   ```text
   /skills add
   ```
3. When prompted for the location, enter the absolute path to `<repo>/skills`.
4. Verify the skills are visible:
   ```text
   /skills list
   ```
5. If you edit any `SKILL.md` file during the session, refresh the registry:
   ```text
   /skills reload
   ```

If you prefer a static install, you can also copy selected skill directories into `~/.copilot/skills/` or `.github/skills/`. Keep the top-level `skills/` tree as the source of truth.

> Note: ARIS skills include extra frontmatter such as `argument-hint` and `allowed-tools` in the Claude-oriented skill set. Copilot CLI only requires `name` and `description`, so test a few core skills first. If a skill does not load cleanly, attach the `SKILL.md` directly with `@skills/.../SKILL.md`.

### 2.4 Set up reviewer MCP

ARIS review-heavy workflows expect an external reviewer. In Copilot CLI, configure MCP servers with `/mcp add`.

#### Option A: Codex reviewer

Install and configure Codex first:

```bash
npm install -g @openai/codex
codex setup
```

Then inside Copilot CLI:

```text
/mcp add
```

Use these values:

- Server name: `codex`
- Command: `codex`
- Args: `mcp-server`

Use the server name `codex` so ARIS skill references to `mcp__codex__codex` stay aligned.

#### Option B: `llm-chat` reviewer (no OpenAI API required)

Create a virtual environment and install the dependency:

```bash
cd /path/to/Auto-claude-code-research-in-sleep
python3 -m venv .venv
.venv/bin/pip install -r mcp-servers/llm-chat/requirements.txt
```

Then add the MCP server from inside Copilot CLI:

```text
/mcp add
```

Use these values:

- Server name: `llm-chat`
- Command: absolute path to `.venv/bin/python3`
- Args: absolute path to `mcp-servers/llm-chat/server.py`
- Env:
  - `LLM_BASE_URL`
  - `LLM_API_KEY`
  - `LLM_MODEL`

Keep the server name `llm-chat` so ARIS skill references to `mcp__llm-chat__chat` remain accurate.

See [LLM_API_MIX_MATCH_GUIDE.md](LLM_API_MIX_MATCH_GUIDE.md) for tested provider configurations.

### 2.5 Repository instructions

Copilot CLI automatically reads `.github/copilot-instructions.md`. This repo uses that file for architecture and workflow guidance, so work from the repo root whenever possible.

## 3. How to Invoke Skills

### Approach A: Use the registered skill name (recommended)

Once the ARIS skill directory is registered, invoke a skill by naming it in the prompt:

```text
Use the /auto-review-loop skill for "factorized gap in discrete diffusion LMs".
```

This is the closest equivalent to Claude Code's `/auto-review-loop "..."`.

### Approach B: Attach the `SKILL.md` file directly

If the skill registry is not set up yet, or you want a one-off invocation, attach the skill file explicitly:

```text
@skills/auto-review-loop/SKILL.md

Run the auto review loop for "factorized gap in discrete diffusion LMs".
```

### Approach C: Attach both the skill and the input artifact

ARIS workflows are file-centric. In Copilot CLI, it is often helpful to attach both the workflow skill and the file it should operate on:

```text
@skills/paper-writing/SKILL.md
@NARRATIVE_REPORT.md

Run the full paper-writing workflow from the attached narrative report.
```

## 4. Workflow Mapping

### Workflow 1: Idea Discovery

**Claude Code:**

```text
/idea-discovery "your research direction"
```

**Copilot CLI equivalent:**

```text
Use the /idea-discovery skill for "your research direction".

If needed, read @templates/RESEARCH_BRIEF_TEMPLATE.md first and help me fill it in before running the workflow.
```

### Workflow 1.5: Experiment Bridge

**Claude Code:**

```text
/experiment-bridge
```

**Copilot CLI equivalent:**

```text
Use the /experiment-bridge skill.
Read @refine-logs/EXPERIMENT_PLAN.md and implement the experiments.
Use the configured reviewer MCP before deployment if the skill requests code review.
```

### Workflow 2: Auto Review Loop

**Claude Code:**

```text
/auto-review-loop "your paper topic"
```

**Copilot CLI equivalent:**

```text
Use the /auto-review-loop skill for "your paper topic".
Read the current narrative docs, experiment outputs, and state files if present.
Use the configured reviewer MCP when the skill asks for an external review.
```

### Workflow 3: Paper Writing

**Claude Code:**

```text
/paper-writing "NARRATIVE_REPORT.md"
```

**Copilot CLI equivalent:**

```text
Use the /paper-writing skill.
Input: @NARRATIVE_REPORT.md
```

### Full Pipeline

**Claude Code:**

```text
/research-pipeline "your research direction"
```

**Copilot CLI equivalent:**

```text
Use the /research-pipeline skill for "your research direction".
If the run becomes too large for one session, break it into stages and carry the output files forward.
```

## 5. State Files and Recovery

ARIS persists progress to files. This works well with Copilot CLI's `/resume` flow.

| File | Purpose |
|------|---------|
| `IDEA_REPORT.md` | Ranked ideas with pilot signals and novelty notes |
| `AUTO_REVIEW.md` | Review history and implemented fixes |
| `REVIEW_STATE.json` | Auto-review loop checkpoint state |
| `PAPER_PLAN.md` | Outline plus claims-evidence matrix |
| `refine-logs/FINAL_PROPOSAL.md` | Refined proposal from Workflow 1 |
| `refine-logs/EXPERIMENT_PLAN.md` | Experiment roadmap |
| `refine-logs/EXPERIMENT_TRACKER.md` | Run-by-run execution status |

To continue work in a new session:

```text
/resume
@skills/auto-review-loop/SKILL.md
@AUTO_REVIEW.md
@REVIEW_STATE.json

Continue the /auto-review-loop skill from the saved state.
```

## 6. Practical Notes

- Prefer the top-level `skills/` tree. `skills/skills-codex/` is for the Codex-native executor path, not the default Copilot path.
- Existing ARIS skills were written for Claude Code, so some instructions still mention Claude-specific tool names or slash chaining. In Copilot CLI, be explicit about subskills, files, and reviewer MCP choices when needed.
- Use `@templates/...` and `@skills/.../SKILL.md` liberally. Copilot CLI is strong at file-grounded workflows.
- Expect trust and tool-approval prompts, especially for long ARIS runs that edit files and execute commands.
- For your first smoke test, start with a smaller workflow such as `research-lit`, `idea-discovery`, or `auto-review-loop` before jumping straight to `/research-pipeline`.
