---
project: sqlrep
researched_at: 2026-09-21
recommended_platform: brak (lokalnie na hoście Linux, Docker)
runner_up: VM z Windows 11 na KVM z ReadTrace (ścieżka zapasowa importu)
context_type: mvp
tech_stack:
  language: Rust + TypeScript
  framework: Tauri 2 (React 19)
  runtime: aplikacja desktopowa (host Linux), npm
---

## Recommendation

**Dla MVP brak wdrożenia i dystrybucji: aplikacja jest budowana i uruchamiana lokalnie na maszynie autora (Linux). Pliki `.trc` importuje własny importer aplikacji przez SQL Server Express w kontenerze Docker (`sys.fn_trace_gettable`), bez Windowsa. Jeśli importer nie przejdzie testu zgodności, zapasową ścieżką jest VM z Windows 11 i ReadTrace.**

Jedynym użytkownikiem jest autor, a aplikacja łączy się po VPN z klientami z jego własnej maszyny. SQLREP to aplikacja desktopowa bez komponentu serwerowego, a dane klientów nie mogą opuszczać maszyny, więc standardowe platformy hostingowe (Cloudflare, Vercel, Netlify, Fly.io, Railway, Render) odpadają na filtrze stacku. Dystrybucja instalatorów wróci do rozważenia, gdy pojawi się druga osoba lub druga maszyna; wtedy trzeba ponownie uruchomić `/10x-infra-research`.

Decyzja autora (2026-09-22): najpierw własny importer, VM budowana tylko wtedy, gdy importer nie spełni kryterium przejścia. Do tego czasu import ręczny przez Nexusa w istniejącej VM z Windows pozostaje dostępny, więc praca z klientami nie jest zagrożona.

