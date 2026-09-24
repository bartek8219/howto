# Zadanie: wybierz model zgłoszeń SCA w Jira pod mechanizm Issue-2-PR (warianty C, D, E)

Jesteś architektem. Przeanalizuj problem i zaproponuj rozwiązanie. Nie masz dostępu do
repozytorium ani do Jira. Poniżej masz pełny opis stanu faktycznego i ustaleń z zespołem.
Ustalenia oznaczone jako **potwierdzone** traktuj jako fakty. Resztę traktuj jako założenia:
wypisz je jawnie i wskaż, jak je sprawdzić.

## Kontekst — co robi system

Pipeline SCA uruchamiany w TeamCity:

1. **Wejście: artefakt `src.zip`** dostarczany przez zespół aplikacji. Zawiera kod aplikacji
   i zrestorowane biblioteki. **Potwierdzone:** pipeline nie wie, z którego brancha pochodzi kod
   (to wybór zespołu), i **nie ma dostępu do Bitbucket**. Tego nie zmieniamy.
2. **Black Duck** skanuje zależności i produkuje raport.
3. **`bdjira diff`** (CLI w .NET) porównuje raport z poprzednim skanem i wypluwa `filtered.json`
   (SARIF) z podziałem na `new` / `changed` / `unchanged` oraz listę `disappeared`.
4. **Agent LLM (remediacja)** czyta `filtered.json` i produkuje `remediation.json` + `remediation.md`:
   per komponent zbiór CVE, wersja docelowa, werdykt, uzasadnienie, ryzyko kompatybilności.
   **Potwierdzone:** gdy nie da się podbić biblioteki bezpośredniej, agent proponuje obejście,
   np. podbicie biblioteki tranzytywnej przez pinning (jawny `PackageReference` albo
   `CentralPackageTransitivePinningEnabled`). Brak pliku = brak zmian.
5. **`bdjira sync`** zakłada i aktualizuje zgłoszenia w Jira na podstawie `remediation.json`.

Stack: .NET 10, **Jira Data Center, REST API v2**, Bitbucket Data Center, TeamCity. Skanowane
aplikacje to projekty .NET (NuGet). **Potwierdzone:** Jira i Bitbucket są zintegrowane
(z Jira można utworzyć branch dla zgłoszenia, działa panel Development).

## Mechanizmy, które już działają i są kontraktem

- **Dedup oparty na etykietach Jira** (globalnych, więc działa cross-project):
  - `bd-scan` — marker „zarządzane przez narzędzie”
  - `bd-app-{app}` — filtr per aplikacja
  - `bd-vuln-{identityHash}` — klucz dedup, gdzie
    `identityHash = sha256(cve + component + version + app)`
  - `bd-content-{contentHash}` — hash treści zgłoszenia
  - `bd-locked-by-human` — ręczna blokada, narzędzie nie rusza takiego zgłoszenia
- **Potwierdzone:** jeśli kolejny skan zawiera te same podatności, sync szuka **otwartego**
  zgłoszenia po kluczu dedup, żeby nie tworzyć duplikatów. Stan lokalny to tylko cache, a źródłem
  prawdy jest Jira.
- **Trójpoziomowa detekcja zmian:**
  1. hash całego raportu → pomija krok LLM;
  2. `identityHash` / `contentHash` per znalezisko → do LLM idą tylko `new` i `changed`;
  3. `bd-content-{hash}` na zgłoszeniu → jeśli bez zmian, synchronizacja to no-op (brak spamu komentarzami).

  Rozmycie któregokolwiek poziomu kosztuje tokeny LLM albo zasypuje Jira komentarzami.
- **Zmiana wejść `identityHash` lub separatora wymaga wersjonowania etykiety i migracji.**
  Inaczej wszystkie istniejące zgłoszenia wyglądają na nowe i powstają masowe duplikaty.
- Obecnie `remediation.json` i `remediation.md` (raport **całej aplikacji**) są załączane do
  **każdego** tworzonego i aktualizowanego zgłoszenia. W Jira usuwanie załączników bywa zablokowane
  uprawnieniami, więc załączniki się kumulują. Linkowanie do artefaktów TeamCity odpada (brak dostępu
  + retencja).
- **W produkcji (wariant 1):** osobne zgłoszenie per komponent (jeden bump zamyka zwykle kilka CVE).
  To punkt wyjścia do migracji.

## Mechanizm Issue-2-PR (ustalenia potwierdzone)

- Osobny agent AI realizuje zgłoszenia i proponuje PR do zatwierdzenia przez człowieka.
  **Nie ma dostępu do TeamCity.** Ma dostęp do Jira i Bitbucket.
- **Zgłoszenia do realizacji wybiera człowiek**, przypisując je do użytkownika technicznego agenta.
  Nie zakładamy, że wszystkie zgłoszenia zostaną zrealizowane przed kolejnym skanem. Backlog może
  trwać przez wiele skanów.
