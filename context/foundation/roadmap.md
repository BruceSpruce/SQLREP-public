---
project: SQLREP
version: 1
status: draft
created: 2026-09-24
updated: 2026-09-24
prd_version: 1
main_goal: quality
top_blocker: time
milestone_id: weekly-analysis-end-to-end
milestone_seq: 1
milestone_status: open
---

# Roadmap: SQLREP

> Derived from `context/foundation/prd.md` (v1) + `tech-stack.md`, `infrastructure.md`, `deploy-plan.md` + auto-researched codebase baseline.
> Edit-in-place; archive when superseded.
> Slices below are listed in dependency order. The "At a glance" table is the index.

## Milestone

**M-1: Cotygodniowa analiza od VPN do raportu** — Status: open

- **Intent:** Użytkownik uruchamia dla jednego klienta cotygodniową analizę, a aplikacja sama przechodzi łańcuch VPN → pobranie trace'ów → import → procedury porównujące → dokument do przeglądu, z wynikami zgodnymi z dotychczasowym procesem ręcznym.
- **Source materials:** `context/foundation/prd.md` (v1)
- **Done when:** every F-NN and S-NN below is `done`.
- **Scope anchors:** US-01; FR-001–FR-005, FR-007 (konieczne); FR-006 (opcjonalne); wymagania niefunkcjonalne: izolacja klientów, retencja 3 miesiące, przewidywalny czas przebiegu bez nadzoru.

## Vision recap

Konsultanci wydajności SQL Server co tydzień ręcznie łączą się VPN-em z klientem, pobierają 30-minutowy trace, importują go przez SQL Nexusa, uruchamiają własne procedury porównujące i ręcznie składają raport — proces długi, monotonny i podatny na przeoczenie zależności (np. spadek liczby wykonań udający degradację wydajności). SQLREP automatyzuje ten łańcuch na lokalnej maszynie z Linuksem, bez wysyłania danych klienta poza nią, a raport zawsze zatwierdza człowiek.

## North star

**S-02: Użytkownik uruchamia procedury porównujące na zaimportowanych trace'ach i dostaje te same wyniki co w procesie ręcznym** — to pierwszy punkt, w którym widać, czy automatyzacja nie wprowadza klienta w błąd (warunek poprawności z PRD), więc przy celu `quality` jest dowodem, od którego zależy sens reszty łańcucha.

> "North star" (gwiazda przewodnia) oznacza tu najmniejszy przepływ od początku do końca, którego udane dowiezienie potwierdza główne założenie produktu — dlatego jest ustawiony tak wcześnie, jak pozwalają zależności.

## At a glance

| ID   | Change ID                        | Outcome (user can …)                                                                 | Prerequisites | PRD refs                          | Status   |
| ---- | -------------------------------- | ------------------------------------------------------------------------------------ | ------------- | --------------------------------- | -------- |
| F-01 | importer-compatibility-gate      | (foundation) wiadomo, czy własny importer spełnia kryterium zgodności z Nexusem       | —             | FR-003, US-01, NFR izolacja klientów | ready    |
| F-02 | client-workspace-isolation       | (foundation) jest minimalny profil klienta i kontrakt izolacji danych per klient      | —             | NFR izolacja klientów, FR-007, Access Control | ready    |
| S-01 | import-client-traces             | zaimportować lokalny folder trace'ów klienta do jego odizolowanej bazy pośredniej    | F-01, F-02    | US-01, FR-003, FR-007             | proposed |
| S-02 | trace-comparison-matches-manual  | uruchomić procedury porównujące i zobaczyć wyniki zgodne z procesem ręcznym          | S-01          | US-01, FR-004                     | proposed |
| S-03 | review-and-approve-report        | wygenerować dokument z wynikami, przejrzeć go i ręcznie zatwierdzić                  | S-02          | US-01, FR-005                     | proposed |
| S-04 | client-vpn-connect               | automatycznie zestawić i rozłączyć VPN z klientem, widząc stan i błędy               | F-02          | US-01, FR-001                     | proposed |
| S-05 | download-client-traces           | automatycznie pobrać tygodniowe pliki trace'ów klienta do jego lokalnego katalogu    | S-04, udział sieciowy z trace'ami u klienta | US-01, FR-002 | proposed |
| S-06 | unattended-weekly-run            | jednym poleceniem uruchomić cały przebieg bez nadzoru i zobaczyć, który etap zawiódł | S-03, S-05    | US-01, FR-001, FR-002, FR-003, FR-004, FR-005, NFR przewidywalny czas przebiegu | proposed |
| S-07 | local-ai-interpretation          | dostać w raporcie oznaczoną interpretację wyników od lokalnego modelu AI             | S-03          | FR-006                            | proposed |
| S-08 | raw-data-retention               | zobaczyć i usunąć surowe dane klienta starsze niż 3 miesiące od wysłania raportu     | S-03          | FR-003, FR-005, NFR retencja      | proposed |

