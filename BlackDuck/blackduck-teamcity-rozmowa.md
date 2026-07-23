# Zapis rozmowy — TeamCity, Black Duck, AI triage, Jira

**Data:** 2026-07-23

---

## Pytanie 1 — Jak działa TeamCity: Build Configurations, Templates, Subprojects

**Odpowiedź:**

W TeamCity wszystko kręci się wokół czterech poziomów:

**Project** — to tylko kontener/folder. Sam nic nie buduje. Trzyma w sobie konfiguracje, szablony, VCS rooty, parametry i connections (np. do Dockera, NuGeta).

**Build Configuration** — właściwy „pipeline". Odpowiednik `azure-pipelines.yml` albo workflow w GitHub Actions. Zawiera: kroki (build steps), triggery, VCS root, parametry, zależności, wymagania wobec agenta. To jedyna rzecz, którą faktycznie uruchamiasz — każde uruchomienie to jeden *Build* z numerem.

**Build Configuration Template** — szablon, z którego dziedziczą konfiguracje. Analogia: klasa abstrakcyjna. Sam się nie uruchamia. Konfiguracja oparta o szablon przejmuje wszystkie kroki i parametry, może dopisać własne i nadpisać parametry, ale **nie może usunąć** odziedziczonego kroku (najwyżej go wyłączyć). Zmiana w szablonie propaguje się natychmiast do wszystkich potomków — to główny powód, dla którego się tego używa: 30 mikroserwisów .NET, jeden szablon „build + test + push do NuGeta".

**Subprojects** — projekty zagnieżdżone. Służą do grupowania i, co ważniejsze, do **dziedziczenia**: parametry, VCS rooty, szablony i connections zdefiniowane w rodzicu są automatycznie widoczne we wszystkich podprojektach.

```
<Root>
└── Płatności                  ← Project
    ├── Templates              ← szablon "dotnet-build"
    ├── Payments.API           ← Build Configuration (z szablonu)
    ├── Payments.Worker        ← Build Configuration (z szablonu)
    └── Płatności.Deploy       ← Subproject
        ├── Deploy to DEV
        └── Deploy to PROD
```

Sam CI/CD składa się z **wielu konfiguracji połączonych zależnościami**:

- **Snapshot dependency** — „nie startuj, dopóki B nie skończy, i użyj dokładnie tej samej rewizji z repo". To buduje *build chain*.
- **Artifact dependency** — „pobierz artefakty z buildu B" (np. opublikowane binarki do deploya).

Typowo: `Build` → `Tests` → `Docker Push` → `Deploy DEV` → (ręczny trigger) `Deploy PROD`. Każdy krok to osobna Build Configuration.

Dwie pułapki na start: szablon nadpisuje się tylko „w górę" (dodajesz, nie odejmujesz) — jeśli potrzebujesz wariantu bez jakiegoś kroku, robi się drugi szablon albo warunkowe wykonanie przez parametr. I druga: kolejność w łańcuchu definiuje się w konfiguracji *końcowej* (to ona deklaruje, od czego zależy), nie w początkowej.

---

## Pytanie 2 — VCS rooty, `(inherited)`, gdzie wskazuje się szablon, gdzie są dependencies

### VCS root

Zapisana definicja połączenia z repozytorium: URL, typ (Git/SVN), branch domyślny, branch specification, dane uwierzytelniające, checkout rules, interwał odpytywania o zmiany. Jeden VCS root = jedno repo (plus reguły gałęzi).

Na kilku poziomach widać go z dwóch powodów:
- **Dziedziczenie w dół** — VCS root z projektu nadrzędnego jest automatycznie dostępny we wszystkich podprojektach i ich konfiguracjach. Stąd rozdział na „own" i te z rodzica.
- **Poziom definicji ≠ poziom użycia** — root definiuje się raz jak najwyżej (żeby nie duplikować credentiali), a podpina w konkretnych Build Configurations. Jeśli zdefiniowany bezpośrednio w podprojekcie — jest używany tylko tam.

Zasada: definiuj na najwyższym poziomie, na którym jest potrzebny więcej niż raz.

### `(inherited)`

Tak — krok pochodzi z szablonu, do którego konfiguracja jest podpięta. Można go **wyłączyć** (disable) i nadpisać część ustawień, ale nie usunąć ani nie przesunąć przed kroki własne (odziedziczone idą pierwsze, chyba że szablon na to pozwala przez parametry).

To samo oznaczenie pojawia się przy parametrach, triggerach i wymaganiach agenta pochodzących z szablonu.

### Gdzie wskazany jest szablon

Dwie różne rzeczy:

