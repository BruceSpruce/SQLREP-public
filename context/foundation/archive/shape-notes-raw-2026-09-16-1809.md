## SQLREP - MVP

### Główny problem
Raz w tygodniu muszę ściągnąć przez VPN z firmy, z którą współpracuję, 30 minutowego trace'a z 
SQL Servera, którego następnie wczytuję przez SQL Nexus do bazy MSSQL i wykonuję na niej 
serię zapytań (mam już zrobione procedury do porównywania tych traceów). Następnie ręcznie robię
raport z tego i wysyłam klientowi mailem. Całość jest długotrwała i nudna, ponadto ja nie zawsze 
wyłapuję, co jest istotne, co spowolniło znacząco, albo gubię zależności ilości wykonań danego
zapytania do średniego czasu itp.

### Co już mam
- w podkatalogu context/foundation/samples/queries zamieściłem źródła moich zapytań wraz z przykładami ja je uruchamiam
- w podkatalogu context/foundation/samples/report zamieściłem przykładowy raport wysłany klientowi, robiony ręcznie

### Najmniejszy zestaw funkcjonalności
- automatyczne zestawianie połączenia VPN z klientem
- pobranie lokalnie plików traceów
- zaczytanie tych plików do bazy SQL Nexusa
- zaimportowanie danych do bazy, na której działają procedury porównujące poszczególne trace'y
- przygotowanie dokumentu wyjściowego w formie doc/odt/e-mail

### Co NIE wchodzi w zakres MVP
- uruchomienie trace'a na instancji klienta, trace uruchamia się z joba raz na tydzień

### Kryteria sukcesu
- automatyczne pobieranie traceów, wczytanie do bazy
- analiza wyników z wykorzystaniem AI
- użycie do analizy AI modelu na lokalnej maszynie (np. przez Ollama)
