---
project: sqlrep
planned_at: 2026-09-22
status: deployed
deployed_at: 2026-09-22
based_on:
  - context/foundation/infrastructure.md
  - context/foundation/tech-stack.md
---

# Pierwsze wdrożenie SQLREP (lokalne środowisko)

## Kontekst

`infrastructure.md` ustala: brak dystrybucji, aplikacja działa lokalnie na hoście Linux; import `.trc` przez własny importer w SQL Server Express 2025 w Dockerze (`fn_trace_gettable`); lokalny LLM (Ollama) tylko na loopback, bez chmury. "Wdrożenie" = przygotowanie tego środowiska, żeby implementacja miała gdzie działać. VM z Windows (ścieżka zapasowa) nie jest budowana.

Decyzje autora: zmiana obecnej usługi Ollama (nie osobna instancja); dane SQL w `$SQLREP_DATA_DIR/mssql`.

Stan sprawdzony 2026-09-22: Docker 29.8 + Compose v5.5.1, użytkownik w grupie `docker`; katalog danych należy do użytkownika; obraz `mcr.microsoft.com/mssql/server:2025-CU9-ubuntu-24.04` istnieje; port 1433 wolny; Ollama override: `OLLAMA_HOST=0.0.0.0`, `OLLAMA_ORIGINS=*`.

Ograniczenia: nie ruszać istniejącej VM z Windows; żadnych danych klientów, haseł ani plików `.env` w repozytorium i w odpowiedziach agenta; nic z katalogu trace'ów na hoście nie jest montowane w tym wdrożeniu.

## Kroki

### 1. SQL Server Express 2025 w Dockerze (agent)

- Katalogi: `$SQLREP_DATA_DIR/mssql/{data,log,backup}`. Kontener działa jako uid 10001 → **bramka ręczna**: `sudo chown -R 10001:0 $SQLREP_DATA_DIR/mssql/{data,log,backup}`.
- Hasło `sa`: generowane (`openssl rand`), zapis do `~/.config/sqlrep/sqlrep.env` (chmod 600, poza repozytorium) jako `MSSQL_SA_PASSWORD=...`. Nie wypisywane na ekran.
- Nowy plik w repozytorium `deploy/docker-compose.yml`:
  - image `mcr.microsoft.com/mssql/server:2025-CU9-ubuntu-24.04` (przypięty tag, nie `latest`),
  - `ACCEPT_EULA=Y`, `MSSQL_PID=Express`, `env_file: ${HOME}/.config/sqlrep/sqlrep.env`,
  - `ports: "127.0.0.1:1433:1433"` (tylko loopback),
  - wolumeny: `$SQLREP_DATA_DIR/mssql/data:/var/opt/mssql/data`, `.../log:/var/opt/mssql/log`, `.../backup:/var/opt/mssql/backup`,
  - `container_name: sqlrep-mssql`, `restart: unless-stopped`, healthcheck przez `sqlcmd -C -Q "SELECT 1"`.
- Uruchomienie: `docker compose -f deploy/docker-compose.yml up -d`.
- Montowanie trace'ów klienta (tylko do odczytu, na czas importu) to zadanie implementacji importera, nie tego wdrożenia.

### 2. Ollama tylko lokalnie (bramka ręczna – sudo)

- Kopia obecnego pliku: `override.conf.bak-2026-09-22`.
- Nowa treść `/etc/systemd/system/ollama.service.d/override.conf`:
  ```
  [Service]
  Environment="OLLAMA_HOST=127.0.0.1:11434"
  Environment="OLLAMA_NO_CLOUD=1"
  ```
  (usunięte `OLLAMA_ORIGINS=*`; domyślne originy obejmują localhost i `tauri://`).
- `sudo systemctl daemon-reload && sudo systemctl restart ollama`.
- Skutek: inne urządzenia w sieci tracą dostęp do Ollama (świadoma decyzja autora).
- Bez pobierania nowych modeli (wybór modelu = implementacja raportu).

### 3. Build aplikacji (agent)

- `npm ci`, `npm run build`, `cargo check --manifest-path src-tauri/Cargo.toml`.
- `npm run tauri build -- --no-bundle` → binarka `src-tauri/target/release/sqlrep` (bez instalatorów; `targets: "all"` wymagałby pobierania narzędzi AppImage).
- `npm run tauri dev` – okno startuje, w konsoli WebView brak błędów CSP (sprawdza autor wizualnie – **bramka ręczna**).

### 4. Zapis wdrożenia (agent)

- Ten plik: dopisać sekcję "Wynik wdrożenia" (co wdrożono, wersje, ścieżki, lokalizacja pliku z hasłem bez wartości, porty, polecenia start/stop/logi, odstępstwa od planu); `status: deployed`.
- `infrastructure.md`: w ryzyku Ollama mitygacja → "wykonane"; "Docker image configuration" w Out of Scope → odnośnik do `deploy/docker-compose.yml`.
- `AGENTS.md`, sekcja komend: `docker compose -f deploy/docker-compose.yml up -d|down|logs`, gdzie leży hasło `sa`.
- Commit (`chore: add local SQL Server container and deployment record`); push tylko na prośbę.

## Czego plan nie robi

