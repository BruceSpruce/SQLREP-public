---
starter_id: tauri
package_manager: npm
project_name: sqlrep
hints:
  language_family: rust
  team_size: solo
  deployment_target: local
  ci_provider: none
  ci_default_flow: none
  bootstrapper_confidence: verified
  path_taken: standard
  quality_override: false
  self_check_answers: null
  has_auth: false
  has_payments: false
  has_realtime: false
  has_ai: true
  has_background_jobs: true
---

## Why this stack

Narzędzie desktopowe dla jednej osoby/małego zespołu, budowane po godzinach przez 3 tygodnie, uruchamiane na hoście z Linuksem (KVM/libvirt, Docker). Obróbkę plików trace'ów `.trc` (ReadTrace/SQL Nexus, wymagane także dla starszych instancji klientów, SQL Server 2008 i nowszych) wykonuje maszyna wirtualna z Windows 11 uruchamiana tylko na czas importu; natywne wsparcie Windowsa jako hosta aplikacji jest poza zakresem. Tauri to rekomendowany domyślny wybór dla pary `(desktop, rust)` i jedyna pozycja w rejestrze z zweryfikowaną (`verified`) wiarygodnością bootstrappera — łączy backend w Ruście (dobrze nadający się do orkiestracji VPN-a, transferu plików i integracji z SQL Serverem) z lekkim, webowym interfejsem do wygenerowanego raportu. Wybrano ścieżkę standardową — rekomendacja została przyjęta bez zmian. Flagi funkcji AI i zadań w tle są ustawione na podstawie wymagań funkcjonalnych z PRD (lokalne wsparcie AI w interpretacji wyników, uruchomienia bez nadzoru w nocy); logowanie, płatności i funkcje czasu rzeczywistego są jawnie poza zakresem. Dla MVP nie ma dystrybucji: aplikacja jest budowana i uruchamiana lokalnie na maszynie autora (szczegóły w `infrastructure.md`). Domyślny kształt startera to CI na GitHub Actions z automatycznym wdrożeniem po scaleniu do main; na razie nieużywany.