Odpowiedzi z wywiadu: aplikacja wymaga długiego procesu lokalnego (VPN + pliki trace'ów, łącznie 5-10 GB), koszt minimalny, brak doświadczenia z platformami, jeden użytkownik, baza MSSQL wyłącznie lokalnie.

## Środowisko

Stan sprawdzony 2026-09-21 na maszynie autora:

- **Host:** Linux z KVM, Docker 29.8; wielordzeniowy CPU i RAM wystarczające do równoległego działania kontenera SQL Server, lokalnego LLM i VM.
- **GPU:** AMD, 16 GB VRAM. Lokalny LLM (FR-006) działa na hoście.
- **Istniejąca VM z Windows** (ręczna obróbka trace'ów w Nexusie; nie modyfikować), z katalogiem trace'ów współdzielonym z hostem.

## Dane i import trace'ów

Punkty oznaczone "do weryfikacji" nie były jeszcze sprawdzone w praktyce.

- **Format:** `.trc` (SQL Trace), trace'y łącznie 5-10 GB na analizę (pliki po ok. 500 MB, sekwencja z jednego dnia to jeden trace). Wspierany zakres instancji klientów: SQL Server 2008 i nowsze (wersje 2008–2017 niezweryfikowane — test pominięty decyzją autora 2026-09-24, wraca przy pierwszym kliencie na starszej wersji).
- **Importer (ścieżka główna):** aplikacja montuje pliki `.trc` jednego klienta tylko do odczytu w kontenerze SQL Server Express 2025 i czyta je przez `sys.fn_trace_gettable`; zdarzenia `RPC:Completed` i `SQL:BatchCompleted` trafiają do bazy pośredniej klienta, normalizator (w Ruście lub T-SQL) wylicza wzorzec zapytania, a z tego powstają tabele odpowiadające `ReadTrace.tblBatches` (`HashID`, `CPU`, `Duration`, `Reads`, `Writes`, `AttnSeq`, `StartTime`) i `ReadTrace.tblUniqueBatches` (`HashID`, `NormText`), z których korzystają procedury autora. Całego schematu ReadTrace nie trzeba odtwarzać.
- **Co już wykazał test (2026-09-22, Klient A, trace z 2026-08-31, 8 plików po 500 MB z SQL Server 2019):** SQL Server 2025 Express w kontenerze czyta całą sekwencję w ok. 20 s (ok. 1,07 mln zdarzeń zakończenia), bez Windowsa. Zakres czasu zgodny z raportem Nexusa (różnice kilka sekund). Dla 17 wzorców z Top N raportu Nexusa, które są wywołaniami procedur, suma wykonań, CPU, Duration, Reads i Writes z grupowania po `ObjectName` zgadza się co do jednostki w 16 przypadkach; w 1 Nexus pokazuje 54% wykonań (prawdopodobnie ta sama nazwa procedury w różnych schematach lub bazach; do wyjaśnienia). Wzorców zapytań ad hoc (44 z 61) nie porównywano, bo wymagają normalizatora. Nie sprawdzono jeszcze trace'ów z SQL Server 2008-2017.
- **Główna trudność - normalizator:** porównania między trace'ami (`sp_Analyze_Top_Consumers_With_Trend`, `sp_Analyze_Performance_Regression`) łączą dane po tekście `[Query Template]`, a filtry opierają się na formacie ReadTrace (np. same nazwy procedur wielkimi literami, prefiks `SET `). Normalizator nie musi dawać identycznych wzorców co ReadTrace, jeśli cała historia zostanie przeliczona nowym importerem (stare pliki `.trc` i raporty Excel Nexusa są zachowane dla klientów); musi natomiast spójnie grupować to samo zapytanie, a filtry w procedurach trzeba dopasować do jego formatu.
- **Kryterium przejścia importera (test na start implementacji, limit czasu 3-4 wieczory):** normalizator + import 2-3 historycznych trace'ów jednego klienta z tych, które są już w lokalnym katalogu trace'ów (bez trace'u ze starszej wersji SQL Server — decyzja autora 2026-09-24); porównanie z raportami Excel Nexusa: co najmniej 95% CPU pokryte wzorcami zgodnymi z Nexusem (lub identyczne sumy przy spójnym grupowaniu) oraz zgodność Top 50. Porównanie liczone lokalnie; na zewnątrz (także do asystenta AI) trafiają tylko wskaźniki zgodności, nigdy teksty zapytań ani wartości klienta. Brak przejścia w limicie czasu = budowa ścieżki zapasowej (VM).
- **Baza (Docker):** SQL Server Express 2025 (`MSSQL_PID=Express`, darmowy produkcyjnie; limit 50 GB na bazę, 4 rdzenie i 1410 MB puli buforów na instancję, sprawdzone w dokumentacji 2026-09-21), pliki danych na katalogu hosta, osobna baza pośrednia i osobna baza wyników na każdego klienta; bazy pośrednie po imporcie to do ok. 1 GB. Przy analizach kilku klientów naraz rozważyć osobny kontener na klienta (limity są wspólne dla instancji).
- **Izolacja klientów:** do kontenera montowane są tylko pliki jednego klienta, tylko do odczytu i tylko na czas importu; nazwy baz i katalogów per klient; baza pośrednia usuwana zgodnie z retencją.
- **Retencja:** 3 miesiące od wysłania raportu (NFR z PRD) dla surowych danych w aplikacji: baza pośrednia z importu i wygenerowane raporty. Zagregowane tabele wyników per klient zostają dłużej (analizy trendów używają do 6 ostatnich trace'ów). Pliki `.trc` usuwa autor ręcznie, poza aplikacją.

## Ścieżka zapasowa: VM z Windows i ReadTrace

Budowana tylko, gdy importer nie spełni kryterium przejścia. Ustalenia z 2026-09-21:

- **Import:** `ReadTrace.exe` z RML Utilities 09.04.0103 (wycofane w paź. 2025, bez dalszych poprawek; formalnie wspiera SQL Server do 2022). Autor używa w Nexusie tylko Tasks -> Import, który uruchamia `ReadTrace.exe` z parametrami, więc wystarczy ReadTrace z linii poleceń (`ReadTrace.exe -I<plik.trc> -o<katalog> -S<serwer> -d<baza> -f`); do sprawdzenia, czy Nexus nie robi nic dodatkowego po imporcie.
- **VM:** nowa, trwała VM projektu zbudowana skryptem od zera (nie istniejącej VM): odchudzony Windows 11 bez aktywacji, instalacja bezobsługowa (`autounattend.xml`), sterowniki virtio, QEMU guest agent lub OpenSSH, .NET 4.8, SQL Server Express 2022 (tylko silnik), RML Utilities, sterownik ODBC/OLEDB. Start i zatrzymanie przez `virsh`, headless, tylko na czas importu. Punkt wyjścia: 4 vCPU / 8 GB RAM.
- **Przepływ:** pliki `.trc` przez współdzielony katalog hosta -> ReadTrace do bazy per klient w VM -> kopiowanie (backup/restore z 2022 do 2025 lub bcp) do bazy w Dockerze -> zatrzymanie VM.
- **Czyszczenie po imporcie:** `DROP DATABASE` bazy klienta, usunięcie katalogu wyjściowego ReadTrace (pliki `.rml` i logi z tekstami zapytań) i `%TEMP%`, restart usługi SQL Server; do sprawdzenia, co jeszcze zostaje w gościu.

## Raportowanie i lokalny LLM

Wymagania z PRD: dokument wyjściowy doc/odt/e-mail (FR-005, must-have), interpretacja wyników wspierana przez AI działające lokalnie (FR-006, nice-to-have), dane klienta nie opuszczają maszyny, raport zawsze przegląda i zatwierdza człowiek. Punkty oznaczone "do weryfikacji" nie były sprawdzone w praktyce.

- **Przepływ:** wyniki procedur porównujących (baza wyników w Dockerze, małe zbiory liczb) -> lokalny LLM interpretuje zależności (np. spadek liczby wykonań zamiast realnej degradacji) -> aplikacja składa dokument lokalnie (odt/doc lub szkic e-maila) -> autor przegląda i zatwierdza -> ręczna wysyłka. Liczby w raporcie pochodzą wyłącznie z procedur SQL; LLM dodaje tylko opis i wnioski, które w raporcie są oznaczone jako wygenerowane.
- **Gdzie działa model:** na hoście Linux. GPU AMD z 16 GB VRAM, co wyznacza górną granicę rozmiaru modelu (orientacyjnie model ok. 14B w kwantyzacji Q4 mieści się w całości; do weryfikacji).
- **Wsparcie GPU (sprawdzone 2026-09-21):** Ollama 0.30.10 wykrywa kartę jako ROCm gfx1200 (15,9 GiB), a test z `qwen2.5-coder:7b` ładuje model w 100% na GPU i generuje ok. 108 tok/s. Oficjalna macierz ROCm dla tej karty wymienia tylko wybrane wersje Ubuntu LTS i RHEL, więc ten host jest poza oficjalnie wspieranym zestawem, mimo że działa. Alternatywą jest llama.cpp z Vulkan (RADV); niezależne testy społeczności pokazują na tej karcie porównywalną lub lepszą prędkość niż ROCm.
- **Konfiguracja Ollama (poprawiona 2026-09-22 przy pierwszym wdrożeniu):** usługa ma teraz `OLLAMA_HOST=127.0.0.1:11434` i `OLLAMA_NO_CLOUD=1`, bez `OLLAMA_ORIGINS=*`; opis stanu sprzed zmiany: nasłuchiwała na wszystkich interfejsach (`OLLAMA_HOST=0.0.0.0:11434`, `OLLAMA_ORIGINS` z `*`) i ma włączone funkcje chmurowe (`OLLAMA_NO_CLOUD` nie ustawione, `OLLAMA_REMOTES=ollama.com`). Dla SQLREP ustawić `OLLAMA_HOST=127.0.0.1:11434` i `OLLAMA_NO_CLOUD=1` albo uruchomić osobną instancję dla aplikacji (zmiana usługi tylko za zgodą autora).
- **Środowisko uruchomieniowe:** Ollama (zainstalowany, działa) albo llama.cpp; wywołanie z Rusta przez lokalny HTTP na `127.0.0.1`; wybór modelu i kwantyzacji do weryfikacji.
- **Jakość po polsku:** raport jest po polsku, więc model wybieramy po jakości opisów technicznych w tym języku; do sprawdzenia na prawdziwych wynikach jednej analizy.
- **Izolacja klientów:** każde uruchomienie to osobna, bezstanowa sesja z kontekstem tylko jednego klienta; brak wspólnej pamięci, cache promptów ani historii między klientami; logi promptów w katalogu danych klienta, usuwane razem z nim.
- **Brak wycieku:** bez telemetrii i bez pobierania modeli w trakcie pracy; wagi modelu pobrane raz i przechowywane lokalnie; cały ruch loopback.
- **Uruchomienia nocne:** krok LLM jest ostatnim etapem; awaria modelu nie blokuje raportu (raport bez interpretacji AI jest ważnym wynikiem, bo FR-006 jest nice-to-have).
- **Zasoby:** pamięć GPU dzieli się z pulpitem hosta; wagi modeli zajmują od kilku do kilkudziesięciu GB na dysku; wolne miejsce trzeba planować razem z trace'ami (5-10 GB na analizę).

## Operational Story

- **Preview deploys**: nie dotyczy.
- **Secrets**: dane dostępowe do VPN i klientów oraz hasło `sa` kontenera wyłącznie lokalnie na maszynie użytkownika; nie commitować ich do repozytorium.
- **Rollback**: powrót do poprzedniego commitu i ponowny lokalny build; import można powtórzyć z zachowanych plików `.trc`.
- **Approval**: raport nigdy nie jest wysyłany automatycznie; wymaga ręcznego zatwierdzenia (reguła z `AGENTS.md`).
- **Logs**: logi z lokalnego uruchomienia aplikacji (`npm run tauri dev`); logi kontenera `docker logs`.

## Risk Register

| Risk | Source | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| Dane klientów lub konfiguracja VPN trafiają do repozytorium | Research finding | M | H | Nie commitować trace'ów ani konfiguracji (`samples/` w `.gitignore`); repozytorium prywatne |
| Frontend wysyła dane klienta poza maszynę (np. przez wstrzyknięty skrypt z tekstu zapytania lub odpowiedzi LLM) | Research finding | L | H | CSP zawężone 2026-09-22 (`connect-src` tylko IPC Tauri); treści klienta renderować jako tekst, nie HTML |
| Normalizator grupuje zapytania inaczej niż ReadTrace, a trendy i filtry w procedurach dają inne wyniki niż proces ręczny | Research finding | M | H | Kryterium przejścia na historycznych trace'ach vs raporty Excel; przeliczenie całej historii nowym importerem; dostosowanie filtrów; ścieżka zapasowa VM |
| `fn_trace_gettable` nie czyta trace'ów ze starszych wersji (2008-2017) | Research finding | L | H | Sprawdzone tylko dla 2019; test starszych wersji odłożony (decyzja autora 2026-09-24) — wykonać przy pierwszym kliencie na wersji 2008–2017 |
| SQL Trace (`fn_trace_gettable`) oznaczony przez Microsoft jako przestarzały; może zniknąć w przyszłej wersji | Research finding | L | M | Przypiąć tag obrazu kontenera (np. `2025-CU9`), nie `latest` |
| Rozbieżność 1 z 17 procedur (54% wykonań) w teście | Research finding | M | L | Grupować po bazie, schemacie i nazwie procedury; wyjaśnić w teście przejścia |
| Dane jednego klienta widoczne przy imporcie innego (wspólny kontener, wolumen, tempdb) | Research finding | M | H | Montowanie plików jednego klienta tylko do odczytu na czas importu; bazy i katalogi per klient; retencja bazy pośredniej |
| Ścieżka zapasowa: bezobsługowa VM z Windows niestabilna; brak guest agent; mało miejsca na dysku; dane klienta zostają w gościu; ReadTrace wycofany | Research finding | M | H | Dotyczy tylko, gdy importer nie przejdzie; szczegóły w sekcji "Ścieżka zapasowa" |
| LLM podaje błędną interpretację lub zmyśla liczby, a raport trafia do klienta | Research finding | M | H | Liczby tylko z SQL; wnioski AI oznaczone w raporcie; obowiązkowe ręczne zatwierdzenie |
| GPU działa poza oficjalnie wspieranym zestawem ROCm; aktualizacja może to zepsuć; model o wymaganej jakości po polsku może nie mieścić się w 16 GB | Research finding | M | M | Przypiąć wersję Ollama; fallback: llama.cpp z Vulkan lub raport bez interpretacji AI |
| Ollama nasłuchuje na `0.0.0.0` z `OLLAMA_ORIGINS=*` i ma włączone funkcje chmurowe (stan faktyczny hosta) | Research finding | H | H | Wykonane 2026-09-22: `OLLAMA_HOST=127.0.0.1:11434`, `OLLAMA_NO_CLOUD=1` w obecnej usłudze (kopia: `override.conf.bak-2026-09-22`); nie używać modeli chmurowych |
| Środowisko uruchomieniowe LLM zapisuje historię promptów lub ma telemetrię | Research finding | M | H | Wyłączyć telemetrię i historię; nasłuch tylko na `127.0.0.1`; logi w katalogu klienta |
| Limity Express (4 rdzenie, 1410 MB puli buforów na instancję) spowalniają import | Research finding | L | L | Odczyt 3,8 GB zajął ok. 20 s; w razie potrzeby osobny kontener na klienta |

## Out of Scope

The following were not evaluated in this research:
- Dystrybucja instalatorów i automatyczne aktualizacje
- Natywne wsparcie Windowsa jako hosta aplikacji
- Wybór konkretnego modelu LLM i biblioteki do generowania dokumentów odt/doc
- Docker image configuration — kontener SQL Server skonfigurowany przy pierwszym wdrożeniu: `deploy/docker-compose.yml` (zapis: `context/deployment/deploy-plan.md`)
- CI/CD pipeline setup
- Production-scale architecture (multi-region, HA, DR)
- Dystrybucja na macOS
