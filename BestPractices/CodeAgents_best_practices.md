# 🤖 AI Coding Agents – Best Practices

Zbiór 15 najlepszych praktyk korzystania z agentów kodujących takich jak **Claude Code** i **Codex CLI**.

---

## 📁 Konfiguracja fundamentów

### 1. Twórz i utrzymuj pliki `CLAUDE.md` / `AGENTS.md`

To najważniejszy punkt startowy. `CLAUDE.md` jest wczytywany automatycznie na początku każdej sesji – umieszczaj tam:

- strukturę repozytorium i opis głównych modułów
- komendy build / test / lint
- konwencje nazewnictwa branchy i commitów
- reguły architektoniczne specyficzne dla projektu

> 💡 Krótki i precyzyjny plik jest lepszy niż długi – agent ignoruje reguły, które giną w szumie.

### 2. Stosuj hierarchię plików konfiguracyjnych

Twórz pliki konfiguracyjne na trzech poziomach:

| Poziom | Lokalizacja | Zakres |
|---|---|---|
| Globalny | `~/.claude/CLAUDE.md` | Wszystkie projekty |
| Repozytorium | `./CLAUDE.md` | Cały projekt |
| Podkatalog | `./src/module/CLAUDE.md` | Specyficzny moduł |

Reguły bliższe bieżącemu katalogowi mają wyższy priorytet. Szczególnie przydatne w monorepo, gdzie różne moduły mogą mieć inne konwencje.

### 3. Inicjalizuj projekt komendą `/init`

Komenda `/init` automatycznie skanuje strukturę projektu, wykrywa frameworki, systemy testów i wzorce kodu, generując szkielet `CLAUDE.md` / `AGENTS.md` jako punkt wyjścia do dalszej edycji. Traktuj output jako draft – zawsze przejrzyj i uzupełnij wygenerowany plik.

---

## 🧠 Promptowanie i planowanie

### 4. Używaj trybu Plan Mode przed kodowaniem

Dla złożonych zadań najpierw wejdź w Plan Mode (`/plan` lub `Shift+Tab`), pozwól agentowi eksplorować i zadawać pytania, a dopiero potem przejdź do implementacji.

Zalecana sekwencja: **Explore → Plan → Implement → Commit**

Chroni przed pisaniem kodu dla źle zrozumianego problemu.

### 5. Stosuj Spec-Driven Development (SDD)

Przed każdym większym zadaniem generuj artefakty specyfikacji:

- `docs/requirements.md` – user stories z acceptance criteria
- `docs/plan.md` – plan implementacji z priorytetami i zależnościami
- `docs/tasks.md` – lista zadań z checkboxami `[ ]` pogrupowana w fazy

Agent realizuje je krok po kroku, co daje pełną kontrolę nad każdą zmianą zamiast budowania całej funkcji "na raz" w jednym prompcie.

Warto rozważyć użycie **OpenSpec** – open-source toolkit od GitHuba, który automatyzuje tworzenie i zarządzanie specyfikacjami w formacie przyjaznym agentom kodującym. OpenSpec standaryzuje strukturę plików spec, ułatwiając ich ponowne użycie i wersjonowanie razem z kodem.

### 6. Buduj prompt ze strukturą Goal / Context / Constraints / Done-when

Każdy prompt powinien zawierać cztery elementy:

| Element | Opis | Przykład |
|---|---|---|
| **Goal** | Co zbudować / naprawić | *"Fix login timeout bug"* |
| **Context** | Które pliki / błędy są istotne | *"Check src/auth/"* |
| **Constraints** | Konwencje, architektura, ograniczenia | *"No new dependencies"* |
| **Done-when** | Kryterium ukończenia | *"All tests pass"* |

❌ Źle: `"fix the login bug"`  
✅ Dobrze: `"Users report login fails after session timeout. Check src/auth/, write a failing test, then fix it and make sure the full test suite passes."`

### 7. Pozwól agentowi przeprowadzić z tobą wywiad

Zamiast samemu pisać spec, poproś agenta:

> I want to build X. Interview me about implementation, edge cases and tradeoffs, then write a complete SPEC.md

Agent zada pytania, których sam byś nie zadał, a potem zacznij nową sesję z gotowym `SPEC.md` jako kontekstem.

---

## 📦 Zarządzanie kontekstem

### 8. Zarządzaj kontekstem agresywnie

Okno kontekstowe to najważniejszy zasób – gdy się zapełni, agent zaczyna "zapominać" wcześniejsze instrukcje.

Przydatne komendy:

- `/clear` – wyczyść kontekst między niepowiązanymi zadaniami
- `/compact` – skompresuj historię gdy sesja jest długa
- `/rewind` – cofnij ostatnie zmiany