Nie buduje VM z Windows, nie modyfikuje istniejącej VM, nie importuje trace'ów, nie tworzy schematu baz ani procedur (implementacja), nie pobiera modeli LLM, nie tworzy instalatorów.

## Weryfikacja

- `docker ps` → `sqlrep-mssql` healthy; `SELECT SERVERPROPERTY('Edition'), SERVERPROPERTY('ProductVersion')` przez `sqlcmd` w kontenerze → Express, 17.0.5005.x (hasło z pliku env, niewyświetlane).
- `ss -ltn` → 1433 i 11434 tylko na `127.0.0.1`; `curl -s --max-time 3 http://<IP-LAN>:11434` → brak połączenia; `curl http://127.0.0.1:11434/api/version` → OK.
- `systemctl show ollama -p Environment` zawiera `OLLAMA_NO_CLOUD=1`.
- Trwałość danych: utworzyć bazę testową `sqlrep_smoke`, `docker compose down && up -d`, baza istnieje, potem ją usunąć.
- `git status` → brak `.env`/haseł w repozytorium; `git grep -n MSSQL_SA_PASSWORD=` → tylko nazwa zmiennej.
- Binarka `src-tauri/target/release/sqlrep` istnieje; okno aplikacji uruchamia się.

## Rollback

- Baza: `docker compose -f deploy/docker-compose.yml down` (dane zostają w `$SQLREP_DATA_DIR/mssql`; usunięcie katalogu = decyzja autora).
- Ollama: przywrócić `override.conf.bak-2026-09-22`, `sudo systemctl daemon-reload`, `sudo systemctl restart ollama`.

## Wynik wdrożenia (2026-09-22)

### Co wdrożono

| Element | Stan |
| --- | --- |
| SQL Server | kontener `sqlrep-mssql`, obraz `mcr.microsoft.com/mssql/server:2025-CU9-ubuntu-24.04`, Express Edition (64-bit) 17.0.5005.3, healthy |
| Konfiguracja | `deploy/docker-compose.yml` (w repozytorium) |
| Dane SQL | `$SQLREP_DATA_DIR/mssql/{data,log,backup}`, właściciel `10001:0` |
| Hasło `sa` | `~/.config/sqlrep/sqlrep.env` (chmod 600, katalog 700, poza repozytorium; wartość nigdzie nie wypisana) |
| Porty | `127.0.0.1:1433` (SQL Server), `127.0.0.1:11434` (Ollama) — oba tylko loopback |
| Ollama | 0.30.10, `OLLAMA_HOST=127.0.0.1:11434`, `OLLAMA_NO_CLOUD=1`; kopia poprzedniej konfiguracji: `/etc/systemd/system/ollama.service.d/override.conf.bak-2026-09-22` |
| Aplikacja | `src-tauri/target/release/sqlrep` (release, bez instalatorów); `npm ci` 0 podatności, `npm run build` i `cargo check` bez błędów |

### Polecenia

- Start: `docker compose -f deploy/docker-compose.yml up -d`
- Stop: `docker compose -f deploy/docker-compose.yml down`
- Logi: `docker compose -f deploy/docker-compose.yml logs -f`
- Zapytanie w kontenerze: `docker exec sqlrep-mssql bash -c '/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "$MSSQL_SA_PASSWORD" -C -Q "SELECT 1"'`
- Ollama: `systemctl status ollama`, `curl http://127.0.0.1:11434/api/version`

### Weryfikacja

- `sqlrep-mssql` healthy; edycja Express, wersja 17.0.5005.3.
- Trwałość: baza `sqlrep_smoke` przetrwała `down` + `up -d`, następnie usunięta.
- `ss -ltn`: 1433 i 11434 tylko na `127.0.0.1`; `curl` na adres LAN hosta:11434 → brak połączenia; `/api/version` na loopback → OK.
- `systemctl show ollama -p Environment` zawiera `OLLAMA_NO_CLOUD=1`.
- Binarka release istnieje; `npm run tauri dev` sprawdzony przez autora (zrzut ekranu z 11:59): okno `sqlrep` renderuje szablon, a w konsoli WebView nie ma błędów ani ostrzeżeń CSP (tylko `[vite] connected`).

### Odstępstwa od planu

- Obrazu SQL Server nie było lokalnie (plan zakładał, że jest) — pobrany z MCR przy pierwszym użyciu, digest `sha256:2b5b581621126574f3d1f75e78d3eebe8d05aedb59ad0cfdf9aa42cb0634d726`.
- Healthcheck loguje się jako `sa` z hasłem ze zmiennej środowiskowej kontenera (`sqlcmd -S localhost -U sa -C -Q "SELECT 1"`); plan podawał samo `sqlcmd -C -Q "SELECT 1"`, które bez poświadczeń nie działa.
- Bramki ręczne z `sudo` przez prefiks `!` w sesji agenta działają tylko wtedy, gdy `sudo` ma jeszcze zapamiętane hasło; prefiks `!` nie daje terminala do wpisania hasła. Bramka 1 (`chown`) i kopia zapasowa `override.conf` przeszły w ten sposób. Podmiana `override.conf` przy pierwszej próbie się nie udała, a ponowna próba z `!` skończyła się błędem `sudo: A terminal is required to authenticate`. Zmianę Ollama wykonano w osobnym terminalu. Na przyszłość: polecenia z `sudo` uruchamiać w zwykłym terminalu.
