![Konveyor](logos/konveyor.png)

# Konveyor AI (Kai)

[Konveyor](https://www.konveyor.io) is a CNCF project focused on application modernization and migration. **Kai** (Konveyor AI) is its AI-assisted component that uses LLMs to automatically analyze legacy code and generate fixes to migrate applications to cloud-native targets.

## Core concept

Kai combines **static analysis** (finding what needs to change) with **LLM-generated code fixes** (making the changes). Instead of just telling you "this Jakarta EE API is deprecated", it generates the updated code for you.

```
Legacy App (e.g. Java EE, old Spring)
        │
        ▼
  Analyzer LSP  ──→  finds violations against migration rules
        │
        ▼
      Kai  ──→  sends violations + source context to LLM
        │
        ▼
  LLM (GPT-4, Granite, etc.)  ──→  generates code fixes
        │
        ▼
  Modernized App (e.g. Quarkus, Spring Boot 3)
```

## Architecture

- **Analyzer LSP** — a language server that runs static analysis rulesets against the source code. Produces a list of incidents (violations) with file + line context.
- **Rulesets** — YAML-based migration rules (e.g. "replace `javax.persistence` with `jakarta.persistence`"). Maintained by the Konveyor community for common migration paths.
- **Kai server** — orchestrates the fix generation loop. Sends incidents and surrounding source code to the configured LLM, receives patches, applies them.
- **LLM backend** — pluggable. Supports OpenAI (GPT-4), IBM Granite (via watsonx), and any OpenAI-compatible API. Can run locally with Ollama.
- **Konveyor Hub** — the broader Konveyor platform UI where Kai is integrated, providing project management, analysis runs, and fix review.

## Supported migration paths

| From | To |
|---|---|
| Java EE / Jakarta EE | Quarkus |
| Spring Boot 2 | Spring Boot 3 |
| Java EE | JBoss / WildFly |
| Proprietary frameworks | Open-source equivalents |
| On-prem app patterns | Cloud-native (12-factor) patterns |

## How a migration run works

1. **Ingest** — point Kai at a source repository
2. **Analyze** — Analyzer LSP scans the code against a chosen ruleset, producing incidents
3. **Fix loop** — for each incident batch, Kai sends:
   - The violation description
   - The relevant source file(s) with context
   - The migration target hint
   ...to the LLM and receives a code patch
4. **Apply** — patches are applied back to the repo
5. **Review** — developer reviews the generated changes in a PR or via the Konveyor Hub UI
6. **Iterate** — re-run analysis on the patched code to catch remaining issues

## Example ruleset (simplified)

```yaml
- id: javax-to-jakarta
  description: Replace javax.* imports with jakarta.*
  when:
    java.referenced:
      pattern: javax.persistence.*
  message: "Use jakarta.persistence instead of javax.persistence"
  effort: 1
  labels:
    - konveyor.io/target=quarkus
```

## Running Kai locally

```sh
# Install via pip
pip install kai

# Run the Kai server
kai-service --kai-config kai-config.toml

# Point it at your source and run analysis
kai analyze --input ./my-app --ruleset quarkus
```

## Key features

| Feature | Description |
|---|---|
| **AI-assisted fixes** | LLM generates actual code changes, not just hints |
| **Pluggable LLMs** | OpenAI, IBM Granite, Ollama (local), any OpenAI-compatible API |
| **Ruleset-driven** | Community-maintained migration rulesets for common paths |
| **Batch processing** | Handles large codebases by processing incidents in batches |
| **IDE integration** | VS Code extension for inline fix suggestions |
| **Konveyor Hub UI** | Full-featured web UI for managing migration projects at scale |

## Key points

- Kai does not fully automate migrations — it **accelerates** them. Human review of generated patches is still essential.
- Works best on **Java** codebases today; support for other languages is in progress.
- The quality of fixes depends heavily on the LLM used — GPT-4 and IBM Granite (fine-tuned on migration data) produce the best results.
- IBM Granite models used by Kai are **fine-tuned specifically on migration tasks**, making them more reliable than a general-purpose LLM for this use case.
- Fits naturally into a GitOps workflow — Kai generates a PR with the proposed changes, team reviews and merges.
- Part of a broader CNCF modernization story alongside tools like the Kubernetes Operator for managing migrated workloads.
