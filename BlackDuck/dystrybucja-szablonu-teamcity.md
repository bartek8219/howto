# Dystrybucja szablonu skanowania bezpieczeństwa — TeamCity

## Kontekst

Narzędzie do wykrywania podatności w bibliotekach oparte na raporcie **Black Duck RAPID**, w którym remediacje przygotowuje agent AI. Logika zamknięta w trzech obrazach Docker uruchamianych jako kolejne stepy jednego procesu:

`BD Scan` → `Agent Remediation` → `Jira Publisher`

Całość zdefiniowana jako szablon (build configuration template) w TeamCity, z parametrami: adres LiteLLM, adres Black Duck, adres Jira oraz tokeny autoryzacyjne.

**Stan obecny:** szablon udostępniony na poziomie projektu zespołu.
**Cel:** udostępnienie rozwiązania całej organizacji.

---

## Ustalenia wstępne

| Ustalenie | Konsekwencja |
|---|---|
| Hostowanie szablonu to temat dla **adminów TeamCity**, nie Jiry | Dwie osobne rozmowy — TeamCity (dystrybucja) i Jira (publikacja ticketów) |
| Szablon umieszczony w **Root Project** jest widoczny dla wszystkich projektów w dół drzewa | Podstawowy mechanizm dystrybucji org-wide |
| **Versioned Settings wyłączone globalnie od 2022** (auto-disable po upgrade serwera, nikt nie przywrócił) | Ogranicza opcje oparte na konfiguracji-jako-kodzie |
| Brak uprawnienia `Enable/disable versioned settings` po stronie zespołu | Uprawnienie należy domyślnie do roli System Administrator |
| Versioned Settings ≠ integracja z Git | VCS Roots (pobieranie kodu do buildów) działają normalnie i nie są tym objęte |
| Prawdziwa logika siedzi w obrazach Docker, szablon to cienki adapter | Wersjonowanie realizowane tagiem obrazu, nie zmianą szablonu |

### Opcje odrzucone

- **Kotlin DSL jako biblioteka Maven/Artifactory** — wymaga włączonych Versioned Settings u *każdego* korzystającego zespołu, nie tylko u nas. Dodatkowo daje tę samą korzyść (pinowanie wersji), którą osiągamy prościej tagiem obrazu Docker.
- **Scan-as-a-service** (centralna konfiguracja wołana przez inne zespoły) — wymaga dostępu do cudzych VCS rootów, obciąża nasze agenty, słaba atrybucja buildów.

---

## Porównanie opcji A i B

| Kryterium | A — szablon w Root Project | B — Kotlin DSL w repo + Versioned Settings |
|---|---|---|
| **Mechanizm** | Szablon zdefiniowany w `_Root`, zespoły robią „Associate with template" na swoim buildzie | Szablon jako `.teamcity/*.kt` w repo, podpięty jako versioned settings projektu |
| **Wykonalność dzisiaj** | Możliwe od ręki | Zablokowane — wymaga włączenia Versioned Settings przez System Admina |
| **Kto może aktualizować** | Tylko admin Root Project (de facto admin serwera) | Właściciel repo, przez PR |
| **Review zmian** | Brak | PR, historia zmian, CODEOWNERS |
| **Propagacja zmiany** | Natychmiastowa do wszystkich podpiętych buildów | Natychmiastowa po merge |
| **Blast radius** | Wysoki — błędna zmiana psuje buildy wszystkich zespołów | Wysoki, ale z bramką review przed merge |
| **Wersjonowanie** | Przez tag obrazu Docker (parametr z defaultem) | Przez tag obrazu Docker + historia Git konfiguracji |
| **Wpływ na innych** | Żaden — Root zyskuje jeden dodatkowy szablon | Włączenie synchronizacji dla projektu włącza ją dla wszystkich podprojektów; możliwy tryb read-only w UI |
| **Narzut wdrożenia** | Minimalny | Wysoki — decyzja na poziomie serwera po 4 latach przerwy |

### Rekomendacja

**Opcja A**, z zaprojektowaniem szablonu tak, by pozostał cienki:

