---
project: "SQLREP"
context_type: greenfield
product_type: desktop
target_scale:
  users: small
  qps: low
  data_volume: small
created: 2026-09-16
updated: 2026-09-16
timeline_budget:
  mvp_weeks: 3
  hard_deadline: null
  after_hours_only: true
checkpoint:
  current_phase: 8
  phases_completed: [1, 2, 3, 4, 5, 6, 7]
  gray_areas_resolved:
    - topic: "pain category"
      decision: "workflow friction + missing capability (no automated/AI analysis) + decision paralysis (unclear what matters in results)"
    - topic: "insight / why not built already"
      decision: "highly niche, specialist workflow — no generic tool exists for this specific SQL Server trace-comparison process"
    - topic: "primary persona scope"
      decision: "several people within the team/company who perform similar analyses for different clients — not a single named user"
    - topic: "auth strategy"
      decision: "login (email/password / OAuth / passwordless) — needed for multi-user, multi-client use"
    - topic: "role model"
      decision: "flat — all logged-in users have the same permissions for MVP"
    - topic: "auth strategy (revised in Socrates round, Step 4.5)"
      decision: "MVP is single-user, single-machine, no login (N/A) — multi-user login deferred beyond MVP; superseded the Step 2 login decision"
    - topic: "AI-assisted interpretation risk (FR-006)"
      decision: "kept as nice-to-have, gated on mandatory human review/approval of the full report before it reaches the client"
    - topic: "product_type"
      decision: "desktop tool (runs locally on user's machine)"
    - topic: "target_scale.users"
      decision: "small — just the user, or a handful of colleagues; domain rule works per-trace and doesn't change at 100x scale"
  frs_drafted: 7
  quality_check_status: accepted
---

## Vision & Problem Statement

Członkowie zespołu świadczącego usługi analizy wydajności SQL Server dla klientów muszą co tydzień ręcznie łączyć się VPN-em z infrastrukturą klienta, ściągać 30-minutowy trace z SQL Servera, wczytywać go przez SQL Nexus do bazy MSSQL, uruchamiać własne procedury porównujące trace'y, a następnie ręcznie składać raport i wysyłać go klientowi mailem. Proces jest długotrwały i monotonny, a wykonująca go osoba nie zawsze wyłapuje to, co istotne — np. gubi zależności między liczbą wykonań danego zapytania a jego średnim czasem wykonania.

Zespół dysponuje już własnymi, sprawdzonymi procedurami SQL do porównywania trace'ów i wie precyzyjnie, jakie zapytania/metryki są istotne dla tego konkretnego typu analizy — to wysoce niszowy, specjalistyczny workflow, dla którego nie istnieje generyczne narzędzie na rynku. Nikt inny tego nie zbudował, bo wymaga to połączenia głębokiej wiedzy domenowej o SQL Server performance tuning z automatyzacją niecodziennego łańcucha czynności (VPN → import do SQL Nexusa → własne procedury porównujące → raport).

## User & Persona

Członek zespołu (konsultant/analityk wydajności SQL Server) wykonujący cykliczne analizy trace'ów dla wielu klientów w ramach współpracy z firmą. Uruchamia analizę raz w tygodniu, po tym jak zaplanowany job u klienta wygenerował trace, i musi dostarczyć klientowi zrozumiały raport. Kilka osób w zespole wykonuje podobne analizy dla różnych klientów — to nie jest scenariusz jednego, nazwanego użytkownika.

## Access Control

MVP: jeden użytkownik, jedno stanowisko/maszyna, brak logowania i rozdzielenia kont (N/A). Login wielo-osobowy (email/hasło / OAuth / passwordless) i model płaski uprawnień są zdecydowane jako kierunek na przyszłość, ale przesunięte poza MVP — patrz runda Sokratejska przy FR-007.

## Success Criteria

### Primary
- Pełen przepływ end-to-end działa: automatyczne połączenie VPN → pobranie plików trace'ów → zaczytanie do bazy SQL Nexusa → uruchomienie procedur porównujących → wygenerowanie dokumentu wyjściowego (doc/odt/e-mail). Zaakceptowany termin: 3 tygodnie pracy po godzinach.

### Secondary
- Analiza wyników z wykorzystaniem AI (lokalny model, np. przez Ollama), wspierająca interpretację wyników porównania trace'ów.

### Guardrails
- Poprawność wyników analizy — te same liczby/wnioski co przy dotychczasowym ręcznym procesie; narzędzie nie może wprowadzać klienta w błąd.
- Bezpieczeństwo połączenia VPN i danych klienta — dane nie mogą wyciekać ani zostać pomieszane między różnymi klientami.

## Functional Requirements

### VPN i pobieranie danych
- FR-001: Użytkownik może automatycznie zestawić połączenie VPN z klientem. Priority: must-have
  > Socrates: Rozważono ryzyko zmiennej konfiguracji VPN u klientów. Rozstrzygnięcie: FR pozostaje bez zmian.
- FR-002: Użytkownik może automatycznie pobrać lokalnie pliki trace'ów z serwera klienta. Priority: must-have
  > Socrates: Rozważono przeciwargument — zbyt szeroki zakres na raz. Rozstrzygnięcie: MVP wspiera najpierw jeden, znany układ plików u klienta; wsparcie dla innych układów przesunięte poza MVP.

### Import i analiza
- FR-003: Użytkownik może zaczytać pobrane pliki trace'ów do bazy SQL Nexusa. Priority: must-have
  > Socrates: Rozważono zmienność formatu trace'ów między klientami i ograniczenia SQL Nexusa. Rozstrzygnięcie: FR pozostaje bez zmian.