- **Agent obsługuje jedno zgłoszenie naraz i nie widzi hierarchii.** Nie zagląda do zgłoszenia
  nadrzędnego (epic / zadanie główne) ani do podrzędnych (subtaski). Zlecenie mu zadania głównego
  nie obejmuje subtasków, więc każdy subtask trzeba zlecić osobno.
- **Praca odbywa się w dialogu przez komentarze** w zgłoszeniu: seria pytań AI, odpowiedzi
  człowieka, zatwierdzenie planu, dopiero potem realizacja i PR. Każde zlecone zgłoszenie to osobny
  taki dialog.
- **Agent jest konfigurowalny instrukcjami, ale nie kodem.** Możemy mu dać instrukcje, np.
  „aktualizuj listę TASKS w opisie”. Nie wiadomo z góry, które zachowania da się tak wymusić
  (np. branch bazowy PR, czytanie załączników lub entity properties, zapis do opisu zgłoszenia).
  Wskaż, które instrukcje są potrzebne i co robić, jeśli któraś okaże się niewykonalna.
- Nie wiadomo, czy agent ma .NET SDK i dostęp do feedu NuGet, czyli czy może zregenerować
  `packages.lock.json` i zbudować projekt. Potraktuj to jako założenie do weryfikacji i pokaż wpływ.
- Jeśli agent coś przeoczy, znalezisko i tak wróci do realizacji przy następnym skanie po merge PR.

## Warianty do porównania — wyłącznie te trzy

- **C — zgłoszenie zbiorcze per aplikacja, per cykl.** Jedno zgłoszenie obejmuje wszystkie
  podatności aplikacji w danym cyklu. **Potwierdzone:** zgłoszenie jest **per cykl**, a nie wieczne.
  Zamyka się po merge i weryfikacji skanem, a kolejne znaleziska trafiają do następnego zgłoszenia
  cyklu. Zaletą z perspektywy Issue-2-PR jest jeden dialog, jedno zatwierdzenie planu i jeden PR
  na aplikację.
- **D — zgłoszenie główne per aplikacja + subtaski per komponent.**
- **E — epic per aplikacja + zwykłe zadania per komponent (Epic Link).**

**Potwierdzone:** z perspektywy Issue-2-PR **D i E działają praktycznie identycznie**. Każdy
komponent to osobne zlecenie, osobny dialog w komentarzach i osobny PR. Różnice między D i E
oceniaj tylko poza Issue-2-PR: board, sprinty, metryki, migracja.

## Lista TASKS w zgłoszeniu (ustalenie potwierdzone)

Chcemy listę zadań w opisie zgłoszenia, w stylu OpenSpec: jeden punkt na znalezisko lub
komponent, ze stanem (do zrobienia / zrobione / odłożone z uzasadnieniem / przeoczone). Służy do
rozliczalności: co agent Issue-2-PR wykonał, co odłożył, a co przeoczył.

**Potwierdzone:** listę **tworzy `bdjira sync`, agent tylko zmienia stan punktów.** Punkty mają
stabilne ID (np. skrót `identityHash`), a sync przy aktualizacji zachowuje ich stany.

Do rozstrzygnięcia w analizie:
- **Współbieżna edycja opisu.** Opis zmieniają dwa procesy (sync w nocy, agent w trakcie pracy),
  a REST v2 zapisuje opis w całości i nie sprawdza, czy ktoś zmienił go w międzyczasie. Wskaż,
  jak uniknąć utraty zmian, i porównaj z alternatywami (np. osobne pole custom field na TASKS).
- **Relacja z `bd-content-{hash}`.** Odhaczanie punktów przez agenta nie może wyglądać dla synca
  jak zmiana treści.
- **Format.** Jira DC nie ma natywnych checkboxów w opisie (wiki markup). Zaproponuj format
  czytelny dla człowieka i jednoznaczny dla agenta, z przykładem.
- **Co sync robi z punktami** w typowych sytuacjach:
  - znalezisko zniknęło w skanie;
  - pojawiło się nowe w trakcie trwającego cyklu (C) lub trwającej pracy agenta (D/E);
  - agent oznaczył punkt jako zrobiony, a skan nadal go widzi.

## Inne ustalenia

- **SLA per severity nie jest wymagane.** Nie traktuj go jako kryterium. Priorytet zgłoszenia może
  służyć wyłącznie widoczności.
- **Zgłoszeń nie przekazujemy do innych zespołów. To nie jest kryterium.** Skan dotyczy naszej
  aplikacji i naszego repo, więc zgłoszenie zawsze należy do zespołu aplikacji. Jeśli podatna
  biblioteka przychodzi tranzytywnie z wewnętrznego pakietu innego zespołu, ten zespół skanuje
  własny kod i dostaje własne zgłoszenia ze swojego skanu. U nas rozwiązaniem jest pinning
  biblioteki tranzytywnej lub inne obejście z remediacji. Jeśli nawet to jest niemożliwe (np. pin
  łamie pakiet wewnętrzny), punkt jest **odkładany** z komentarzem. Oceń, co odłożenie oznacza dla
  zamknięcia cyklu (C) i zgłoszenia (D/E) oraz czy odłożony punkt przechodzi do następnego cyklu.