## Streams

Navigation aid — groups items that share a Prerequisites chain. Canonical ordering still lives in the dependency graph below; this table is the proposed reading order across parallel tracks.

| Stream | Theme                         | Chain                                                     | Note                                                                                              |
| ------ | ----------------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| A      | Poprawny import i porównanie  | `F-01` → `S-01` → `S-02` → `S-03` → (`S-07`, `S-08`)       | Ścieżka celu `quality`: najpierw zgodność z Nexusem, potem raport; S-07 i S-08 rozgałęziają się po S-03. |
| B      | Dostęp do klienta             | `F-02` → `S-04` → `S-05`                                   | Biegnie równolegle do A (główne ryzyko: czas); F-02 zasila też S-01 w strumieniu A.                |
| C      | Pełny przebieg                | `S-06`                                                    | Łączy strumień A w `S-03` i strumień B w `S-05`.                                                   |

## Baseline

What's already in place in the codebase as of `2026-09-24` (auto-researched + user-confirmed).
Foundations below assume these are present and do NOT re-scaffold them.

- **Frontend:** partial — szablon Tauri 2 + React 19 (`src/App.tsx`), brak ekranów aplikacji.
- **Backend / API:** partial — `src-tauri/src/lib.rs` ma tylko przykładową komendę `greet`.
- **Data:** partial — SQL Server Express 2025 w Dockerze działa (`deploy/docker-compose.yml`, tylko loopback); brak sterownika bazy w backendzie, schematu, baz per klient; procedury porównujące autora istnieją poza aplikacją.
- **Auth:** absent (z założenia) — FR-007: jeden użytkownik, bez logowania; nic do budowania.
- **Deploy / infra:** present — lokalny build i kontener bazy wdrożone (`context/deployment/deploy-plan.md`, status `deployed`); CI celowo nieużywane.
- **Observability:** absent — brak logowania i raportowania błędów per etap; wprowadzane w pierwszych slice'ach, które tego potrzebują (S-01, S-06).
- **Lokalny LLM:** present na hoście — Ollama tylko na loopback, bez funkcji chmurowych; niepodpięty do aplikacji.

## Foundations

### F-01: Test zgodności importera

- **Outcome:** (foundation) normalizator i import 2-3 historycznych trace'ów jednego klienta są porównane z raportami Excel Nexusa, a decyzja „własny importer czy ścieżka zapasowa z VM” jest zapisana razem ze wskaźnikami zgodności (pokrycie CPU, zgodność Top 50).
- **Change ID:** importer-compatibility-gate
- **PRD refs:** FR-003, US-01, NFR izolacja klientów
- **Unlocks:** S-01 (wybór ścieżki importu), weryfikacja zgodności wyników w S-02
- **Prerequisites:** historyczne pliki `.trc` i raporty Excel Nexusa zgromadzone w lokalnym katalogu trace'ów; aktualny trace bazowy skopiowany lokalnie wraz z raportem Nexusa jako punkt odniesienia (poza repozytorium); ta sama sekwencja zostaje też u klienta do testów pobierania w S-05
- **Parallel with:** F-02, S-04
- **Blockers:** —
- **Unknowns:**
  - Skąd rozbieżność liczby wykonań dla 1 z 17 procedur (ta sama nazwa w różnych schematach/bazach?) — Owner: user. Block: no.
- **Risk:** Stoi pierwszy, bo od wyniku zależy kształt S-01; porażka w limicie czasu z `infrastructure.md` oznacza budowę VM i wyraźnie większy zakres importu.
- **Status:** ready

### F-02: Profil klienta i izolacja danych

