# SQLREP

Aplikacja desktopowa (Tauri 2 + React 19 + TypeScript, backend w Rust) automatyzująca cotygodniową analizę trace'ów SQL Server: VPN → pobranie trace'ów → import do bazy w schemacie zgodnym z ReadTrace/SQL Nexus → procedury porównujące → raport. Interpretację wyników wspiera lokalny LLM; dane klienta nie opuszczają maszyny.

Status: wczesne MVP — środowisko lokalne gotowe, logika importu i raportu w implementacji.

## Wymagania

- Linux, Node.js, Rust, biblioteki systemowe Tauri (webkit2gtk)
- Docker z Compose (SQL Server Express 2025)
- Opcjonalnie Ollama nasłuchująca na `127.0.0.1:11434`

## Uruchomienie

```sh
# SQL Server: hasło sa w ~/.config/sqlrep/sqlrep.env (MSSQL_SA_PASSWORD=...),
# dane w $SQLREP_DATA_DIR (domyślnie ~/sqlrep-data)
docker compose -f deploy/docker-compose.yml up -d

npm ci
npm run tauri dev
```

Wymagania i decyzje: `context/foundation/` (PRD, stack, infrastruktura), zapis pierwszego wdrożenia: `context/deployment/deploy-plan.md`. Wskazówki dla agentów AI: `AGENTS.md`.

## Licencja

MIT — zobacz [LICENSE](LICENSE).
