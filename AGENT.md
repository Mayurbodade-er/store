# AGENT.MD: Operating Protocol & Guardrails for AI Coding Agents
**Project:** RetailOS / Store ERP  
**Architecture:** Strict Vertical Slice Architecture (VSA) + Scaled Trunk-Based Development (TBD)  
**Target Stack:** Pure Rust (Dioxus 0.7 Web/Desktop + Tauri v2.12 + SQLCipher SQLite)  
**Document Version:** 1.0  
**Effective Date:** October 2026  

---

## 1. Prime Directive: Mandatory Human Verification Gate

> [!CAUTION]
> ### THE GOLDEN RULE OF MERGING
> **NO AI AGENT IS PERMITTED TO DIRECTLY MERGE OR PUSH CODE TO `main`.**  
> Under Scaled Trunk-Based Development, `main` is the perpetual, releasable single source of truth. Every proposed change MUST undergo explicit human verification and review before merging.

### 1.1 Agent Boundaries
1. **Never Execute Direct Push to `main`**: Autonomous agents must only commit and push to short-lived feature or fix branches (`feat/*`, `fix/*`, `nfr/*`, `docs/*`).
2. **Never Auto-Merge Pull Requests**: Even if all automated CI checks and tests pass, merging into `main` requires an explicit human review and click of the merge button.
3. **Handover Protocol**: When work on a slice is finished, the agent must output a structured **Human Verification Handover Report** containing:
   - Branch name and commit hash.
   - Traceability mapping (`FR-TRACE` & `NFR-TRACE`).
   - Summary of changes across `internal/`, `external/`, and `test/`.
   - Verification proofs: output of `cargo fmt`, `cargo clippy`, and `cargo nextest`.
   - Token check: confirmation that `SLICE.md` is updated and `< 250 tokens`.

---

## 2. Inviolable Architectural Invariants

Every agent operating on this codebase must adhere strictly to these non-negotiable architectural rules:

### 2.1 Pure Rust Stack Lockdown
- **UI Framework**: **Dioxus 0.7** exclusively.
- **Cross-Platform Shell**: **Tauri v2.12** exclusively.
- **Database**: `rusqlite 0.40.2` with SQLCipher (native desktop) and IndexedDB/OPFS SQLite (web WASM).
- **Strictly Forbidden**: NEVER introduce JavaScript, TypeScript, React, Vue, Flutter, Python backends, or Node.js runtime servers.

### 2.2 Strict Vertical Slice Isolation (VSA)
- Every feature lives inside `src/features/<slice_name>/` following the 10-item canonical template:
  - `SLICE.md` (Manifest < 250 tokens)
  - `mod.rs` (Public gateway re-exporting only external items)
  - `internal/` (`logic.rs`, `models.rs`, `repo.rs`) — `pub(crate)` only!
  - `external/` (`style.css`, `components/`, `hooks/`, `views.rs`) — Public consumption interface.
  - `test/` (`test/fr/` functional tests + `test/nfr/` non-functional tests).
- **Cross-Slice Isolation**: A slice must NEVER import private internals (`crate::features::other_slice::internal::*`) of another slice. Interaction occurs strictly through `packages/core-*` traits, public events, or foreign key IDs.

### 2.3 UI/UX SoftMorph Design System
- **Neomorphism Palette**: Oatmeal Clay (`#EFE9E0`) light surface, Smoked Basalt (`#1A1C20`) dark surface, Soft Terracotta (`#E06D3B`) accent, Apricot Cream (`#F6E2D5`) tactile bumps.
- **Zero Blue**: NEVER use blue buttons, blue badges, or blue accents (violates brand guidelines).
- **No Flat Borders**: Use feathered dual Gaussian shadows (convex unpressed / concave pressed), never harsh `1px solid black` borders.

### 2.4 Currency & Fixed-Point Mathematics
- **Zero Floats (`f32`/`f64`) in Money**: NEVER use IEEE-754 floating-point numbers for prices, taxes, weights, discounts, or ledger balances.
- **Fixed-Point Decimal**: Always use `rust_decimal::Decimal` (128-bit fixed-point) with explicit rounding modes.

---

## 3. Token-Efficiency & Agent Context Optimization

To prevent context bloat and ensure fast, deterministic reasoning, agents must observe these token-saving directives:

```
                  AGENT TOKEN EFFICIENCY WORKFLOW
+-----------------------------------------------------------------+
| 1. Read SLICE.md first (< 250 tokens)                           |
|    - Understands models, tables, RBAC rules without opening 10 files |
+-----------------------------------------------------------------+
                                |
                                v
+-----------------------------------------------------------------+
| 2. Style Changes? Edit external/style.css directly              |
|    - Avoids reading or touching Rust RSX component logic       |
+-----------------------------------------------------------------+
                                |
                                v
+-----------------------------------------------------------------+
| 3. Domain Logic Change? Edit internal/logic.rs & test/fr/       |
|    - Keeps changes surgical and contained inside slice          |
+-----------------------------------------------------------------+
```

1. **Read `SLICE.md` First**: Before opening multiple files within a slice, read `src/features/<slice_name>/SLICE.md`. It provides the complete mental model in under 250 tokens.
2. **Style Isolation**: If asked to modify visual appearance, colors, padding, or shadows, modify `external/style.css`. Do not re-write or inflate Rust component files (`.rs`) unless structure/interactivity changes.
3. **Targeted Context Loading**: Never load the entire repository into memory. Only open files directly referenced in the slice task.