- **Outcome:** (foundation) istnieje minimalny profil klienta (nazwa, lokalne katalogi, nazwy baz, miejsce na dane dostępowe poza repozytorium) oraz jedna konwencja, która gwarantuje, że żadna ścieżka, baza ani cache nie są współdzielone między klientami.
- **Change ID:** client-workspace-isolation
- **PRD refs:** NFR izolacja klientów, FR-007, Access Control
- **Unlocks:** S-01, S-04 (oba równoległe strumienie potrzebują tej samej konwencji, zanim powstaną dane klienta)
- **Prerequisites:** —
- **Parallel with:** F-01
- **Blockers:** —
- **Unknowns:** —
- **Risk:** Bez wspólnej konwencji równoległe strumienie A i B wymyśliłyby niezgodne układy katalogów i baz; kontrakt jest minimalny — zarządzanie klientami rośnie dopiero w slice'ach, które go używają.
- **Status:** ready

## Slices

### S-01: Import trace'ów klienta

- **Outcome:** Użytkownik wskazuje klienta i lokalny folder z plikami `.trc`, uruchamia import i widzi podsumowanie (liczba zdarzeń, zakres czasu) albo zrozumiały komunikat błędu; dane trafiają wyłącznie do odizolowanej bazy pośredniej tego klienta.
- **Change ID:** import-client-traces
- **PRD refs:** US-01, FR-003, FR-007
- **Prerequisites:** F-01, F-02
- **Parallel with:** S-04, S-05
- **Blockers:** —
- **Unknowns:**
  - Ścieżka importu (własny importer czy VM) — rozstrzyga F-01. Owner: user. Block: no.
- **Risk:** Pierwszy slice, który dotyka danych klienta — tu powstaje wzorzec komunikatów błędów per etap i montowania plików tylko jednego klienta tylko do odczytu.
- **Status:** proposed

### S-02: Porównanie trace'ów zgodne z procesem ręcznym

- **Outcome:** Użytkownik przenosi zaimportowane dane do bazy wyników klienta (zagregowanej, przechowywanej dłużej na potrzeby trendów), uruchamia procedury porównujące i widzi wyniki zgodne z tymi, które dał proces ręczny dla tych samych trace'ów.
- **Change ID:** trace-comparison-matches-manual
- **PRD refs:** US-01, FR-004
- **Prerequisites:** S-01, co najmniej dwa zaimportowane trace'y tego samego klienta
- **Parallel with:** S-04, S-05
- **Blockers:** —
- **Unknowns:**
  - Jak dostosować filtry procedur (oparte na formacie wzorców ReadTrace) do formatu własnego normalizatora? — Owner: user. Block: no.
  - Ile historycznych trace'ów per klient przeliczyć nowym importerem, żeby procedury trendów miały pełną historię? — Owner: user. Block: no.
- **Risk:** To gwiazda przewodnia: jeśli wyniki odbiegają od procesu ręcznego, dalsze kroki automatyzują błąd — dlatego stoi zaraz po imporcie, przed VPN-em i raportem.
- **Status:** proposed

### S-03: Przegląd i zatwierdzenie raportu

- **Outcome:** Użytkownik generuje z wyników porównania dokument wyjściowy, przegląda go w aplikacji i ręcznie oznacza jako zatwierdzony/wysłany; aplikacja niczego nie wysyła sama.
- **Change ID:** review-and-approve-report
- **PRD refs:** US-01, FR-005
- **Prerequisites:** S-02
- **Parallel with:** S-04, S-05
- **Blockers:** —
- **Unknowns:**
  - Który format jako pierwszy: odt, doc czy szkic e-maila (obecny raport ma postać wiadomości e-mail)? — Owner: user. Block: no.
- **Risk:** Liczby w dokumencie mogą pochodzić wyłącznie z procedur SQL; data zatwierdzenia jest potrzebna do retencji w S-08.
- **Status:** proposed

### S-04: Połączenie VPN z klientem

- **Outcome:** Użytkownik z poziomu aplikacji zestawia i rozłącza VPN z wybranym klientem, widzi stan połączenia i zrozumiały komunikat przy błędzie; dane dostępowe są przechowywane lokalnie, poza repozytorium.
- **Change ID:** client-vpn-connect
- **PRD refs:** US-01, FR-001
- **Prerequisites:** F-02
- **Parallel with:** F-01, S-01, S-02, S-03, S-07, S-08
- **Blockers:** —
- **Unknowns:**
  - Rozstrzygnięte: VPN pierwszego wspieranego klienta da się zestawić z hosta bez interakcji, a dane dostępowe są przechowywane lokalnie poza repozytorium.
  - Czy przebieg nocny (S-06) ma dostęp do danych dostępowych VPN, gdy sesja użytkownika jest zablokowana? — Owner: user. Block: no.