- FR-004: Użytkownik może zaimportować dane do bazy z procedurami porównującymi trace'y i uruchomić te procedury. Priority: must-have
  > Socrates: Rozważono ryzyko przeoczenia przypadku szczególnego przy pełnej automatyzacji oraz konieczność aktualizacji procedur. Rozstrzygnięcie: FR pozostaje bez zmian.

### Raportowanie
- FR-005: Użytkownik może wygenerować dokument wyjściowy (doc/odt/e-mail) z wynikami porównania trace'ów. Priority: must-have
  > Socrates: Rozważono ryzyko błędów bez ręcznej korekty i niezgodność formatu z oczekiwaniami klienta. Rozstrzygnięcie: FR pozostaje bez zmian.
- FR-006: Użytkownik może otrzymać wsparcie AI (lokalny model, np. Ollama) w interpretacji wyników porównania. Priority: nice-to-have
  > Socrates: Rozważono ryzyko niewiarygodnej interpretacji AI bez nadzoru eksperta. Rozstrzygnięcie: kept jako nice-to-have, pod warunkiem że użytkownik zawsze sprawdza cały raport i go zatwierdza przed wysłaniem do klienta — AI wspiera, nie zastępuje przeglądu człowieka.

### Dostęp
- FR-007: Użytkownik może uruchomić i obsłużyć narzędzie jako jeden użytkownik, bez logowania (N/A — brak autoryzacji w MVP). Priority: must-have. Login wielo-osobowy odłożony poza MVP.
  > Socrates: Rozważono, że login wielo-osobowy jest przedwczesny, gdy realnie testuje to jedna osoba. Rozstrzygnięcie: zrewidowano — MVP działa bez logowania (jeden użytkownik, jedna maszyna); multi-user login przesunięty do v2.

## User Stories

### US-01: Cotygodniowa analiza trace'ów klienta

- **Given** zaplanowany job u klienta wygenerował nowy 30-minutowy trace i jest dostępny do pobrania
- **When** zalogowany użytkownik uruchamia analizę dla tego klienta
- **Then** narzędzie automatycznie łączy się VPN-em, pobiera pliki trace'ów, zaczytuje je do bazy SQL Nexusa, uruchamia procedury porównujące i generuje dokument wyjściowy gotowy do wysłania klientowi

#### Acceptance Criteria
- Połączenie VPN, pobranie plików i import do SQL Nexusa odbywają się bez ręcznej interwencji użytkownika poza uruchomieniem procesu
- Wyniki porównania w dokumencie wyjściowym są zgodne z wynikami, jakie użytkownik otrzymałby, wykonując proces ręcznie
- W razie błędu na którymkolwiek etapie (VPN, pobieranie, import) użytkownik widzi zrozumiały komunikat o tym, co się nie powiodło

## Business Logic

Aplikacja (z wykorzystaniem AI) wykrywa i wyjaśnia zależności w wynikach porównania trace'ów, które umykają manualnej analizie — np. rozpoznaje, że pogorszenie czasu wykonania zapytania wynika ze spadku liczby jego wykonań, a nie z rzeczywistej degradacji wydajności.

Reguła konsumuje wyniki procedur porównujących trace'y uruchomionych z różnymi parametrami (czasy wykonania, liczby odczytów/zapisów, zużycie CPU, liczba wykonań danego zapytania w każdym z porównywanych okresów).

Wynikiem jest zestaw zapytań, których czasy, odczyty, zapisy lub zużycie CPU znacznie się zmieniły i na które trzeba zwrócić uwagę — albo zacząć je obserwować i zweryfikować w kolejnym trace'ie.

Użytkownik trafia na ten wynik na samym końcu procesu — jako finalny etap, tuż przed przygotowaniem dokumentu wyjściowego dla klienta.

## Non-Functional Requirements

- Dane klienta (trace'y, wyniki analiz) nie są przechowywane dłużej niż ustalony okres retencji po wysłaniu raportu do klienta.
- Cały proces (od zestawienia VPN do gotowego dokumentu) kończy się w rozsądnym, przewidywalnym czasie od uruchomienia, tak by użytkownik mógł zostawić go do działania bez nadzoru (np. na noc).
- Dane różnych klientów są w pełni odizolowane od siebie — brak możliwości przypadkowego wymieszania wyników między klientami.

## Non-Goals

- Multi-user / logowanie — przesunięte poza MVP; wersja jednoosobowa, bez autoryzacji (patrz Faza 4.5).
- Uruchamianie trace'a na instancji klienta — trace generowany jest automatycznie jobem u klienta raz w tygodniu; poza zakresem tego narzędzia.
- Automatyczne wysyłanie raportu do klienta bez przeglądu — zawsze wymagana ręczna weryfikacja i zatwierdzenie raportu przez użytkownika przed wysyłką.
- Obsługa wielu różnych układów plików/struktur trace'ów u różnych klientów — MVP wspiera jeden znany, ustalony układ; inne układy poza zakresem.

## Quality cross-check

All elements present — no gaps. Cross-check ran on 2026-09-16: Access Control (present), Business Logic (present), Project artifacts (present), Timeline-cost ack (present, mvp_weeks=3), Non-Goals (present, 4 entries), Preserved behavior (n/a — greenfield). `quality_check_status: accepted`.

## Reference material (from seed notes)

- `context/foundation/samples/queries/` — istniejące procedury/zapytania SQL używane do porównywania trace'ów (CheckRanking.sql, TopCPU_to_SQL.sql, klient_a.sql, klient_b.sql, NexusDatabaseStructure.sql, klient_c.sql, sp_Analyze_Top_Consumers_With_Trend.sql)
- `context/foundation/samples/report/` — przykładowy raport wysłany klientowi, tworzony ręcznie ("Analiza baseline_a z 20 sierpnia.mbox")
