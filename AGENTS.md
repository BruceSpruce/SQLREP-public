# Repository Guidelines

SQLREP to aplikacja desktopowa (Tauri 2 + React 19 + TypeScript, backend w Rust), która automatyzuje tygodniowy przepływ analizy trace'ów SQL Server: VPN → pobranie trace'ów → import do bazy w schemacie zgodnym z ReadTrace/SQL Nexus → procedury porównujące → raport. Zakres i wymagania: @context/foundation/prd.md, wybór stacku: @context/foundation/tech-stack.md.

## Hard rules

- Dane klienta (trace'y, wyniki) nie mogą trafić do zewnętrznej usługi w chmurze; AI działa wyłącznie lokalnie (FR-006).
- Dane różnych klientów są odizolowane: żadna ścieżka, baza ani cache nie może być współdzielona między klientami.
- Raport nigdy nie jest wysyłany automatycznie; wymaga ręcznego zatwierdzenia użytkownika.
- MVP nie ma logowania ani multi-user; nie dodawaj auth. Wspierany jest jeden układ plików trace'ów u klienta.
- Aplikacja działa wyłącznie na hoście Linux. Pliki `.trc` (SQL Server 2008 i nowsze) importuje własny importer przez SQL Server Express w Dockerze (`sys.fn_trace_gettable`); VM z Windows i ReadTrace to tylko ścieżka zapasowa, budowana gdy importer nie przejdzie testu zgodności. Nie dodawaj natywnego wsparcia Windowsa. Szczegóły: @context/foundation/infrastructure.md.
- Nie wyświetlaj tekstów zapytań ani wartości z danych klienta w odpowiedziach agenta (trafiają do chmury); raportuj tylko wskaźniki zgodności i liczności.
- Nie edytuj `context/archive/`; zmiany w PRD rób przez `/10x-prd`.

## Project Structure

- `src/` — frontend React (`main.tsx`, `App.tsx`); Vite dev server na porcie 1420 (@vite.config.ts).
- `src-tauri/src/lib.rs` — komendy Tauri i `run()`.
- `src-tauri/capabilities/default.json` — uprawnienia okna. Nowy plugin wymaga trzech rzeczy: zależności w `Cargo.toml`, `.plugin(...)` w `run()` i wpisu tutaj.
- `context/` — PRD, tech-stack i zmiany (workflow 10x); nie jest kodem aplikacji.

## Build, Test, and Development Commands

- `npm run tauri dev` — aplikacja z hot reload (uruchamia `npm run dev`).
- `npm run build` — `tsc` + build frontendu; musi przejść bez błędów typów.
- `npm run tauri build` — instalatory (target `all`).
- `cargo check --manifest-path src-tauri/Cargo.toml` — szybka kontrola backendu.
- `docker compose -f deploy/docker-compose.yml up -d|down|logs` — lokalny SQL Server Express (`sqlrep-mssql`, `127.0.0.1:1433`, dane w `$SQLREP_DATA_DIR/mssql`). Hasło `sa` jest w `~/.config/sqlrep/sqlrep.env` (poza repozytorium); nie wypisuj go.

Brak skonfigurowanych testów, lintera i CI; nie zakładaj `npm test`.

## Coding Style

- Reguły TypeScript: @tsconfig.json.
- Nowe komendy backendu: funkcja z `#[tauri::command]` w @src-tauri/src/lib.rs ORAZ wpis w `generate_handler![]` w `run()` (bez tego runtime zwraca „command not found"). `greet` to szablon i może zniknąć.
- Logikę SQL/VPN/plików trzymaj w Rust, a frontend niech tylko woła `invoke` i wyświetla wynik.

## Gotchas

- `npm run build` sprawdza tylko frontend, a `cargo check` tylko Rust; uruchamiaj oba.
- `tsconfig` ma `noUnusedLocals`/`noUnusedParameters`: nieużywany import psuje `npm run build`.
- Port 1420 jest `strictPort`; `tauri dev` nie wystartuje, jeśli jest zajęty.
- `tauri.conf.json` ma zawężone CSP (`connect-src` tylko IPC Tauri). Nie poluzowuj go; połączenia z bazą i z LLM idą przez backend w Ruście, nie z frontendu.
- `src-tauri/gen/schemas` jest generowany i ignorowany przez git; `capabilities/default.json` odwołuje się do niego dopiero po pierwszym buildzie.
- `tauri build` buduje instalatory tylko dla systemu hosta; na Linuksie wymaga bibliotek systemowych webkit2gtk.

## Commit Guidelines

Conventional Commits z prefiksem (w historii występują `chore:` i `docs:`; `feat:`/`fix:` do potwierdzenia). Domyślna gałąź: `main`.
