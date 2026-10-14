# V.E.N.A.
### Verified Exploration and Neural Adaptation

**V.E.N.A. is a local-first, permission-controlled web research and incremental-learning system.** Its purpose is to let a learning engine explore approved web sources while an independent security gate decides what the system may access, what it may see, and what it may retain for future learning.

> **Name note:** “Verified Exploration and Neural Adaptation” is the expansion chosen for this project. It is a project name, not a claim that V.E.N.A. already performs human-like reasoning or verified learning.

## Why V.E.N.A. exists

A conventional language model usually answers from information encoded in its weights and any context or retrieval sources supplied to it. Its knowledge does not automatically update just because it has opened a webpage. V.E.N.A. is an attempt to build the surrounding system that can research new information, preserve source provenance, and update a small learning component under explicit permissions.

The central idea is **intelligence may request access; the security gate decides access**.

V.E.N.A. is intended to:
- Research public, explicitly permitted web pages.
- Enforce domain and browser-action restrictions outside the learning model.
- Separate information the model is allowed to **see** from information it is allowed to **learn from**.
- Queue extracted content for human review before it enters approved memory or updates the learner.
- Keep source URLs and local records so learned material can be traced back to its origin.
- Improve incrementally without using Transformer architectures in the initial design.

## How it differs from a normal LLM

| Typical LLM by itself | V.E.N.A. system |
| --- | --- |
| Answers from its trained weights and current context | Adds a controlled research and memory workflow around a learning component |
| Does not automatically browse unless connected to tools | Uses a browser bridge to request permitted research actions |
| A prompt may ask it to follow rules, but a prompt alone is not a security boundary | A separate gate is designed to enforce permissions independently of the model |
| Reading a page does not automatically change model weights | Viewing, storing, approving and learning are separate stages |
| Knowledge may be difficult to trace to a particular source | The planned knowledge store preserves source and review metadata |
| Often uses Transformer architecture | The initial V.E.N.A. learner is deliberately Transformer-free |

V.E.N.A. is **not itself a new general-purpose LLM yet**. The current starter is a small Python application with a stable character tokenizer, an incremental character n-gram learner, a SQLite store, and a basic policy layer. It is a foundation for the larger system, not proof of autonomous intelligence.

## Core design: the security gate

The V.E.N.A. Gate is the policy-enforcement layer between a learning controller and the browser/data pipeline. The model can propose a topic or action, but it cannot grant itself permissions.

### Two independent permissions

1. **May see:** whether a page or extracted content may be exposed to the learning controller.
2. **May learn:** whether content may be retained in approved knowledge and used to update the learner or a future training dataset.

These permissions must remain separate. A page may be readable for research while still being ineligible for permanent storage or learning.

### Planned controls

- **Domain allowlist and blocklist:** allow specific trusted domains and deny others by default.
- **Action policy:** begin with navigation and reading; deny sensitive actions such as form submission, uploads, downloads, purchases, account changes and arbitrary script execution unless a future design explicitly authorizes a narrowly scoped action.
- **Content boundary:** treat page text as untrusted data, never as instructions that override the controller's policy.
- **Learning approval:** extracted content starts in a pending-review queue. The learner must not update from an item until the human approves it.
- **Resource limits:** cap page count, session duration, response size, bandwidth and storage as implementation matures.
- **Emergency stop:** provide a reliable way to stop the research session.
- **Provenance and auditability:** record source URLs, timestamps, review state and relevant policy decisions.
- **Network restrictions:** future hardening should validate redirects and network destinations and add egress restrictions, not rely only on browser-level domain checks.

**Security limitation:** the current policy layer is an application-level gate, not a hardened operating-system sandbox. It does not by itself guarantee protection against every network escape, browser exploit, DNS rebinding or malicious page. Stronger isolation may require a dedicated unprivileged macOS account or isolated runtime plus network egress controls.

## Architecture