- **Default template** (Project → General Settings) — „każda konfiguracja w tym projekcie i podprojektach automatycznie dziedziczy ten szablon". Wygodne, rzadziej używane.
- **Podpięcie per konfiguracja** — Build Configuration → **Edit Configuration Settings** → **General Settings**. Na górze linia *Based on template: `nazwa`*. To źródło `(inherited)`.

Nie jest to decyzja jednorazowa. Prawy górny róg ekranu edycji → menu **Actions**:
- **Attach to template…** — podepnij (od TeamCity 2017.2 konfiguracja może mieć **kilka** szablonów naraz, stosowanych po kolei)
- **Detach from template** — odepnij; TeamCity zapyta, czy skopiować ustawienia z szablonu do konfiguracji (zwykle chcesz „tak", inaczej stracisz kroki)

### Dependencies

Build Configuration → **Edit Configuration Settings** → lewe menu → **Dependencies**. Na jednej stronie sekcje *Snapshot Dependencies* i *Artifact Dependencies*, obie z „Add new dependency".

Dodatkowo:
- Artifact dependency może być też zdefiniowana jako osobny **build step** typu „Artifact dependency" — jeśli nie ma jej w Dependencies, warto zerknąć w Build Steps.
- Cały łańcuch graficznie: poziom projektu → zakładka **Build Chains**.

---

## Pytanie 3 — Czy 3-etapowy proces (Black Duck + AI triage + Jira) da się przełożyć na szablon dystrybuowany w organizacji

**Kontekst od użytkownika:** Build Configuration z trzema krokami opartymi o obrazy dockerowe — (1) Black Duck Detect na `src.zip` po build+restore, (2) agent remediacji przez Agent SDK + LiteLLM, (3) deterministyczny IssuePublisher do Jiry. Pytanie o dystrybucję do innych zespołów, globalną widoczność szablonu oraz problem dostarczenia `src.zip` przy różnych stackach (.NET, Java).

**Odpowiedź:**

Tak, wykonalne i architektonicznie sensowne — ale sam szablon TeamCity to zła granica dla logiki.

### Dystrybucja szablonu

Szablon w projekcie `<Root>` jest widoczny we wszystkich projektach instancji. To standardowy wzorzec dla „narzędzi korporacyjnych". Zespół robi nową Build Configuration → **Attach to template** → wypełnia parametry.

Ograniczenia:
- **Brak wersjonowania.** Zmiana propaguje się natychmiast do wszystkich zespołów. Nie ma „zespół X zostaje na v1.2".
- **Kto może edytować** — uprawnienia do projektu Root, czyli zwykle tylko administratorzy TeamCity.
- **Nie da się usunąć odziedziczonego kroku** — wariant wymaga osobnego szablonu albo warunku na parametrze.

### Wersjonuj obrazy, nie szablon

Szablon powinien być **cienką skorupą**: 3 kroki `docker run`, każdy z tagiem obrazu z parametru:

```
%security.scanner.version%   default: 1.4
%security.remediation.version%
%security.publisher.version%
```

Logika, prompty, mapowanie na Jirę — w obrazach, wydawanych z semver i changelogiem. Zespół, który nie chce jeszcze v2, nadpisuje parametr u siebie. Dodatkowy zysk: to samo narzędzie odpalisz z GitLab CI czy GitHub Actions. Szablon przestaje być produktem, staje się adapterem.

Sam szablon warto trzymać w **Kotlin DSL / versioned settings** — zmiany idą przez PR i review, nie przez klikanie w UI produkcyjnym.

### Problem src.zip — odwróć odpowiedzialność

Nie musisz budować cudzych aplikacji. **Zdefiniuj kontrakt, nie implementację.** Szablon wymaga na wejściu artefaktu o ustalonej nazwie i strukturze. Zespół odpowiada za jego wyprodukowanie w *swoim* buildzie — tylko on wie, czy to `dotnet restore`, `mvn dependency:go-offline`, czy `npm ci`.

```
Payments.Build          (własność zespołu)
   └─ produkuje artefakt: scan-input.zip
        ↑
Payments.SecurityScan   (z szablonu)
   ├─ snapshot dependency  → Payments.Build
   └─ artifact dependency  → scan-input.zip
```

Snapshot dependency gwarantuje tę samą rewizję kodu, artifact dependency dociąga paczkę. Zespół dopisuje jeden krok „spakuj po restore". Do tego krótki dokument: co ma być w archiwum dla .NET (`obj/project.assets.json`, `packages/`), co dla Javy (`~/.m2` albo `target/`), i jak ma się nazywać.

**Ważna uwaga o Black Ducku.** Detect działa najlepiej uruchamiany *w środowisku builda*, bo detektory wołają natywne menedżery pakietów (`dotnet list package`, `mvn dependency:tree`) i dostają prawdziwy graf zależności tranzytywnych. Odpalony na rozpakowanym zipie w obcym kontenerze degraduje się w dużej mierze do **signature scan** — dopasowywania hashy plików. Działa, ale jest bardziej szumny i gorzej wykrywa wersje.

Stąd propozycja podziału na dwie warstwy:
- **Krok 1 (Detect)** — jako szablon lub meta-runner podpinany do *builda zespołu*, tuż po restore, w jego checkout dir. Publikuje raport + BDIO jako artefakt.
- **Kroki 2–3 (LLM + Jira)** — stack-agnostyczne, w centralnej konfiguracji ciągnącej raport i źródła przez artifact dependency.

Więcej pracy przy wdrożeniu, ale duża różnica w jakości skanu, przy zachowaniu zasady „nie budujemy cudzego kodu".

### Sekrety i parametry

- Klucz do LiteLLM — centralnie w Root jako parametr `password`; zespoły dziedziczą i nie widzą wartości. *(uwaga: skorygowane w dalszej części rozmowy — patrz pytanie 4, pkt 3)*
- Token Jira i nazwa projektu BD — parametry z pustą wartością i `prompt` w spec. TeamCity oznaczy konfigurację jako niegotową do uruchomienia — darmowy walidator wdrożenia.
- **Agent requirement na Dockera** — inaczej ktoś dostanie build na agencie bez runtime'u i zgłosi to jako bug narzędzia.

### Co ugryzie w praktyce

- **Idempotencja Jiry.** Bez deduplikacji (klucz: `BD vulnerability ID + komponent + wersja + repo`) backlog zostanie zaspamowany przy każdym buildzie. Publisher musi sprawdzić, czy issue istnieje, i ewentualnie tylko zaktualizować komentarz. Największe ryzyko wizerunkowe dla adopcji.
- **Trigger.** Detect + analiza LLM to minuty i realne pieniądze. Nie na każdym pushu — schedule nocny albo tylko `main`.
- **Wyniki LLM w raporcie.** Obok werdyktu zapisywać wersję obrazu, model i hash promptu — do odtworzenia za pół roku.

---

## Pytanie 4 — Przenośność, Kotlin DSL, klucze per zespół, deduplikacja

### 4.1 Czy szablon TeamCity jest przenośny na GitLab / GitHub / Jenkins?

**Nie.** Przenośne są **obrazy dockerowe**. Szablon TeamCity (także w Kotlin DSL) działa tylko w TeamCity — Kotlin DSL to format konfiguracji TeamCity, nie standard branżowy.

Jeśli cała logika siedzi w obrazach, a szablon to trzy `docker run` z parametrami, to zespół na GitLabie pisze u siebie kilkanaście linijek YAML-a wołających te same obrazy:

```yaml
security-scan:
  image: registry.firma/security/bd-detect:1.4
  script: [ "/entrypoint.sh --input=scan-input.zip" ]
```

Utrzymujesz jeden produkt (obrazy) i N cienkich adapterów per platforma CI. Gdyby logika była w szablonie, migracja na GitLaba oznaczałaby przepisanie narzędzia od zera.

Konsekwencja praktyczna: **kontrakt wejścia/wyjścia obrazów definiuj przez ENV i pliki**, nie przez parametry TeamCity. Czyli `BD_PROJECT_NAME`, `LITELLM_KEY`, wejście w `/work/input`, wyjście w `/work/output/report.json`. Wtedy adapter to naprawdę mapowanie zmiennych.

### 4.2 Jak wygląda budowanie szablonu w Kotlin DSL?

Tak — to kod w repo, podpinany do TeamCity.

Struktura: katalog `.teamcity/`, w nim `settings.kts` plus pliki z definicjami. Włączenie: **Project → Versioned Settings** (VCS root + format Kotlin). TeamCity generuje startowy kod z obecnej konfiguracji — nie pisze się od zera.

```kotlin
object SecurityScanTemplate : Template({
    name = "Security Scan (Black Duck + AI triage)"

    params {
        param("scanner.version", "1.4")
        password("litellm.key", "", display = ParameterDisplay.PROMPT)
        param("bd.project.name", "", display = ParameterDisplay.PROMPT)
    }

    steps {
        script {
            name = "Black Duck Detect"
            scriptContent = "detect.sh"
            dockerImage = "registry.firma/security/bd-detect:%scanner.version%"
        }
        // ...
    }

    requirements { exists("docker.version") }
})
```

Dwa tryby synchronizacji: **two-way** (zmiany w UI wracają commitem do repo) albo **one-way** (repo = jedyne źródło prawdy, UI read-only). Dla współdzielonego narzędzia — one-way, inaczej ktoś „szybko poprawi" coś w UI.

Haczyk: versioned settings włączone na projekcie obejmują **całe poddrzewo**. Skoro szablon musi siedzieć w Root, żeby był widoczny dla wszystkich, włączenie DSL na Root oznacza, że jedno repo opisuje konfigurację całego serwera. Dla wielu organizacji to zaleta (config-as-code), ale to decyzja o rząd wielkości większa niż „chcę wersjonować jeden szablon". Podprojekt może nadpisać ustawienia własnym repo, więc da się to zdekomponować — trzeba zaplanować.

Alternatywa na start: szablon w UI, DSL tylko dla obrazów. Mniej elegancko, ale odblokowuje pilota.

### 4.3 Klucze per zespół (korekta wcześniejszej rekomendacji)

Rozdzielić dwie rzeczy:
- **Klucz upstream do Anthropic** — jeden, zna go tylko instancja LiteLLM, nigdy nie trafia do TeamCity.
- **Virtual key per zespół** — LiteLLM generuje klucze z własnym budżetem (`max_budget`, reset miesięczny), rate limitem (TPM/RPM), listą dozwolonych modeli i tagiem do rozliczeń. Zespół trzyma swój jako parametr `password` **w swoim projekcie**, nie w Root.

Efekt: zespół wysycający budżet dostaje 429 tylko dla siebie, reszta jedzie dalej. Centralny podgląd spendu per zespół zachowany; limit podnosisz bez rozdawania nowych sekretów.

Token Jiry per zespół — tak, i to nawet nie z powodu limitów, tylko uprawnień i atrybucji. Issue ma trafić do projektu zespołu, założone przez ich service account. Centralny token oznaczałby konto z dostępem do wszystkich projektów Jiry w firmie, czego security raczej nie przepuści.

Zabezpieczenie wdrożeniowe: obraz powinien walić błędem od razu na starcie, jeśli klucza brak lub jest nieważny — nie po dwóch minutach skanu.

### 4.4 Deduplikacja

Ogarnięta po stronie użytkownika. Największym pozostałym ryzykiem adopcji jest **jakość triage'u LLM** — przed szerokim rolloutem warto przepuścić narzędzie przez 2–3 repa o znanym wyniku i policzyć, ile realnych podatności zostało odrzuconych jako false positive. Jeden przeoczony CVE w pilocie kosztuje więcej zaufania niż setka nadmiarowych ticketów.

---

## Pytanie 5 — Czy da się to później łatwo przenieść na Jenkinsa?

Tak — pod warunkiem trzymania się podziału „logika w obrazach, CI tylko woła obrazy". Wtedy port to napisanie adaptera, nie narzędzia.

Adapter w Jenkinsie to `Jenkinsfile`:

```groovy
stage('BD Detect') {
  agent { docker { image "registry.firma/security/bd-detect:${SCANNER_VERSION}" } }
  environment { LITELLM_KEY = credentials('litellm-team-payments') }
  steps { sh '/entrypoint.sh --input=/work/input' }
}
```

Odpowiednikiem szablonu TeamCity jest **Shared Library** — repo z kodem Groovy, rejestrowane globalnie w Jenkinsie, wołane przez zespoły jako `securityScan(bdProject: 'payments')`. Model jest nawet lepszy od TeamCity, bo biblioteka jest **wersjonowana tagiem** (`@Library('security@v2')`), czyli daje to, czego szablonowi TeamCity brakuje.

Co się nie przeniesie 1:1:
- **Snapshot/artifact dependency** — Jenkins nie ma tego konceptu. Najczęściej robi się z tego kolejne `stage` w tym samym pipelinie zespołu (prostsze niż łańcuch buildów), albo `copyArtifacts` z osobnego joba.
- **Sekrety** — parametry `password` → Credentials Store, wstrzykiwane przez `withCredentials`/`environment`. Mapowanie proste, ale przepisywane ręcznie.
- **Wymagania agenta** — `exists("docker.version")` → labels na node'ach.

Szacunkowo dzień pracy na zespół pilotażowy. Warunek: obrazy nie zakładają niczego o środowisku — brak twardych ścieżek TeamCity, brak `%param%`, komunikacja wyłącznie przez ENV, wolumeny i kod wyjścia. Przy tej dyscyplinie Jenkins, GitLab i GitHub Actions to tylko trzy różne sposoby napisania `docker run`.