- **Risk:** Zależy od infrastruktury klienta, więc biegnie równolegle do strumienia importu, by nie blokować dowodu poprawności; MVP obsługuje VPN jednego klienta, pozostali — patrz Open Roadmap Questions.
- **Status:** proposed

### S-05: Pobranie trace'ów klienta

- **Outcome:** Po zestawieniu VPN użytkownik pobiera automatycznie pliki trace'ów z bieżącego tygodnia do lokalnego katalogu klienta i widzi, co zostało pobrane albo co się nie udało.
- **Change ID:** download-client-traces
- **PRD refs:** US-01, FR-002
- **Prerequisites:** S-04, udział sieciowy z trace'ami u klienta
- **Parallel with:** S-01, S-02, S-03, S-07, S-08
- **Blockers:** —
- **Unknowns:**
  - Rozstrzygnięte: wspierany w MVP układ plików to udział sieciowy u klienta, z którego host pobiera trace'y przez SMB po zestawieniu VPN; łączenie po adresie z profilu klienta.
  - W katalogu źródłowym mogą leżeć inne pliki, a część jest w trakcie zapisu. Pobieranie musi brać tylko sekwencję trace'u bazowego z danego tygodnia i tylko pliki już zamknięte (np. po zakończeniu okna 30 minut lub przy stabilnym rozmiarze). Po pobraniu i sprawdzeniu kompletności (rozmiar lub suma kontrolna) aplikacja usuwa pobrane pliki u klienta, nie dotykając innych plików ani plików jeszcze niepobranych. — Owner: user. Block: no.
- **Risk:** Duży wolumen (kilka GB na analizę) po VPN-ie — przerwany transfer musi być widoczny, a nie po cichu dawać niepełny trace do importu.
- **Status:** proposed

### S-06: Cotygodniowy przebieg bez nadzoru

- **Outcome:** Użytkownik jednym poleceniem uruchamia dla klienta cały łańcuch VPN → pobranie → import → porównanie → dokument, może zostawić go bez nadzoru (np. na noc), a po zakończeniu widzi gotowy dokument albo etap, na którym przebieg się zatrzymał, z czytelnym komunikatem.
- **Change ID:** unattended-weekly-run
- **PRD refs:** US-01, FR-001, FR-002, FR-003, FR-004, FR-005, NFR przewidywalny czas przebiegu
- **Prerequisites:** S-03, S-05
- **Parallel with:** S-07, S-08
- **Blockers:** —
- **Unknowns:** —
- **Risk:** Składa już działające kroki, więc ryzyko leży w obsłudze błędów między etapami i rozłączeniu VPN-u po awarii, a nie w samych krokach.
- **Status:** proposed

### S-07: Interpretacja wyników przez lokalny model AI

- **Outcome:** Użytkownik dostaje w dokumencie opis zależności w wynikach (np. wzrost czasu wynikający ze spadku liczby wykonań), wygenerowany przez model działający lokalnie i oznaczony jako wygenerowany; awaria modelu nie blokuje raportu.
- **Change ID:** local-ai-interpretation
- **PRD refs:** FR-006
- **Prerequisites:** S-03
- **Parallel with:** S-04, S-05, S-06, S-08
- **Blockers:** —
- **Unknowns:**
  - Który lokalny model daje wystarczającą jakość opisów technicznych po polsku w dostępnej pamięci GPU? — Owner: user. Block: no.
- **Risk:** Wymaganie opcjonalne — przy braku czasu to pierwszy kandydat do przeniesienia do następnego kamienia milowego; ryzykiem jest błędna interpretacja, dlatego raport zawsze zatwierdza człowiek.
- **Status:** proposed

### S-08: Retencja surowych danych

- **Outcome:** Użytkownik widzi, które bazy pośrednie i wygenerowane raporty klienta przekroczyły 3 miesiące od wysłania raportu, i usuwa je, zachowując zagregowane tabele wyników potrzebne do trendów.
- **Change ID:** raw-data-retention
- **PRD refs:** FR-003, FR-005, NFR retencja
- **Prerequisites:** S-03
- **Parallel with:** S-04, S-05, S-06, S-07
- **Blockers:** —
- **Unknowns:** —
- **Risk:** Usuwanie danych musi respektować granice klientów i nigdy nie dotykać zagregowanych wyników ani plików `.trc` (te użytkownik usuwa ręcznie).
- **Status:** proposed

