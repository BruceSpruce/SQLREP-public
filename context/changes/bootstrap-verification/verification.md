---
bootstrapped_at: 2026-09-18T16:50:00Z
starter_id: tauri
starter_name: Tauri
project_name: sqlrep
language_family: rust
package_manager: npm
cwd_strategy: subdir-then-move
bootstrapper_confidence: verified
phase_3_status: ok
audit_command: cargo audit (+ npm audit --json for the frontend)
---

## Hand-off

```yaml
starter_id: tauri
package_manager: npm
project_name: sqlrep
hints:
  language_family: rust
  team_size: solo
  deployment_target: self-host
  ci_provider: github-actions
  ci_default_flow: auto-deploy-on-merge
  bootstrapper_confidence: verified
  path_taken: standard
  quality_override: false
  self_check_answers: null
  has_auth: false
  has_payments: false
  has_realtime: false
  has_ai: true
  has_background_jobs: true
```

## Why this stack

Narzędzie desktopowe dla jednej osoby/małego zespołu, budowane po godzinach przez 3 tygodnie, z twardym wymogiem działania zarówno na Linuksie, jak i na Windowsie. Tauri to rekomendowany domyślny wybór dla pary `(desktop, rust)` i jedyna pozycja w rejestrze z zweryfikowaną (`verified`) wiarygodnością bootstrappera, która spełnia wymóg przenośności między systemami — łączy backend w Ruście (dobrze nadający się do orkiestracji VPN-a, transferu plików i integracji z SQL Serverem) z lekkim, webowym interfejsem do wygenerowanego raportu. Wybrano ścieżkę standardową — rekomendacja została przyjęta bez zmian. Flagi funkcji AI i zadań w tle są ustawione na podstawie wymagań funkcjonalnych z PRD (lokalne wsparcie AI w interpretacji wyników, uruchomienia bez nadzoru w nocy); logowanie, płatności i funkcje czasu rzeczywistego są jawnie poza zakresem. Domyślny sposób dystrybucji to self-host (ręczna/wewnętrzna dystrybucja instalatorów), a CI działa na GitHub Actions z automatycznym wdrożeniem po scaleniu do main — zgodnie ze standardowym kształtem tego startera.

## Pre-scaffold verification

| Signal      | Value                                          | Severity | Notes                          |
| ----------- | ---------------------------------------------- | -------- | ------------------------------ |
| npm package | create-tauri-app v4.7.4 published 2026-09-04   | fresh    | resolved from cmd_template     |
| GitHub repo | tauri-apps/tauri - not run                     | n/a      | `gh` CLI not installed         |

## Scaffold log

**Resolved invocation**: `npm create tauri-app@latest .bootstrap-scaffold -- --template react-ts --manager npm --yes`
**Strategy**: subdir-then-move
**Exit code**: 0
**Files moved**: 38
**Conflicts (.scaffold siblings)**: none
**.gitignore handling**: moved silently (absent in cwd)
**.bootstrap-scaffold cleanup**: deleted

**History**: the first attempt with `package_manager: cargo` failed (exit 1: `react-ts` template not supported for `cargo`). The hand-off was corrected to `package_manager: npm` at the user's request and the run repeated.

**Warnings**:
- The CLI reported missing system dependencies: Rust (`cargo`) and webkit2gtk & rsvg2. Needed for `npm run tauri dev` / `build`.
- (Resolved after the run: all placeholders renamed to `sqlrep`, identifier `com.example.sqlrep`.) Scaffold names were placeholders: `package.json` name, `src-tauri/Cargo.toml` name (`bootstrap-scaffold`, lib `bootstrap_scaffold_lib`), `productName` and `identifier` (`com.example.bootstrap-scaffold`) in `src-tauri/tauri.conf.json`. 
- `npm install` was run manually after the move (the template does not install).

## Post-scaffold audit

### Frontend - `npm audit --json`
**Summary**: 0 CRITICAL, 0 HIGH, 0 MODERATE, 0 LOW (62 dependencies: 6 prod, 57 dev)

### Rust - `cargo audit` (run later, after installing Rust 1.98.1 and cargo-audit 0.22.2)
**Scanned**: `src-tauri/Cargo.lock` (472 crates; lockfile generated with `cargo generate-lockfile`, since the scaffold ships none), advisory DB with 1251 advisories
**Summary**: 0 vulnerabilities (0 CRITICAL, 0 HIGH, 0 MODERATE, 0 LOW); 7 warnings, all transitive (Tauri dependency tree)

| Crate              | Version | Kind        | ID                |
| ------------------ | ------- | ----------- | ----------------- |
| proc-macro-error   | 1.0.4   | unmaintained | RUSTSEC-2024-0370 |
| unic-char-property | 0.9.0   | unmaintained | RUSTSEC-2025-0081 |
| unic-char-range    | 0.9.0   | unmaintained | RUSTSEC-2025-0075 |
| unic-common        | 0.9.0   | unmaintained | RUSTSEC-2025-0080 |
| unic-ucd-ident     | 0.9.0   | unmaintained | RUSTSEC-2025-0100 |
| unic-ucd-version   | 0.9.0   | unmaintained | RUSTSEC-2025-0098 |
| glib               | 0.18.5  | unsound      | RUSTSEC-2024-0429 |

Notes: no action required now; these come from upstream Tauri/GTK crates and clear as Tauri updates its dependencies. Re-run `cargo audit` after `cargo update`.

## Hints recorded but not acted on

| Hint                    | Value                |
| ----------------------- | -------------------- |
| bootstrapper_confidence | verified             |
| quality_override        | false                |
| path_taken              | standard             |
| self_check_answers      | null                 |
| team_size               | solo                 |
| deployment_target       | self-host            |
| ci_provider             | github-actions       |
| ci_default_flow         | auto-deploy-on-merge |
| has_auth                | false                |
| has_payments            | false                |
| has_realtime            | false                |
| has_ai                  | true                 |
| has_background_jobs     | true                 |

## Next steps

Next: a future skill will set up agent context (CLAUDE.md, AGENTS.md). For now, your project is scaffolded and verified - happy hacking.

Useful manual steps in the meantime:
- Install Tauri Linux system libraries (Rust is installed), then `npm run tauri dev`.
- `git init` / commit as you see fit; no `.scaffold` siblings to review.