```text
                 Research request / topic
                           |
                           v
                 V.E.N.A. Controller
                           |
                           v
                  +------------------+
                  |    V.E.N.A. Gate |
                  | domain + actions  |
                  | visibility policy |
                  | resource limits   |
                  +------------------+
                           |
                   permitted actions
                           v
                 Chromium + agent-browser
                           |
                     page extraction
                           v
                 Untrusted content boundary
                           |
                           v
                    Pending review queue
                           |
                 human approves or rejects
                           |
                 +---------+----------+
                 |                    |
                 v                    v
          Approved knowledge    Incremental learner
          SQLite + provenance   character n-grams
                 |                    |
                 +---------+----------+
                           |
                 Search / generation / evaluation
```

This diagram describes the intended system. Not every box is implemented in the current starter yet.

## Technology choices

| Area | Initial choice | Reason |
| --- | --- | --- |
| Main language | Python | Small, inspectable local components |
| Browser engine | Chromium | Mature browser foundation |
| Browser automation | agent-browser | Reuse an existing open-source browser-control interface instead of writing a browser engine |
| Policy gate | Custom V.E.N.A. code | Project-specific access, visibility and learning rules |
| Local storage | SQLite | Simple local persistence for policy and knowledge records |
| Tokenizer | Unicode code-point IDs (ord(character)) | Stable IDs across English, Telugu and other scripts without vocabulary migration |
| Learner | Incremental character n-gram baseline | Lightweight, non-Transformer statistical learning |
| Deployment target | Local macOS first | Fits the initial MacBook Air M2, 8 GB RAM development constraint |

The browser engine and automation layer are reused components; the project-specific value is the policy gate and the controlled learning workflow. The gate must validate every browser action and every write into approved learning data.

## Tokenizer and learning approach

### Stable multilingual tokenizer

The starter maps each Unicode code point to a stable integer ID using its code point value (ord(character)). This naturally handles text in English, Telugu and many other writing systems without a growing vocabulary or token-ID remapping.

This approach is intentionally simple:
- Existing IDs do not change as new text arrives.
- It avoids vocabulary migration that could invalidate learned statistics.
- It is not compression-efficient and can produce longer sequences than subword tokenizers.
- A tokenizer converts text into IDs; **the tokenizer itself does not learn facts or make the system intelligent**.

### Incremental character n-gram learner

The starter includes a small character n-gram learner that can update statistics incrementally. It is a lightweight baseline for experimenting with continual updates. It is not a Transformer and should not be described as equivalent to a modern general-purpose LLM.

The project will measure learning with repeatable tests and held-out examples rather than assuming that adding more text always improves quality.

### Knowledge storage is not the same as weight training

V.E.N.A. keeps these concepts separate:

- **Reading:** extracting information from a permitted page.
- **Memory:** storing approved content with source metadata.
- **Learning:** updating the incremental learner from approved text.
- **Training export:** preparing reviewed, licensed and deduplicated material for a future experiment.

Saving a fact in SQLite does not automatically change model weights. Likewise, updating n-gram statistics is not the same as pretraining a large language model.

## Local-first constraints

The initial target is a **MacBook Air M2 with 8 GB RAM**. The first phase is intended to run locally without a cloud GPU. Work should be incremental and memory-conscious; large pages and datasets should not all be loaded into memory at once.

The initial scope deliberately excludes:
- Transformer models or Transformer-based learning.
- A mandatory cloud GPU.
- Unrestricted crawling of the entire internet.
- Unreviewed automatic learning from arbitrary webpages.
- Treating webpage instructions as trusted controller instructions.
- Mixing this project with LISA.

A laptop can run a lightweight local prototype, but it is not automatically a reliable 24/7 service. Continuous operation and larger-scale collection would require separate resource and deployment decisions.

## Current implementation status

The current local starter lives at ~/VENA-workspace/VENA_Starter on the developer's Mac.

