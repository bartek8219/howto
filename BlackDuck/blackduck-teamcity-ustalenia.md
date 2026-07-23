# Black Duck + AI Triage + Jira — ustalenia architektoniczne

**Cel dokumentu:** przeniesienie ustaleń na maszynę roboczą i kontynuacja prac nad szablonem TeamCity dystrybuowanym w skali organizacji.
**Data:** 2026-07-23
**Status:** narzędzie 3-etapowe działa jako Build Configuration; następny krok = wydzielenie szablonu.

---

## 1. Punkt wyjścia

Istniejący proces w TeamCity — jedna Build Configuration, trzy kroki oparte o obrazy dockerowe:

| # | Krok | Opis |
|---|------|------|
| 1 | **Black Duck Detect** | `detect.jar` skanuje podatności w archiwum `src.zip`; kod musi być po `build` + `restore`, żeby zależności były rozwiązane |
| 2 | **Agent remediacji** | Agent SDK → LiteLLM; LLM ocenia zasadność podatności i weryfikuje ją względem kodu źródłowego |
| 3 | **IssuePublisher** | Deterministyczny tool; na podstawie raportu BD + werdyktu LLM rejestruje issue w Jira |

**Cel docelowy:** to samo jako gotowe narzędzie do samoobsługowego wpięcia przez dowolny zespół w organizacji, niezależnie od stacku (.NET, Java, ...).

---

## 2. Decyzja centralna: logika w obrazach, nie w szablonie

> **Szablon TeamCity jest adapterem, nie produktem.**

Szablon = cienka skorupa: trzy `docker run` + parametry + agent requirements.
Cała logika, prompty, mapowanie na Jirę, obsługa błędów = w obrazach dockerowych.

### Uzasadnienie

1. **Wersjonowanie.** Szablon TeamCity nie ma wersji — zmiana propaguje się natychmiast do wszystkich zespołów, bez możliwości „zespół X zostaje na v1.2". Obrazy mają semver i tagi, więc zespół może kontrolować moment aktualizacji.
2. **Przenośność.** Obraz uruchomisz z TeamCity, Jenkinsa, GitLab CI i GitHub Actions. Szablon TeamCity (także w Kotlin DSL) działa **wyłącznie** w TeamCity — Kotlin DSL to format konfiguracji TeamCity, nie standard branżowy.
3. **Testowalność.** Obraz da się odpalić lokalnie bez serwera CI.

### Wersje jako parametry szablonu

```
%security.scanner.version%       default: 1.4
%security.remediation.version%   default: 1.4
%security.publisher.version%     default: 1.4
```

Zespół nadpisuje u siebie, jeśli nie chce jeszcze nowej wersji.

---

## 3. Kontrakt obrazów (KRYTYCZNE dla przenośności)

Wejście/wyjście definiowane **wyłącznie** przez ENV, wolumeny i kod wyjścia. Zero założeń o środowisku CI.

**Zabronione w obrazach:**
- twarde ścieżki TeamCity (`%teamcity.build.checkoutDir%` itp.)
- składnia `%param%`
- jakiekolwiek zależności od zmiennych specyficznych dla TeamCity

**Kontrakt:**

| Element | Konwencja |
|---------|-----------|
| Wejście | `/work/input` (rozpakowane / archiwum) |
| Wyjście | `/work/output/report.json`, BDIO, artefakty pośrednie |
| Konfiguracja | ENV: `BD_PROJECT_NAME`, `BD_URL`, `LITELLM_KEY`, `LITELLM_MODEL`, `JIRA_URL`, `JIRA_TOKEN`, `JIRA_PROJECT` |
| Sygnalizacja błędu | kod wyjścia (nie parsowanie logów) |

**Fail fast:** obraz waliduje obecność i ważność kluczy **na starcie**, nie po dwóch minutach skanu.

---

## 4. Problem `src.zip` — odwrócenie odpowiedzialności

Nie budujemy cudzych aplikacji. Definiujemy **kontrakt wejścia**, nie implementację buildu.

```
Payments.Build                (własność zespołu)
   └─ produkuje artefakt: scan-input.zip
        ↑
Payments.SecurityScan         (z szablonu centralnego)
   ├─ snapshot dependency  → Payments.Build   (ta sama rewizja kodu)
   └─ artifact dependency  → scan-input.zip
```

