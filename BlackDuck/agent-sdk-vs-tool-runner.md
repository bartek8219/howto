# Agent SDK vs Client SDK (tool_runner) — wybór dla pipeline'u remediacji Black Duck

Kontekst: krok CI w TeamCity, skan Black Duck RAPID → agent LLM wykonujący remediację
podatności w zależnościach .NET. Uruchamiane na userze technicznym, w obrazie Docker.

## Porównanie opcji

| | Agent SDK | Client SDK + `tool_runner` | Client SDK + ręczna pętla |
|---|---|---|---|
| Narzędzia plikowe | gotowe, wtrenowane w model | piszesz sam | piszesz sam |
| Zarządzanie kontekstem | automatyczna kompakcja | Twoja sprawa | Twoja sprawa |
| Zawartość obrazu | natywna binarka CLI | czysty wheel | czysty wheel |
| Kontrola nad promptem | ograniczona | pełna | pełna |
| Prompt caching | zarządzany przez CLI | Ty ustawiasz `cache_control` | Ty ustawiasz |
| Testowalność | fake na poziomie adaptera | wstrzykiwalny klient | trywialna |
| Retry / 429 | w CLI | w kliencie (backoff + jitter) | w kliencie |
| Obserwowalność | parsowanie JSONL | obiekty Pythona | pełna |
| DB approval | binarka third-party w wheelu | jeden pakiet PyPI | jeden pakiet |

## Argument za Agent SDK w CI

Największa różnica nie jest w wygodzie, tylko w **edit success rate**. Pisząc własne
narzędzia przez `@beta_tool` definiujesz np. `edit_file(path, old, new)` — i model ma
dokładnie zero treningu na Twoim schemacie. Wbudowany `Edit` ma wtrenowaną semantykę
i tolerancję na whitespace. Przy remediacji, gdzie efektem ma być kompilujący się kod,
przekłada się to bezpośrednio na liczbę nieudanych buildów.

Poza tym:

- **Kompakcja kontekstu** — drzewo zależności .NET (`Directory.Packages.props`,
  transitive graph) szybko zapycha okno kontekstowe.
- **`setting_sources=["project"]`** — agent czyta `CLAUDE.md` i skills bezpośrednio
  z repozytorium, więc zespoły mogą wpływać na zachowanie remediacji bez zmiany obrazu.
- **Hooki `PreToolUse`** — twarde, deterministyczne blokowanie niebezpiecznych komend
  Basha, niezależnie od tego, co zdecyduje model.

## Argument za `tool_runner` w CI

**Determinizm i audytowalność.** Definiujesz dokładnie trzy narzędzia —
`read_file`, `bump_package_version`, `run_build` — i model fizycznie nie może zrobić nic
innego. Żadnego dowolnego Basha, żadnego `curl`. Dla kroku CI, który ma prawo modyfikować
kod produkcyjny, to jest mocny argument przed komisją bezpieczeństwa.

Poza tym:

- **Obraz bez zvendorowanej binarki** — jeden pakiet PyPI zamiast natywnego binarium
  third-party, którego skanery SCA nie rozłożą na komponenty.
- **Pełna kontrola nad układem promptu** pod prompt caching — wymierny wpływ na koszt
  przy powtarzalnych uruchomieniach.
- **Wstrzykiwalny klient w testach** — fake zamiast mockowania subprocessu.

## Uwagi

- `client.beta.messages.tool_runner()` pozostaje w wersji **beta** (namespace `beta`,
  dekoratory `@beta_tool` / `@beta_async_tool`). To helper na poziomie SDK, nie
  eksperymentalna usługa — pod spodem leci zwykłe `/v1/messages`. Ryzykiem jest zmiana
  sygnatury między wersjami pakietu `anthropic`, więc wersję trzeba przypiąć.
- Ręczną pętlę warto odpuścić. Dokumentacja Anthropica zaleca tool runner jako domyślny
  wybór; każda iteracja zwraca wiadomość asystenta *przed* wykonaniem narzędzi, więc
  gating, human-in-the-loop i podmiana wyniku narzędzia są dostępne bez pisania pętli
  ręcznie.
- Przy routingu przez LiteLLM sprawdź, czy proxy przepuszcza nagłówek `anthropic-beta`
  bez modyfikacji (pass-through `/anthropic` zwykle tak, unified endpoint bywa różnie).
- Ustaw `ANTHROPIC_LOG=info` w kroku TeamCity. Tool runner zwraca modelowi błąd narzędzia
  jako `is_error: true` z samym komunikatem wyjątku; pełny stack trace trafia wyłącznie
  do standardowego `logging`.

## Rekomendowany podział warstwowy

| Tier | Zakres | Technologia |
|---|---|---|
| 1 | Triage: klasyfikacja findingu | `client.messages.parse()` — structured output, bez narzędzi |
| 2 | Bump wersji + weryfikacja buildu | `tool_runner` z wąskim zestawem narzędzi |
| 3 | Remediacja kodu (breaking changes) | Agent SDK |

Agent SDK odpala się wtedy tylko dla przypadków, w których tier 2 zwrócił „build nie
przechodzi po bumpie". Zysk jest podwójny: niższy koszt i czas builda, oraz deterministyczna
ścieżka działająca, gdy warstwa agentowa zawiedzie.
