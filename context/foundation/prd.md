---
project: "SQLREP"
version: 1
status: draft
created: 2026-09-16
context_type: greenfield
product_type: desktop
target_scale:
  users: small
  qps: low
  data_volume: small
timeline_budget:
  mvp_weeks: 3
  hard_deadline: null
  after_hours_only: true
---

## Vision & Problem Statement

Członkowie zespołu świadczącego usługi analizy wydajności SQL Server dla klientów muszą co tydzień ręcznie łączyć się VPN-em z infrastrukturą klienta, ściągać 30-minutowy trace z SQL Servera, wczytywać go przez SQL Nexus do bazy MSSQL, uruchamiać własne procedury porównujące trace'y, a następnie ręcznie składać raport i wysyłać go klientowi mailem. Proces jest długotrwały i monotonny, a wykonująca go osoba nie zawsze wyłapuje to, co istotne — np. gubi zależności między liczbą wykonań danego zapytania a jego średnim czasem wykonania.

Zespół dysponuje już własnymi, sprawdzonymi procedurami SQL do porównywania trace'ów i wie precyzyjnie, jakie zapytania/metryki są istotne dla tego konkretnego typu analizy — to wysoce niszowy, specjalistyczny workflow, dla którego nie istnieje generyczne narzędzie na rynku. Nikt inny tego nie zbudował, bo wymaga to połączenia głębokiej wiedzy domenowej o SQL Server performance tuning z automatyzacją niecodziennego łańcucha czynności (VPN → import do SQL Nexusa → własne procedury porównujące → raport).

## User & Persona

Członek zespołu (konsultant/analityk wydajności SQL Server) wykonujący cykliczne analizy trace'ów dla wielu klientów w ramach współpracy z firmą. Uruchamia analizę raz w tygodniu, po tym jak zaplanowany job u klienta wygenerował trace, i musi dostarczyć klientowi zrozumiały raport. Kilka osób w zespole wykonuje podobne analizy dla różnych klientów — to nie jest scenariusz jednego, nazwanego użytkownika.

## Success Criteria

### Primary
- Pełen przepływ end-to-end działa: automatyczne połączenie VPN → pobranie plików trace'ów → zaczytanie do bazy SQL Nexusa → uruchomienie procedur porównujących → wygenerowanie dokumentu wyjściowego (doc/odt/e-mail). Zaakceptowany termin: 3 tygodnie pracy po godzinach.

### Secondary
- AI-wspierana interpretacja wyników porównania trace'ów, z wykorzystaniem modelu działającego lokalnie na maszynie użytkownika (dane klienta nie są wysyłane do zewnętrznej usługi w chmurze).

### Guardrails
- Poprawność wyników analizy — te same liczby/wnioski co przy dotychczasowym ręcznym procesie; narzędzie nie może wprowadzać klienta w błąd.
- Bezpieczeństwo połączenia VPN i danych klienta — dane nie mogą wyciekać ani zostać pomieszane między różnymi klientami.

## User Stories

### US-01: Cotygodniowa analiza trace'ów klienta

- **Given** zaplanowany job u klienta wygenerował nowy 30-minutowy trace i jest dostępny do pobrania
- **When** użytkownik uruchamia analizę dla tego klienta
- **Then** narzędzie automatycznie łączy się VPN-em, pobiera pliki trace'ów, zaczytuje je do bazy SQL Nexusa, następnie dane z bazy SQL Nexusa, kopiuje do osobnej bazy z zaagregowanymi wynikami, na tej bazie uruchamia procedury porównujące i generuje dokument wyjściowy gotowy do wysłania klientowi