Zespół dopisuje u siebie jeden krok „spakuj po restore". Do dostarczenia: krótki dokument z zawartością archiwum per stack:
- **.NET** — `obj/project.assets.json`, katalog `packages/`
- **Java** — `~/.m2` lub `target/`
- nazewnictwo artefaktu — ustalone i stałe

### ⚠️ Zastrzeżenie dot. jakości skanu Black Duck

Detect działa najlepiej **w środowisku builda** — jego detektory wołają natywne menedżery pakietów (`dotnet list package`, `mvn dependency:tree`) i dostają prawdziwy graf zależności tranzytywnych.

Uruchomiony na rozpakowanym zipie w obcym kontenerze degraduje się w dużej mierze do **signature scan** (dopasowywanie hashy plików) — bardziej szumnie, gorzej z wykrywaniem wersji.

**Rekomendowany podział na dwie warstwy:**

| Warstwa | Gdzie działa | Uwagi |
|---------|--------------|-------|
| **Krok 1 (Detect)** | w buildzie zespołu, tuż po restore, w jego checkout dir | jako osobny szablon lub meta-runner; publikuje raport + BDIO jako artefakt |
| **Kroki 2–3 (LLM + Jira)** | centralna konfiguracja | stack-agnostyczne; ciągną raport i źródła przez artifact dependency |

Więcej pracy wdrożeniowej, ale istotnie lepsza jakość wyników. Do decyzji przed budową szablonu.

---

## 5. Dystrybucja szablonu w TeamCity

Szablon zdefiniowany w projekcie **`<Root>`** jest widoczny we wszystkich projektach instancji.
Zespół: nowa Build Configuration → **Actions → Attach to template** → uzupełnia parametry.

### Ograniczenia do zaakceptowania / obejścia

| Ograniczenie | Obejście |
|--------------|----------|
| Brak wersjonowania szablonu | wersjonowanie obrazów (pkt 2) |
| Edycja wymaga uprawnień do Root (zwykle tylko admini TC) | zaplanować proces zmian / ownership |
| Nie da się usunąć odziedziczonego kroku (tylko disable) | warianty przez parametr warunkowy lub drugi szablon |

### Przypomnienie mechaniki TeamCity

- Szablon podpina się **per Build Configuration** (Edit Configuration Settings → General Settings → *Based on template*), a nie na poziomie projektu. `Default template` na projekcie to opcjonalne, osobne ustawienie.
- Podpięcie jest odwracalne: **Actions → Attach to template / Detach from template**. Od 2017.2 konfiguracja może mieć kilka szablonów naraz.
- Kroki z szablonu mają w UI dopisek `(inherited)`; to samo dotyczy parametrów, triggerów i wymagań agenta.
- Zależności: Edit Configuration Settings → **Dependencies** (obie sekcje na jednej stronie). Podgląd łańcucha: poziom projektu → zakładka **Build Chains**.

---

## 6. Kotlin DSL / Versioned Settings

Szablon jako kod w repo: katalog `.teamcity/` + `settings.kts`. Włączenie: **Project → Versioned Settings** (VCS root + format Kotlin). TeamCity generuje startowy kod z obecnej konfiguracji.

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

**Tryby synchronizacji:** two-way (zmiany w UI wracają commitem) / one-way (repo = jedyne źródło prawdy, UI read-only).
Dla współdzielonego narzędzia → **one-way**, inaczej ktoś „szybko poprawi" coś w UI.

### ⚠️ Haczyk do przemyślenia

Versioned settings włączone na projekcie obejmują **całe poddrzewo**. Szablon musi siedzieć w Root, żeby był widoczny globalnie → włączenie DSL na Root oznacza, że jedno repo opisuje konfigurację całego serwera. Podprojekt może nadpisać ustawienia własnym repo, więc da się to zdekomponować, ale trzeba to zaplanować.

**Ścieżka na start:** szablon w UI, DSL tylko dla obrazów. Mniej elegancko, odblokowuje pilota.

---

## 7. Sekrety i parametry

### LiteLLM — model kluczy

| Poziom | Co |
|--------|-----|
| Upstream key do Anthropic | jeden, zna go **wyłącznie** instancja LiteLLM; nigdy nie trafia do TeamCity |
| Virtual key per zespół | generowany przez LiteLLM: własny `max_budget` (reset miesięczny), rate limit TPM/RPM, lista dozwolonych modeli, tag rozliczeniowy |