## Backlog Handoff

| Roadmap ID | Change ID                       | Suggested issue title                                        | Ready for `/10x-plan` | Notes |
| ---------- | ------------------------------- | ------------------------------------------------------------ | --------------------- | ----- |
| F-01       | importer-compatibility-gate     | Test zgodności importera z raportami Nexusa                  | yes                   | Run `/10x-plan importer-compatibility-gate` |
| F-02       | client-workspace-isolation      | Profil klienta i kontrakt izolacji danych                    | yes                   | Run `/10x-plan client-workspace-isolation` |
| S-01       | import-client-traces            | Import folderu trace'ów do bazy pośredniej klienta           | no                    | Czeka na F-01, F-02 |
| S-02       | trace-comparison-matches-manual | Procedury porównujące z wynikami zgodnymi z procesem ręcznym | no                    | Czeka na S-01 |
| S-03       | review-and-approve-report       | Generowanie, przegląd i zatwierdzenie raportu                | no                    | Czeka na S-02 |
| S-04       | client-vpn-connect              | Automatyczne połączenie VPN z klientem                       | no                    | Czeka na F-02 |
| S-05       | download-client-traces          | Automatyczne pobieranie trace'ów klienta                     | no                    | Czeka na S-04 |
| S-06       | unattended-weekly-run           | Cotygodniowy przebieg bez nadzoru z raportem błędów etapów   | no                    | Czeka na S-03, S-05 |
| S-07       | local-ai-interpretation         | Interpretacja wyników przez lokalny model AI                 | no                    | Czeka na S-03; opcjonalne |
| S-08       | raw-data-retention              | Retencja surowych danych klienta (3 miesiące)                | no                    | Czeka na S-03 |

## Open Roadmap Questions

1. **Jeśli F-01 nie przejdzie i trzeba budować ścieżkę zapasową z VM, czy S-07 (AI) przechodzi do następnego kamienia milowego, żeby zmieścić się w budżecie z PRD?** — Owner: user. Block: — (zmienia zakres S-01 i miejsce S-07, nie blokuje planowania F-01/F-02).
2. **Czy analizy kilku klientów mogą biec jednocześnie (osobny kontener bazy na klienta), czy MVP zakłada jeden przebieg naraz?** — Owner: user. Block: — (dotyczy S-01 i S-06; domyślnie jeden przebieg naraz).
3. **Kiedy dołączyć VPN pozostałych klientów (inne konfiguracje VPN)?** — Owner: user. Block: — (MVP: VPN jednego klienta; kandydat do kolejnego kamienia milowego).

(PRD nie ma otwartych pytań — powyższe wynikły przy układaniu roadmapy.)

## Parked

- **Multi-user / logowanie** — Why parked: PRD §Non-Goals, FR-007 (przesunięte do v2).
- **Uruchamianie trace'a na instancji klienta** — Why parked: PRD §Non-Goals (trace generuje job u klienta).
- **Automatyczne wysyłanie raportu bez przeglądu** — Why parked: PRD §Non-Goals; raport zawsze zatwierdza użytkownik.
- **Obsługa wielu układów plików trace'ów u różnych klientów** — Why parked: PRD §Non-Goals (MVP: jeden znany układ).
- **Ścieżka zapasowa importu przez VM z Windows i ReadTrace** — Why parked: `infrastructure.md` — budowana tylko, jeśli F-01 nie przejdzie kryterium.
- **Analiza blokad, deadlocków i sesji Extended Events (XE)** — Why parked: poza obecnym PRD (US-01/FR-003 obejmują trace bazowy) — kandydat na kolejny kamień milowy po aktualizacji PRD przez `/10x-prd`.
- **Test zgodności importera z trace'ami z SQL Server 2008–2017** — Why parked: decyzja autora (2026-09-24) — F-01 opiera się tylko na trace'ach dostępnych lokalnie; wraca, gdy pojawi się klient na starszej wersji.
- **Dystrybucja instalatorów i aktualizacje** — Why parked: `infrastructure.md` §Out of Scope (jeden użytkownik, jedna maszyna).

## Milestone History

## Done
