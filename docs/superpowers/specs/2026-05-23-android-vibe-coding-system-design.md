# Android Vibe Coding System — Design Spec
Date: 2026-05-23

## Purpose
A layered skill system for AI agents (vibe coders) doing Android development. Built by a senior Android engineer to encode expert knowledge — concepts, implementation patterns, and guardrails — into reusable Claude Code skills. Any Android project that includes CLAUDE.md from this system gets the full rule set automatically; section skills and agent roles are invoked on demand.

---

## Layer 1: CLAUDE.md (Global Rules)

Auto-loaded by Claude Code for any Android project that includes it. Contains non-negotiable rules that apply to every task regardless of which skill is active.

**Rule categories:**
- Language: Always Kotlin. Never Java for new code.
- Architecture default: MVVM + Repository pattern unless explicitly overridden.
- Threading: Coroutines only. Never Thread, AsyncTask, or RxJava.
- DI: Hilt is preferred. Koin accepted if already in project. Never manual DI for non-trivial graphs.
- UI: Jetpack Compose preferred. XML Views accepted for existing projects — do not mix without reason.
- Storage: Room over raw SQLite. DataStore over SharedPreferences for new code.
- Networking: Retrofit + OkHttp. Never HttpURLConnection or Volley.
- Lifecycle: Never hold Context in non-lifecycle-aware objects. Never ignore lifecycle.
- Main thread: Never block. IO always on Dispatchers.IO, CPU on Dispatchers.Default.
- Nullability: Explicit — no `!!` except where non-null is guaranteed by contract.
- Error handling: Never swallow exceptions silently. Use sealed classes for result types.

---

## Layer 2: Section Skills (9 skills)

One skill per roadmap section. Each skill has three parts:
1. **Concept** — what it is, why it exists, when to use it
2. **Implementation patterns** — idiomatic Kotlin/Android code snippets
3. **Guardrails** — explicit DO/DON'T rules with reasoning

| Skill file | Covers |
|---|---|
| `android-app-components.md` | Activity, Services, Content Provider, Broadcast Receiver, Intents (implicit/explicit), Intent Filters, Lifecycle, State Changes |
| `android-ui-layouts.md` | ConstraintLayout (XML + Compose), TextView, ImageView, Fragments, Jetpack Compose basics, App Shortcuts, Navigation Components |
| `android-architecture.md` | MVVM, MVP, MVC, MVI, Repository Pattern, Builder/Factory/Observer patterns, Flow, LiveData, DI (Hilt, Koin, Dagger, Kodein) |
| `android-storage.md` | SharedPreferences, DataStore, Room Database, File System |
| `android-networking.md` | Retrofit, OkHttp, Apollo Android (GraphQL) |
| `android-async.md` | Coroutines, Threads, WorkManager |
| `android-firebase.md` | Auth, Crashlytics, Remote Config, FCM, Firestore, AdMob, Play Services, Maps |
| `android-code-quality.md` | Linting, Ktlint, Detekt, Timber, LeakCanary, Chucker, Jetpack Benchmark |
| `android-testing.md` | JUnit (unit tests), Espresso (UI tests), test strategy |

---

## Layer 3: Agent Role Skills (4 skills)

Specialized agents invoked to take on a specific role. Each role skill defines:
- The agent's responsibility and decision scope
- Which section skills to pull in for the current task
- How to interact with other roles

| Skill file | Role | Activates when |
|---|---|---|
| `android-architect.md` | Designs app structure, picks patterns, defines module boundaries | Starting a feature, planning app architecture |
| `android-implementer.md` | Writes feature code following established patterns | Implementing a specific screen, component, or API call |
| `android-debugger.md` | Diagnoses crashes, memory leaks, ANRs, performance issues | Investigating a bug, crash, or perf regression |
| `android-reviewer.md` | Reviews code against Android best practices and project rules | Before PR merge, validating a vibe coder's output |

---

## Skill File Template

Every section skill follows this structure:

```markdown
---
name: android-<section>
description: <one-line trigger description>
---

# <Section Name>

## Concept
[What it is, why it exists, when to use each sub-topic]

## Implementation Patterns
[Idiomatic Kotlin/Android code snippets per subtopic]

## Guardrails
### DO
- ...

### DON'T
- ...

## References
[Links to official docs from the roadmap]
```

---

## File Structure

```
AndroidVibeX/
├── CLAUDE.md                              ← copy into any Android project
├── skills/
│   ├── sections/
│   │   ├── android-app-components.md
│   │   ├── android-ui-layouts.md
│   │   ├── android-architecture.md
│   │   ├── android-storage.md
│   │   ├── android-networking.md
│   │   ├── android-async.md
│   │   ├── android-firebase.md
│   │   ├── android-code-quality.md
│   │   └── android-testing.md
│   └── agents/
│       ├── android-architect.md
│       ├── android-implementer.md
│       ├── android-debugger.md
│       └── android-reviewer.md
└── docs/
    └── superpowers/specs/
        └── 2026-05-23-android-vibe-coding-system-design.md
```

---

## Success Criteria
- A vibe coder can drop CLAUDE.md into any Android project and immediately get enforced Android best practices
- Invoking any section skill gives concept + code pattern + guardrails for that domain
- Invoking an agent skill puts Claude into a clear, scoped role with defined responsibilities
- No skill duplicates what android-cli already covers (tooling)
- Each skill is self-contained and can be used independently