Zespół trzyma swój virtual key jako parametr typu `password` **w swoim projekcie**, nie w Root.
Efekt: zespół wysycający budżet dostaje 429 tylko dla siebie. Centralny podgląd spendu per zespół zachowany; limit podnosisz bez rozdawania nowych sekretów.

### Jira

Token **per zespół** — nie z powodu limitów, tylko uprawnień i atrybucji. Issue ma trafić do projektu zespołu, założone przez ich service account. Centralny token = konto z dostępem do wszystkich projektów Jiry w firmie (security tego nie przepuści).

### Pozostałe parametry

- Parametry wymagane od zespołu (`bd.project.name`, `jira.project`, tokeny): pusta wartość + `display = PROMPT`. TeamCity oznaczy konfigurację jako niegotową do uruchomienia — darmowy walidator wdrożenia.
- **Agent requirement na Dockera** — inaczej ktoś dostanie build na agencie bez runtime'u i zgłosi to jako bug narzędzia.

---

## 8. Ryzyka operacyjne

| Ryzyko | Status / mitygacja |
|--------|--------------------|
| Spam w Jirze | ✅ **deduplikacja już ogarnięta** |
| Koszt i czas (Detect + LLM to minuty i realne pieniądze) | nie wieszać na każdym pushu — schedule trigger nocny albo tylko `main` |
| Audytowalność werdyktów LLM | zapisywać obok werdyktu: wersję obrazu, model, hash promptu — do odtworzenia za pół roku |
| **Jakość triage'u LLM** | ⚠️ **główne ryzyko adopcji** — przed rolloutem przepuścić 2–3 repa o znanym wyniku i policzyć, ile realnych podatności odrzucono jako FP. Jeden przeoczony CVE kosztuje więcej zaufania niż setka nadmiarowych ticketów |

---

## 9. Portowalność na inne CI

Warunek: dyscyplina z pkt 3 (komunikacja wyłącznie przez ENV, wolumeny, kod wyjścia).

### GitLab CI

```yaml
security-scan:
  image: registry.firma/security/bd-detect:1.4
  script: [ "/entrypoint.sh --input=scan-input.zip" ]
```

### Jenkins

```groovy
stage('BD Detect') {
  agent { docker { image "registry.firma/security/bd-detect:${SCANNER_VERSION}" } }
  environment { LITELLM_KEY = credentials('litellm-team-payments') }
  steps { sh '/entrypoint.sh --input=/work/input' }
}
```

Odpowiednik szablonu = **Shared Library** (repo Groovy, rejestrowane globalnie, wołane jako `securityScan(bdProject: 'payments')`). Model jest **lepszy niż w TeamCity**, bo biblioteka jest wersjonowana tagiem: `@Library('security@v2')`.

**Co nie przenosi się 1:1:**

| TeamCity | Jenkins |
|----------|---------|
| Snapshot / artifact dependency | brak konceptu — kolejne `stage` w tym samym pipelinie (prostsze) albo `copyArtifacts` |
| Parametry `password` | Credentials Store + `withCredentials` / `environment` |
| `requirements { exists("docker.version") }` | labels na node'ach |

Szacunek: ~1 dzień pracy na zespół pilotażowy.

---

## 10. Następne kroki

1. **Decyzja architektoniczna:** jedna warstwa (wszystko centralnie, na `src.zip`) czy dwie warstwy (Detect u zespołu, LLM+Jira centralnie)? — patrz pkt 4. Blokuje kształt szablonu.
2. Ustalić i zapisać **kontrakt ENV/wolumenów** dla trzech obrazów; usunąć z nich wszelkie zależności od TeamCity.
3. Otagować obrazy semver, opublikować w rejestrze firmowym, dodać changelog.
4. Zbudować szablon w Root (na start: UI), sparametryzować wersje obrazów.
5. Skonfigurować LiteLLM: virtual keys per zespół + budżety.
6. Napisać dokument dla zespołów: zawartość i nazewnictwo `scan-input.zip` per stack (.NET / Java).
7. Pilot na 1 zespole → pomiar jakości triage'u (pkt 8) → dopiero potem rollout.
8. Do rozważenia później: przeniesienie szablonu do Kotlin DSL (pkt 6, zaplanować zakres versioned settings).