---

## 4. Git & Trunk-Based Development Playbook for Agents

Agents must follow the branch lifecycle defined in `docs/git-strategy-trunk-based-development.md`:

### 4.1 Branch Lifecycle
```bash
# 1. Always start from latest main
git checkout main
git pull --rebase origin main

# 2. Spawn short-lived branch (< 24 hours, atomic diff)
git checkout -b feat/pos-cart-parking

# 3. Work surgically inside the slice
# ... modify files in src/features/pos_cart_parking/ ...

# 4. Run local pre-flight checks (MUST PASS 100%)
cargo fmt --all -- --check
cargo clippy --all-targets --all-features -- -D warnings
cargo nextest run --package feature-pos-cart-parking

# 5. Rebase on trunk before push
git fetch origin main
git rebase origin/main

# 6. Commit with Conventional Commits
git commit -am "feat(pos): implement 3-slot cart parking and tactile hold badge

FR-TRACE: FR-POS-06
NFR-TRACE: NFR-PERF-01"

# 7. Push branch and STOP for human verification
git push -u origin feat/pos-cart-parking
```

### 4.2 Commit Message Rules
- Format: `<type>(<scope>): <summary>`
- Allowed types: `feat`, `fix`, `perf`, `security`, `test`, `refactor`, `style`, `chore`, `docs`.
- Scope must match slice name (`pos`, `inventory`, `rbac`, `khata`, `core-db`).
- Must include traceability footers: `FR-TRACE: <ID>` and `NFR-TRACE: <ID>`.

---

## 5. In-Slice Quality Gate & Definition of Done (DoD)

Before requesting human verification, an agent must verify that every item on this checklist is complete:

| Check | Command / Verification | Threshold |
| :--- | :--- | :--- |
| **Code Formatting** | `cargo fmt --all -- --check` | Zero diffs |
| **Compiler Warnings** | `cargo clippy --all-targets --all-features -- -D warnings` | Zero warnings |
| **In-Slice Functional Tests** | `cargo nextest run --package feature-<slice>` | 100% Passing (0 failures) |
| **In-Slice Performance** | `cargo bench --bench perf_<slice>` | POS `< 50ms`, Barcode `< 5ms`, PIN `< 45ms` |
| **Security & Fuzzing** | `cargo test --test sec_*` | 0 leaks, zero plaintext secrets |
| **Slice Manifest** | `cat src/features/<slice>/SLICE.md` | Exists, accurate, `< 250 tokens` |
| **Dependencies** | `cargo deny check` & `cargo audit` | 0 high/critical vulnerabilities |

---

## 6. Prohibited Practices & Error Recovery

### 6.1 Strictly Prohibited Actions
- ❌ **NEVER** use `#[allow(warnings)]` or `#[allow(dead_code)]` to bypass Clippy errors. Fix the underlying issue.
- ❌ **NEVER** use `unwrap()` or `expect()` in production `internal/` or `external/` code. Always use `Result<T, E>` with explicit error variants.
- ❌ **NEVER** print secrets, PINs, or raw database credentials via `println!`, `dbg!`, or `log::info!`.
- ❌ **NEVER** execute destructive git commands (`git reset --hard origin/main`, `git push --force`) without explicit human instruction.

### 6.2 Error Recovery Protocol
1. **Compilation Failure**: Read the compiler diagnostic carefully, check borrow checker lifecycles, and fix the root cause. If stuck after 2 attempts, articulate the specific lifetime or type mismatch to the human.
2. **Test Failure**: Inspect whether failure is due to stale test expectations or broken business logic. Never modify a test to make it pass unless the business rule itself was intentionally updated.
3. **Merge Conflict during Rebase**: If conflicts occur during `git rebase origin/main`, examine the diff in the slice. Resolve conflicts only within the slice's own files. If conflicts touch shared core crates, request human guidance.

---

## 7. Standard Human Handover Report Template

When an agent completes a task, it must present its output to the human in this exact format:

```markdown
### 📋 Human Verification Handover Report

- **Branch:** `feat/<slice-name>` (Commit: `<hash>`)
- **Vertical Slice:** `src/features/<slice_name>/`
- **Traceability:**
  - Functional: `FR-TRACE: <BRD-ID>`
  - Non-Functional: `NFR-TRACE: <NFR-ID>`

#### Summary of Changes:
- **`internal/`**: [Brief description of logic, model, or repo changes]
- **`external/`**: [Brief description of UI, views, components, style.css]
- **`test/`**: [List of added unit, integ, rbac, perf, and sec tests]
- **`SLICE.md`**: [Updated, token count: ~XXX tokens]

#### Pre-Flight Quality Gate Results:
- `cargo fmt`: ✅ Clean
- `cargo clippy`: ✅ 0 warnings (-D warnings enforced)
- `cargo nextest`: ✅ XX passed, 0 failed
- `criterion bench`: ✅ Latency met (< XXms budget)

**Status:** Ready for human code review and merge into `main`.
```

---
*Every AI agent operating on RetailOS must honor this document as its highest procedural directive.*