1. Szablon zawiera wyłącznie trzy stepy Docker i listę parametrów — cała logika pozostaje w obrazach.
2. Tag obrazu jako parametr z defaultem, np. `%security.scan.image.tag% = 1.4.0`. **Nigdy `latest`.** Aktualizacja = podbicie defaultu w jednym miejscu; zespół może nadpisać u siebie i zostać na starszej wersji.
3. Parametry per-zespół (klucz LiteLLM, projekt Jira, próg severity) z pustym defaultem — build ma się jawnie wywalić z czytelnym komunikatem, zamiast po cichu wysłać ticket w złe miejsce.
4. Ustalona z adminami ścieżka aktualizacji szablonu (my zgłaszamy zmianę, oni aplikują) — patrz pytania poniżej.

**Opcja B** pozostaje w zapasie na wypadek, gdyby okazało się, że Versioned Settings da się włączyć dla pojedynczego projektu bez ruszania reszty serwera.

---

## Pytania do adminów TeamCity

### Hosting i uprawnienia
- Czy możecie umieścić szablon w Root Project i **kto będzie mógł go później edytować**?
- Czy da się nadać rolę pozwalającą edytować ten jeden szablon bez pełnych praw do Root? Jeśli nie — jaka jest uzgodniona ścieżka aktualizacji (ticket, PR, okno serwisowe)?
- Uwaga: **nie** ustawiać szablonu jako `default template` na Root — podpięłoby go do wszystkich buildów w firmie.
- Czy zespół dziedziczący szablon może wyłączyć lub nadpisać step? Czy z punktu widzenia compliance chcemy, żeby mógł?

### Versioned Settings
- Czy wyłączenie od 2022 to świadoma decyzja, czy tylko nikt nie wrócił do tematu?
- Czy da się włączyć Versioned Settings dla **jednego** projektu, nie ruszając reszty serwera?

### Infrastruktura
- Czy wszystkie pule agentów mają **Dockera** oraz dostęp sieciowy do Black Duck, LiteLLM i Jiry? *(Najczęstsza cicha przeszkoda — szablon działa u nas, wywala się u połowy zespołów na firewallu.)*
- Jak przechowywać **sekrety**? Współdzielony secure token storage czy password parameters per zespół? Kto może odczytać wartości?
- Czy masowa adopcja nie uderzy w limity licencyjne lub pojemność agentów?

---

## Pytania do adminów Jira

### Dostęp i uprawnienia
- **Konto techniczne**: jeden service account dla wszystkich projektów czy token per zespół? Kto jest właścicielem, jak wygląda rotacja?
- **Permission scheme**: czy konto można dodać do globalnej grupy z uprawnieniem *Create Issues*, czy trzeba to robić projekt po projekcie?
- Jira Cloud czy Data Center? *(determinuje typ auth: API token + email vs PAT)*

### Model docelowy
- Jeden centralny projekt bezpieczeństwa z komponentami per zespół, czy tickety w projektach zespołów?
- **Pola wymagane i screeny**: które projekty mają nietypowe obowiązkowe pola custom? Potrzebny minimalny kontrakt: typ issue, summary, description, labels.

### Skala i higiena
- **Deduplikacja**: jaki mechanizm akceptujecie — label typu `bd-CVE-2024-XXXX-<komponent>` + JQL przed create, czy dedykowane pole?
- **Rate limits**: jaki jest limit API? Co się stanie, gdy pierwszy skan monorepo wygeneruje 300 issues naraz?
- **Notyfikacje i automatyzacje**: czy masowe tworzenie ticketów nie odpali reguł automation ani nie zaleje skrzynek?

### Compliance
- Czy treść generowana przez LLM może trafiać do Jiry i czy wymaga oznaczenia (label `ai-generated`, disclaimer w opisie)?

---

## Następny krok

Przed obiema rozmowami warto mieć **działający pilot z drugim zespołem**. Rozmowa „chcę to udostępnić" idzie znacznie gorzej niż „dwa zespoły już tego używają, chcę to sformalizować".