- Domyślny limit długości pól tekstowych w Jira DC to 32 767 znaków
  (`jira.text.field.character.limit`). Ma to znaczenie dla opisu zbiorczego w C.

## Problemy do rozstrzygnięcia

Uwzględnij każdy z nich dla C, D i E:

1. **Dopasowanie do Issue-2-PR:** liczba zleceń, dialogów, zatwierdzeń planu i PR-ów na aplikację;
   nakład pracy człowieka.
2. **Konflikty i poprawność PR-ów:** pliki lock/manifest; build walidujący graf zależności, który
   po merge innego PR przestaje obowiązywać; zachowanie przy squash-merge. Jeśli wariant daje wiele
   PR-ów per aplikacja, oceń strategie branchy (niezależne PR, stacked PRs, inne) w granicach tego,
   co da się wymusić instrukcjami.
3. **Detekcja zmian:** granularność `bd-content-{hash}` i ryzyko spamu komentarzami, gdy jedno
   zgłoszenie obejmuje wiele podatności (C) lub gdy zgłoszenie główne agreguje treść (D).
4. **Cykl C w szczegółach:**
   - co wyznacza początek i koniec cyklu;
   - jak wygląda klucz dedup zgłoszenia cyklu obok `bd-vuln-*` (etykiety per znalezisko na jednym
     zgłoszeniu);
   - co z nowymi znaleziskami, gdy zgłoszenie cyklu jest już w realizacji lub ma zatwierdzony plan:
     dopisać do bieżącego cyklu (zmiana zakresu w trakcie) czy odłożyć do następnego (dwa otwarte
     zgłoszenia per aplikacja i dedup po obu)?
   - co z punktami odłożonymi i przeoczonymi przy zamknięciu cyklu.
5. **Lista TASKS i rozliczalność:** kontrakt sync ↔ agent (kto co zapisuje i gdzie), współbieżność,
   format, reguły przejść stanów punktu.
6. **Cykl życia i domknięcie pętli:** kiedy i przez kogo zgłoszenie (lub punkt) zamyka się po
   merge. Skan obejmuje to, co zespół dostarczy w `src.zip` z dowolnego brancha. Stąd ryzyko
   migotania (znalezisko znika i wraca) oraz merge do brancha, którego zespół nie skanuje.
   Uwzględnij, że dedup szuka tylko otwartych zgłoszeń. Sprawdź też, co dzieje się ze zgłoszeniem
   zamkniętym przez człowieka („Won't fix” / „Accepted risk”), gdy znalezisko nadal jest w skanie.
7. **Widoczność na boardzie:** karty, sprinty, velocity, metryki zespołów (istotne przy subtaskach
   w D).
8. **Artefakty:** gdzie trzymać `remediation.json` / `.md` przy braku dostępu pipeline'u do
   Bitbucket i przy kumulujących się załącznikach. Oceń, czy artefakt ma dotyczyć całej aplikacji,
   czy komponentu, i co z tego agent Issue-2-PR realnie przeczyta.
9. **Wzrost w czasie:** co się dzieje po 2–4 latach nocnych skanów, w tym przez to, że
    `identityHash` zawiera wersję (po bumpie kolejne CVE tworzą nowy byt). Dla C per cykl oceń
    rozmiar pojedynczego zgłoszenia cyklu względem limitu 32 767 znaków.
10. **Koszt migracji** z wariantu 1 (per komponent, produkcja) do C, D i E, z uwzględnieniem
    ograniczeń Jira REST API v2 (np. brak bulk edit, brak konwersji zadania w subtask przez API).

## Czego oczekuję

1. **Tabela porównawcza C / D / E** względem problemów 1–10. Oznacz, które kryteria przesądzają
   sprawę, i powiedz to wprost.
2. **Rekomendacja** z jawnym uzasadnieniem kompromisów i listą założeń do zweryfikowania przed
   wdrożeniem.
3. **Kontrakt sync ↔ Issue-2-PR:** które sekcje i pola zgłoszenia zapisuje sync, które agent,
   czego żaden z nich nie dotyka (np. assignee). Dołącz przykładowy opis zgłoszenia z sekcją TASKS
   dla rekomendowanego wariantu.
4. **Szkic instrukcji dla agenta Issue-2-PR** potrzebnych w rekomendowanym wariancie, z oceną,
   które z nich są krytyczne i co jeśli nie zadziałają.
5. **Diagramy Mermaid:** co najmniej przepływ end-to-end (od `src.zip` do zamknięcia zgłoszenia),
   diagram stanów zgłoszenia (i punktu TASKS, jeśli ma własny cykl) oraz sekwencję dialogu
   agent ↔ człowiek w komentarzach.

Nie zakładaj z góry, że któryś wariant jest właściwy. Jeśli potrzebujesz założeń, których nie ma
w opisie, wypisz je jawnie zamiast zgadywać.