**Confirmed from the current terminal output:**
- The Python CLI starts and shows commands for initialization, status, domain policy, fetching, review, approval, search, tokenization and generation.
- The three starter tests pass: domain normalization, incremental learner, and stable multilingual tokenizer.
- The detected Python version in that terminal was Python 3.9.6.
- Node.js, npm, Homebrew and agent-browser were not detected on the PATH in that terminal session.

**Not yet confirmed as working:**
- Installing and launching agent-browser.
- End-to-end browser extraction through the intended browser automation.
- The full controlled research loop.
- A dashboard/control center.
- Network-level isolation or a hardened OS sandbox.
- Autonomous research planning or robust factual verification.
- A production-ready LLM or any large-scale model training.

The current test results validate only the tested starter components; they do not certify the whole security architecture.

## Getting started with the starter

The project is under active development. From the project directory on the development Mac:

```bash
cd ~/VENA-workspace/VENA_Starter
source .venv/bin/activate
python -m vena --help
python -m unittest discover -s tests -v
```

The current CLI advertises these command groups:

```text
init
status
allow-domain
block-domain
fetch
browser-read
pending
show
approve
reject
search
generate
tokenize
```

Use `python -m vena <command> --help` to inspect a command's options before using it. Browser commands depend on the browser integration being installed and configured; that integration has not yet been confirmed in the current environment.

The starter's README describes its local data files as:
- ./config.json for configuration and policy.
- ./data/vena.sqlite3 for local storage.

Back up these files if you need to preserve the local configuration and corpus. Do not add secrets, credentials or private browser data to Git.

## Roadmap

### Phase 1 — Reliable local foundation
- Keep the current tests passing.
- Resolve the Python/runtime and browser-tool installation requirements.
- Initialize the local database and inspect the default policy.
- Confirm the browser starts and can read a single explicitly allowlisted public page.

### Phase 2 — Enforce the gate
- Test allowed and blocked domains.
- Validate navigation targets and redirects.
- Enforce read-only browser actions by default.
- Apply output-size and session resource limits.
- Ensure webpage text cannot change the controller's permissions.

### Phase 3 — Review before learning
- Extract content into a pending queue.
- Show source, timestamp and extracted text for review.
- Support explicit approval and rejection.
- Verify rejected or pending items cannot update approved memory or learner statistics.

### Phase 4 — Knowledge and incremental learning
- Preserve provenance and content hashes.
- Add deduplication and quality checks.
- Update the n-gram learner only from approved material.
- Evaluate on held-out examples and track regressions.
- Keep search/retrieval separate from model learning.

### Phase 5 — Control center and stronger isolation
- Build local start/stop and policy controls.
- Add an audit trail and emergency stop.
- Introduce stricter network egress and process isolation.
- Add bounded, resumable research sessions and explicit storage quotas.

### Phase 6 — Evaluate future learning architectures
- Measure the current n-gram baseline honestly.
- Consider other non-Transformer models only if testing shows a clear need.
- Export only content with appropriate permissions and licensing for any later training experiment.

## Responsible collection

V.E.N.A. is intended for permitted, responsible research. Public accessibility does not automatically grant permission to reuse content for model training.

- Respect robots.txt, website terms, rate limits and access controls.
- Follow copyright and content licensing requirements.
- Do not bypass authentication, paywalls or technical restrictions.
- Prefer sources that explicitly permit the intended collection and reuse.
- Store provenance and licensing notes alongside collected content.
- Treat all web content as untrusted input.

## Contributing and project principles

The implementation should remain small, inspectable and testable. Changes to policy enforcement should include tests for both allowed and denied behavior. Do not claim a security guarantee that has not been tested.

Project principles:
1. **Policy before autonomy.**
2. **Human approval before learning.**
3. **May see is not the same as may learn.**
4. **Source provenance matters.**
5. **Incremental updates must be evaluated.**
6. **No Transformers in the initial design.**
7. **V.E.N.A. remains separate from LISA.**

---

**Status:** Early prototype / active development. The three core tests pass, while browser integration and the end-to-end controlled learning loop remain unfinished.
