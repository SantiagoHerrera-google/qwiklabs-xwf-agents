# 🤖 Qwiklabs AI Agents & Skills Configuration

This repository contains the local AI agent configurations, skills, and behavior rules (optimized for Gemini/Jetski) specifically focused on and tailored for the main **qwiklab-website** repository. These tools are used by the engineering team to automate and standardize development workflows, code reviews, and architectural alignment.

## 📋 Repository Purpose & Future

To prevent polluting the main application repository with local AI configuration files during this phase, the `.agents` directory is currently ignored in the main project's `.gitignore`. 

This repository is maintained separately and cloned locally as a temporary solution. The engineering team is currently evaluating whether to:
1. Merge these configurations directly into the main `qwiklabs` repository once stabilized.
2. Maintain them as a official **Git Submodule** within the main project.

---

## 🚀 Quick Installation

To activate these AI tools in your local development environment:

### 1. Clean Up Existing Configurations
From the root of the main `qwiklab-website` repository, ensure there is no conflicting `.agents` directory:
```bash
rm -rf .agents
```

### 2. Clone this Repository
Clone this repository directly into the `.agents` directory:
```bash
git clone <REPOSITORY_URL> .agents
```

Once cloned, your local Jetski/Gemini agent will automatically discover and load the rules and skills on the next interaction.

---

## 📂 Repository Structure & Available Tools

Once installed, the repository is organized into the following components:

### 1. `rules/` (Guidelines & Constraints)
Rules define automatic triggers, style guidelines, and codebase context that the AI agent loads into its environment.
*   **[code_review_trigger.md](file:///usr/local/google/home/hesantiago/development/qwiklab-website/.agents/rules/code_review_trigger.md)**: Configures the agent to automatically activate the `@code-review` skill when specific phrases (e.g., "haz un code review") are detected in the chat.
*   **[data_model.md](file:///usr/local/google/home/hesantiago/development/qwiklab-website/.agents/rules/data_model.md)**: Provides the agent with a detailed, structured context of the Dashboard and Learning Activity data models (including Mermaid diagrams), ensuring precise database query analysis.
*   **[architectural_patterns.md](file:///usr/local/google/home/hesantiago/development/qwiklab-website/.agents/rules/architectural_patterns.md)**: Documents best practices for time-series caching (cache-aside), reactive UI (Turbo Streams), and SQL safety (Arel/SafeActiveRecord) to guide architectural reviews.

### 2. `skills/` (Executable Capabilities)
Skills are structured instructions that teach the agent how to perform complex multi-step workflows.
*   **[code-review](file:///usr/local/google/home/hesantiago/development/qwiklab-website/.agents/skills/code-review/SKILL.md)**: A comprehensive, zero-prompt optimized skill that runs project linters (RuboCop, Sorbet, i18n-tasks, Haml-Lint) directly in the terminal, attempts smart autofixes, and outputs a structured architectural report.

---

## 🛠️ Contributing

When adding new automation tools:
*   Place general guidelines or data model context in `rules/`.
*   Place actionable workflows (e.g., deployment guides, database migration checks) in `skills/`.
*   Always ensure paths to files use absolute URIs (`file:///...`) or workspace-relative paths to help the agent resolve them correctly.