Dla dużych repozytoriów użyj serwera MCP **Repomix**, który upakuje cały codebase do zwięzłego, ustrukturyzowanego pliku – agent dostaje pełny obraz projektu bez marnowania kontekstu na eksplorację plików jeden po drugim.

> 💡 Zasada: jedna sesja = jeden spójny cel.

### 9. Deleguj eksplorację do subagentów

Zamiast pozwolić głównemu agentowi czytać setki plików i zapełniać kontekst, użyj subagentów do zbadania kodu:

> Use a subagent to investigate how authentication handles token refresh, then report back only the findings.

Subagent raportuje tylko wnioski, nie zaśmieca głównej sesji.

### 10. Stosuj wzorzec Writer / Reviewer z równoległymi sesjami

Uruchom dwie sesje:

- **Sesja A** – implementuje funkcję
- **Sesja B** (świeży kontekst) – robi code review

Świeży kontekst poprawia jakość recenzji, bo agent nie jest stronniczy wobec kodu, który sam napisał.

---

## ✅ Weryfikacja i niezawodność

### 11. Zawsze dawaj agentowi sposób weryfikacji jego pracy

Agent działa dramatycznie lepiej, gdy może sam uruchomić testy lub sprawdzić output. Dodaj do promptu:

> Write tests, run the test suite and fix any failures before finishing.

Bez kryterium sukcesu agent produkuje kod, który "wygląda dobrze", ale nie działa.

### 12. Konfiguruj Hooks dla deterministycznych akcji

W przeciwieństwie do instrukcji w `CLAUDE.md` (które są sugestią), hooki **gwarantują** wykonanie akcji po każdej operacji. Konfiguracja w `.claude/settings.json`.

Przykładowe hooki:

- `"Write a hook that runs eslint after every file edit"`
- `"Write a hook that runs dotnet format after saving .cs files"`
- `"Write a hook that blocks writes to the migrations folder"`

---

## 🚀 Skalowanie i automatyzacja

### 13. Pakuj powtarzalne workflow w Skills

Gdy ten sam prompt używasz wielokrotnie, utwórz `SKILL.md` w `.claude/skills/` lub `.agents/skills/`. Skill definiuje nazwę, opis i instrukcje – uruchamiasz go przez `/skill-name`.

Przykłady przydatnych Skills:

- Code review wg wewnętrznej checklisty
- Generowanie draft release notes
- Triage logów błędów

### 14. Podłączaj zewnętrzne systemy przez MCP

Gdy potrzebny kontekst leży poza repo (Jira, Figma, bazy danych, monitoring), użyj serwerów MCP zamiast kopiować dane do promptów.

Szczególnie wartościowy jest **Repomix MCP**, który eksponuje cały codebase jako narzędzia wywoływalne przez agenta w czasie rzeczywistym. Instalacja jedną komendą:

```bash
claude mcp add repomix -- npx -y repomix --mcp
```

Kluczowe funkcje Repomix MCP:

- `pack_codebase` z flagą `compress: true` – redukcja tokenów ~70% dzięki tree-sitterowi
- filtrowanie plików (`.repomixignore`)
- liczenie tokenów przed wysłaniem
- wbudowane sprawdzanie bezpieczeństwa (nie ujawni sekretów z `.env`)
- obsługa zdalnych repozytoriów przez URL GitHuba bez lokalnego klonowania

Zalecane workflow:

1. `pack_codebase (compress: true)` → przegląd architektury
2. `pack_codebase (full, wybrany moduł)` → głęboka analiza

> 💡 Dodawaj serwery MCP stopniowo – zacznij od 1-2 narzędzi, które eliminują realny manualny loop, zamiast dodawać wszystko na raz.

### 15. Ucz się z błędów i aktualizuj konfigurację iteracyjnie

Gdy agent dwa razy popełni ten sam błąd, poproś go o retrospektywę:

> You made this mistake twice. Add a rule to CLAUDE.md to prevent it.

Traktuj `CLAUDE.md` jak żywy dokument:

- commituj do gita razem z kodem
- regularnie usuwaj przestarzałe reguły
- testuj zmiany obserwując, czy zachowanie agenta faktycznie się zmienia

---

## 📚 Materiały dodatkowe

- [Claude Code – oficjalna dokumentacja](https://code.claude.com/docs/en/best-practices)
- [Codex CLI – best practices](https://developers.openai.com/codex/learn/best-practices/)
- [Repomix MCP Server](https://repomix.com/guide/mcp-server)
- [Spec-Driven Development with AI – JetBrains Blog](https://blog.jetbrains.com/junie/2025/10/how-to-use-a-spec-driven-approach-for-coding-with-ai/)
- [GitHub: OpenSpec / Spec-Driven Development toolkit](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)

---

*Ostatnia aktualizacja: Marzec 2026*