#### Acceptance Criteria
- Połączenie VPN, pobranie plików i import do SQL Nexusa odbywają się bez ręcznej interwencji użytkownika poza uruchomieniem procesu
- Wyniki porównania w dokumencie wyjściowym są zgodne z wynikami, jakie użytkownik otrzymałby, wykonując proces ręcznie
- W razie błędu na którymkolwiek etapie (VPN, pobieranie, import) użytkownik widzi zrozumiały komunikat o tym, co się nie powiodło

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
- FR-006: Użytkownik może otrzymać wsparcie AI, działające lokalnie na jego maszynie (dane klienta nie są wysyłane do zewnętrznej usługi w chmurze), w interpretacji wyników porównania. Priority: nice-to-have
  > Socrates: Rozważono ryzyko niewiarygodnej interpretacji AI bez nadzoru eksperta. Rozstrzygnięcie: kept jako nice-to-have, pod warunkiem że użytkownik zawsze sprawdza cały raport i go zatwierdza przed wysłaniem do klienta — AI wspiera, nie zastępuje przeglądu człowieka.

### Dostęp
- FR-007: Użytkownik może uruchomić i obsłużyć narzędzie jako jeden użytkownik, bez logowania (N/A — brak autoryzacji w MVP). Priority: must-have. Login wielo-osobowy odłożony poza MVP.
  > Socrates: Rozważono, że login wielo-osobowy jest przedwczesny, gdy realnie testuje to jedna osoba. Rozstrzygnięcie: zrewidowano — MVP działa bez logowania (jeden użytkownik, jedna maszyna); multi-user login przesunięty do v2.

## Non-Functional Requirements

- Surowe dane klienta z analizy (baza pośrednia z importu, wygenerowane raporty) nie są przechowywane w aplikacji dłużej niż 3 miesiące po wysłaniu raportu do klienta (okres retencji). Zagregowane metryki zapytań (tabele wyników per klient) są przechowywane dłużej, bo są potrzebne do analizy trendów z wielu trace'ów. Retencją plików trace'ów zarządza użytkownik ręcznie, poza aplikacją.
- Cały proces (od zestawienia VPN do gotowego dokumentu) kończy się w rozsądnym, przewidywalnym czasie od uruchomienia, tak by użytkownik mógł zostawić go do działania bez nadzoru (np. na noc).
- Dane różnych klientów są w pełni odizolowane od siebie — brak możliwości przypadkowego wymieszania wyników między klientami.

## Business Logic

Aplikacja (z wykorzystaniem AI) wykrywa i wyjaśnia zależności w wynikach porównania trace'ów, które umykają manualnej analizie — np. rozpoznaje, że pogorszenie czasu wykonania zapytania wynika ze spadku liczby jego wykonań, a nie z rzeczywistej degradacji wydajności.

Reguła konsumuje wyniki procedur porównujących trace'y uruchomionych z różnymi parametrami (czasy wykonania, liczby odczytów/zapisów, zużycie CPU, liczba wykonań danego zapytania w każdym z porównywanych okresów).

Wynikiem jest zestaw zapytań, których czasy, odczyty, zapisy lub zużycie CPU znacznie się zmieniły i na które trzeba zwrócić uwagę — albo zacząć je obserwować i zweryfikować w kolejnym trace'ie.

Użytkownik trafia na ten wynik na samym końcu procesu — jako finalny etap, tuż przed przygotowaniem dokumentu wyjściowego dla klienta.

## Access Control

MVP: jeden użytkownik, jedno stanowisko/maszyna, brak logowania i rozdzielenia kont (N/A). Login wielo-osobowy (email/hasło / OAuth / passwordless) i model płaski uprawnień są zdecydowane jako kierunek na przyszłość, ale przesunięte poza MVP — patrz runda Sokratejska przy FR-007.

## Non-Goals

- Multi-user / logowanie — przesunięte poza MVP; wersja jednoosobowa, bez autoryzacji.
- Uruchamianie trace'a na instancji klienta — trace generowany jest automatycznie jobem u klienta raz w tygodniu; poza zakresem tego narzędzia.
- Automatyczne wysyłanie raportu do klienta bez przeglądu — zawsze wymagana ręczna weryfikacja i zatwierdzenie raportu przez użytkownika przed wysyłką.
- Obsługa wielu różnych układów plików/struktur trace'ów u różnych klientów — MVP wspiera jeden znany, ustalony układ; inne układy poza zakresem.

## Open Questions

No open questions — all required PRD elements were captured during shaping (shape-notes.md Quality cross-check: accepted, no gaps).
