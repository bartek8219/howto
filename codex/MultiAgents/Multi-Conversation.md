<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# Codex CLI 0.79.0 - Funkcjonalność Multi-Conversation Agent Control

## Czym jest Multi-Conversation Agent Control?

Funkcjonalność **multi-conversation agent control** dodana w wersji Codex 0.79.0 (opublikowanej 19 grudnia 2025) umożliwia sesji Codex programowe uruchamianie i wysyłanie wiadomości do innych konwersacji. Jest to fundamentalna technologia umożliwiająca tworzenie systemów **sub-agentów** — gdzie jeden agent orchestrator może koordynować pracę wielu wyspecjalizowanych agentów pracujących równocześnie lub sekwencyjnie.[^1_1][^1_2][^1_3][^1_4]

Funkcjonalność stanowi **"infrastrukturę" (plumbing)** dla implementacji wieloagentowych workflow-ów w Codex. Pozwala na tworzenie architektur, w których centralny orchestrator deleguje prace poszczególnym agentom, monitoruje ich postęp i zarządza handoff-ami między nimi.[^1_4]

## Dostępne API

### API App Server (Level protokołu)

Funkcjonalność jest dostępna poprzez app-server Codex, który ekspozuje następujące endpointy JSON-RPC:[^1_5]


| Endpoint | Opis | Zastosowanie |
| :-- | :-- | :-- |
| **thread/start** | Tworzy nową konwersację (thread) | Uruchamianie nowego agenta |
| **thread/resume** | Wznawia wcześniej zapisaną konwersację | Kontynuowanie pracy istniejącego agenta |
| **thread/fork** | Fork istniejącej konwersacji w nowy thread (0.80.0+) | Branching alternatywnych ścieżek decyzyjnych |
| **turn/start** | Uruchamia pojedynczą turę w konwersacji | Wysyłanie promptu do agenta |
| **turn/interrupt** | Przerywa bieżącą turę | Zatrzymanie pracy agenta |

Każde żądanie otrzymuje notyfikacje zwrotne zawierające `turn/started`, `turn/completed`, oraz detale `item/*`.

### Interfejs SDK TypeScript

Jeśli pracujesz poprzez Codex SDK TypeScript (`@openai/codex`), dostęp odbywa się bardziej intuicyjnie:[^1_6]

```typescript
// Tworzenie nowej konwersacji
const thread = codex.startThread();

// Wznowienie zapisanej konwersacji
const thread = codex.resumeThread(savedThreadId);

// Uruchamianie promptu
const turn = await thread.run("explain this codebase");

// Streamowanie odpowiedzi
const { events } = await thread.runStreamed("your prompt");
```


## Praktyczna implementacja - Multi-Agent Workflow

Dokumentacja, którą Ci przesłałem, pokazuje praktyczne zastosowanie za pomocą **OpenAI Agents SDK**. Oto struktura:

### Setup infrastruktury[^1_1]

Najpierw konfigurujemy Codex CLI jako MCP server:

```python
import asyncio
from agents import Agent, Runner
from agents.mcp import MCPServerStdio

async def main() -> None:
    async with MCPServerStdio(
        name="Codex CLI",
        params={
            "command": "npx",
            "args": ["-y", "codex", "mcp"],
        },
        client_session_timeout_seconds=360000,
    ) as codex_mcp_server:
        # Tutaj definiujemy agentów
        return
```


### Definiowanie agentów z handoff-ami[^1_1]

Każdy agent posiada swoje instrukcje oraz referencje do innych agentów, do których może przekazać kontrolę:

```python
designer_agent = Agent(
    name="Designer",
    instructions="You are the Designer...",
    model="gpt-5",
    mcp_servers=[codex_mcp_server],
)

developer_agent = Agent(
    name="Game Developer",
    instructions="You are an expert in building games...",
    mcp_servers=[codex_mcp_server],
)

# Konfiguracja handoff-ów - Designer przekazuje do Developer-a
designer_agent.handoffs = [developer_agent]
```


### Orchestration pattern - Project Manager[^1_1]

Dokumentacja pokazuje zaawansowany wzorzec z **Project Manager** agenttem, który:

1. **Czyta** specification/task list od użytkownika
2. **Tworzy** trzy dokumenty bazowe:
    - `REQUIREMENTS.md` — specyfikacja produktu
    - `TEST.md` — warunki akceptacji
    - `AGENT_TASKS.md` — instrukcje dla każdego agenta
3. **Handoff sekwencyjny z walidacją**:
    - Handoff do Designer-a
    - **Czeka** na stworzenie `design_spec.md`
    - Dopiero wtedy handoff do Frontend/Backend developerów (równolegle)
    - **Weryfikuje** istnienie ich plików wyjściowych
    - Na koniec handoff do Tester-a
4. **Nie wysyła status updates** — jedynie handoff-y, eliminując szum kontekstowy
```python
project_manager_agent = Agent(
    name="Project Manager",
    instructions="""
    Convert input task list into three project files.
    
    Process:
    1) Create REQUIREMENTS.md, TEST.md, AGENT_TASKS.md
    2) Handoff to Designer → wait for design_spec.md
    3) Handoff in parallel to Frontend and Backend → wait for outputs
    4) Handoff to Tester with all artifacts
    
    Do NOT respond with status updates. Just handoff to next agent.
    """,
    handoffs=[designer_agent, frontend_dev_agent, backend_dev_agent, tester_agent],
    mcp_servers=[codex_mcp_server],
)
```


## Co musisz zrobić, aby tego użyć?

### Wymogi wstępne

1. **Zainstaluj zależności**:[^1_1]
```bash
python -m venv .venv
source .venv/bin/activate
pip install --upgrade openai openai-agents python-dotenv
```

2. **Zainstaluj Codex CLI**:[^1_1]
```bash
npm install -g @openai/codex@0.79.0
```

3. **Ustawienia OpenAI API key** — przechowaj w `.env`:
```bash
OPENAI_API_KEY=sk-...
```


### Kroki implementacji

**Krok 1: Setup MCP Server**

Stwórz `codex_mcp.py`:

```python
import os
from dotenv import load_dotenv
from agents import Agent, Runner, set_default_openai_api
from agents.mcp import MCPServerStdio

load_dotenv(override=True)
set_default_openai_api(os.getenv("OPENAI_API_KEY"))

async def main() -> None:
    async with MCPServerStdio(
        name="Codex CLI",
        params={"command": "npx", "args": ["-y", "codex", "mcp"]},
        client_session_timeout_seconds=360000,
    ) as codex_mcp_server:
        # Agenci będą się komunikować poprzez ten server
        pass
```

**Krok 2: Zdefiniuj swoje agenty**

Każdy agent potrzebuje:

- `name` — identyfikator
- `instructions` — szczegółowe wytyczne (czym się zajmuje)
- `model` — "gpt-5", "gpt-5-codex" itd.
- `handoffs` — lista agentów, którym może przekazać kontrolę
- `mcp_servers=[codex_mcp_server]` — dostęp do Codex CLI

```python
planner_agent = Agent(
    name="Planner",
    instructions="You analyze the task and create a plan...",
    handoffs=[builder_agent, reviewer_agent],
    mcp_servers=[codex_mcp_server],
)

builder_agent = Agent(
    name="Builder",
    instructions="You implement the code based on the plan...",
    handoffs=[planner_agent],
    mcp_servers=[codex_mcp_server],
)

reviewer_agent = Agent(
    name="Reviewer",
    instructions="You review the code for quality...",
    handoffs=[planner_agent],
    mcp_servers=[codex_mcp_server],
)
```

**Krok 3: Uruchom workflow**

```python
task = """
Build a chess game in HTML/CSS/JS.
Planner breaks down requirements.
Builder implements the code.
Reviewer checks quality and suggests improvements.
"""

result = await Runner.run(planner_agent, task, max_turns=30)
```


### Kluczowe wzory (Best Practices)

| Praktyka | Opis | Dlaczego |
| :-- | :-- | :-- |
| **Gated Handoffs** | Przeprowadź handoff dopiero gdy wymagane pliki istnieją | Zapobieganie blind handoff-om; agenci mają pewność że warunki spełnione |
| **Task Segregation** | Każdy agent ma odrębne foldery wyjściowe (`/design`, `/frontend`, `/backend`) | Izolacja artefaktów; czytelny kontekst; eliminacja konfliktów |
| **Approval Policies** | Ustaw `approval-policy: "never"` w MCP callach dla non-interactive agentów | Agenci mogą pracować autonomicznie bez czekania na human approval |
| **Sandbox Config** | Ustaw `sandbox: "workspace-write"` do write operacji | Bezpieczna izolacja; kontrola uprawnień |
| **No Status Updates** | Handoff bez pośrednich raportów statusu | Oszczędność tokenów; czystszy kontekst |
| **Parallel Where Possible** | Frontend i Backend mogą pracować równocześnie po Design | Przyśpieszenie; nie blokowanie na sekwencyjnych zależnościach |

## Ograniczenia i uwagi

1. **Model context** — każdy agent otrzymuje pełną historię konwersacji, którą sobie zachowuje. Dla długotrwałych projektów, `/compact` endpoint (w GPT-5 Responses API) może być pomocny do kompresji kontekstu.[^1_7]
2. **Nauczone zachowania modelu** — GPT-5 jest "naturalnie pomocny" i czasem agent orchestrator chce sam robić pracę zamiast delegować. Rozwiązanie: ograniczenie dostępu do write-tools dla coordinatora lub jawne instrukcje o delegacji.[^1_8]
3. **Konflikt zmian** — jeśli wielu agentów edytuje te same linie kodu równolegle, mogą powstać konflikty przy merging. Best practice: tematyczne rozdzielenie pracy między agentów.[^1_9]
4. **Token usage** — każde spawn-owanie nowego agenta tworzy nową konwersację; monitoruj usage, szczególnie przy wielu handoff-ach.

## Podsumowanie

**Multi-conversation agent control** to technika umożliwiająca Codex zarządzanie wieloagentowymi systemami. Implementujesz to poprzez:

1. ✅ Setup Codex CLI jako MCP server w OpenAI Agents SDK
2. ✅ Definiowanie agentów z jasnym `instructions` i `handoffs`
3. ✅ Orchestration pattern z gatingiem handoff-ów na podstawie artefaktów
4. ✅ Uruchomienie via `Runner.run()` z max_turns dla bezpieczeństwa

Praktycznie: Stwórz agenta orchestrator-a (Project Manager), który czyta specification, tworzy dokumenty bazowe, i sekwencyjnie/równolegle deleguje prace do wyspecjalizowanych agentów (Designer, Developer, Tester), weryfikując każdy krok przed przejściem do następnego.[^1_1]
<span style="display:none">[^1_10][^1_11][^1_12][^1_13][^1_14][^1_15][^1_16][^1_17][^1_18][^1_19][^1_20][^1_21][^1_22][^1_23][^1_24][^1_25][^1_26][^1_27][^1_28][^1_29][^1_30][^1_31][^1_32][^1_33][^1_34][^1_35][^1_36][^1_37][^1_38][^1_39]</span>

<div align="center">⁂</div>

[^1_1]: https://developers.openai.com/codex/guides/agents-sdk/

[^1_2]: https://releasebot.io/updates/openai/codex

[^1_3]: https://github.com/openai/codex/discussions/8862

[^1_4]: https://www.reddit.com/r/codex/comments/1q6ecfd/multiconversation_agent_control/

[^1_5]: https://fossies.org/linux/codex-rust/codex-rs/app-server/README.md

[^1_6]: https://blog.stackademic.com/openai-codex-sdk-for-creating-our-own-codex-agent-bee5ad08fe57

[^1_7]: https://cookbook.openai.com/examples/gpt-5/gpt-5-1-codex-max_prompting_guide

[^1_8]: https://www.reddit.com/r/codex/comments/1q9hny1/finally_got_true_multiagent_group_chat_working_in/

[^1_9]: https://www.youtube.com/watch?v=Xt7BZYhJXkg

[^1_10]: https://community.openai.com/t/codex-generates-multiple-responses-simultaneously/1287639

[^1_11]: https://lobehub.com/mcp/andreahaku-codex_mcp

[^1_12]: https://www.reddit.com/r/CodexAutomation/comments/1q97f8m/codex_cli_updates_0780_0800_branching_threads/

[^1_13]: https://developers.openai.com/codex/mcp/

[^1_14]: https://github.com/openai/codex/releases

[^1_15]: https://releasebot.io/updates/openai

[^1_16]: https://code.visualstudio.com/docs/copilot/customization/mcp-servers

[^1_17]: https://www.freshports.org/misc/codex/

[^1_18]: https://developers.openai.com/codex/changelog/

[^1_19]: https://github.com/openai/codex/issues/8838

[^1_20]: https://www.reddit.com/r/ClaudeCode/comments/1myp5mc/is_there_a_way_to_make_claude_code_and_codex_cli/

[^1_21]: https://developers.openai.com/codex/cli/

[^1_22]: https://developers.openai.com/codex/cli/slash-commands/

[^1_23]: https://blog.promptlayer.com/how-openai-codex-works-behind-the-scenes-and-how-it-compares-to-claude-code/

[^1_24]: https://developers.openai.com/codex/cli/features/

[^1_25]: https://steipete.me/posts/just-talk-to-it

[^1_26]: https://developers.openai.com/changelog/

[^1_27]: https://openai.com/index/introducing-upgrades-to-codex/

[^1_28]: https://sourceforge.net/projects/openai-codex.mirror/files/rust-v0.79.0/

[^1_29]: https://github.com/openai/codex/issues/4972

[^1_30]: https://developers.openai.com/codex/config-reference/

[^1_31]: https://www.reddit.com/r/OpenAIDev/new/

[^1_32]: https://github.com/openai/codex/issues/4514

[^1_33]: https://developers.openai.com/codex/guides/agents-md/

[^1_34]: https://www.reddit.com/r/OpenAI/comments/1qbcb38/codex_as_a_chatgpt_app_chat_in_the_web_app_and/

[^1_35]: https://www.reddit.com/r/Anthropic/comments/1plvv9h/friend_open_sourced_mcp_for_cc_that_talks_to/

[^1_36]: https://developers.openai.com/codex/config-advanced/

[^1_37]: https://hexdocs.pm/codex_sdk/

[^1_38]: https://dev.to/proflead/how-to-use-openais-codex-agent-in-vs-code-a-step-by-step-guide-2oa5

[^1_39]: https://www.recent.dev


---

# mam juz codex ale nie rozumiem pierwszego kroku instalacji zależności, czy to jest konieczne?

Nie, nie jest konieczne, jeśli **nie chcesz używać OpenAI Agents SDK**.

Rozłożmy to na przypadki:

## Przypadek 1: Używasz Codex CLI bezpośrednio (bez Agents SDK)

Jeśli uruchamiasz Codex tylko z linii poleceń:

```bash
npx codex
# lub po instalacji
codex
```

**Nie potrzebujesz** instalować zależności Pythona. Po prostu uruchamiasz CLI i rozmawiasz z agentem w terminalu. To jest najbardziej bezpośrednie podejście.

## Przypadek 2: Chcesz programowo kontrolować Codex z kodu Python/Node.js

Wtedy **tak, potrzebujesz** zainstalować SDK:

```bash
# Dla Python
pip install openai openai-agents

# Lub dla TypeScript/JavaScript
npm install @openai/codex
```

Ponieważ dokumentacja, którą Ci przesłałem, pokazuje **zaawansowany wzorzec multi-agent** — gdzie Python script spawnuje agentów, kontroluje handoff-y, sprawdza czy pliki istnieją, itp. To wymaga SDK-a.

## Praktycznie dla Ciebie

Jeśli:

- ✅ Chcesz po prostu używać Codex w terminalu → **żadne zależności Pythona nie są potrzebne**
- ✅ Chcesz integrować Codex z własnym kodem (np. automatyzacja, orchestration) → **zainstaluj SDK**

Zakładam, że Ty chcesz robić bardziej zaawansowane rzeczy (multi-agent workflows), bo inaczej nie pytałbyś o multi-conversation. Ale jeśli interesujesz Cię **samo wdrożenie**:

Wystarczy, że masz:

1. **Node.js 18+** (dla `npx codex`)
2. **OpenAI API key** (w `.env` lub zmiennej `OPENAI_API_KEY`)
3. Codex CLI (już masz)

Depencje Pythona (`openai`, `openai-agents`) potrzebne są **tylko** jeśli tworzysz własny orchestrator script w Pythonie, który będzie programowo zarządzać agentami.[^2_1]

Czy chcesz pracować w terminalu czy tworzysz automation script?

<div align="center">⁂</div>

[^2_1]: https://developers.openai.com/codex/guides/agents-sdk/


---

# czy może użyć opcji multi-agent jeśli uruchamiam codex w terminalu powershell bez kontrolowania tego z poziomu kodu python?

Tak, można, ale **nie w tradycyjny sposób multi-agent workflow** pokazany w dokumentacji.

Rozłożę ci dostępne opcje:

## Opcja 1: Slash commands w CLI - Prosta delegacja

W terminalu PowerShell, gdy już jesteś w sesji Codex, możesz używać slash commands:[^3_1]

```bash
codex
# Jesteś w interactive sesji

# Utwórz nowy thread/konwersację
/new

# Wznów poprzednią
/resume <thread-id>

# Fork istniejącą konwersację
/fork
```

Każde `/new` tworzy nową niezależną konwersację. Możesz mieć **wiele okien terminala** otwartych jednocześnie, każde z innym agentem. Ale to nie jest **orchestration** — ty ręcznie zarządzasz handoff-ami między nimi (kopiując kontekst, opisując co poprzedni agent zrobił).

## Opcja 2: AGENTS.md - Instrukcje dla agenta o sub-rolach

Bardziej eleganckie podejście: stwórz plik `AGENTS.md` w projekcie. Powiedz agencie, że ma **różne role** i może się "przełączać" między nimi:[^3_2]

```markdown
# AGENTS.md

## Setup

You are an agent that can take on multiple roles:

### Role 1: Designer
- Analyzes requirements
- Creates UI/UX specification
- Outputs: design_spec.md

### Role 2: Developer  
- Implements code based on design
- Reads design_spec.md
- Outputs: implementation files

### Role 3: Tester
- Reviews code for bugs
- Tests functionality
- Outputs: test_report.md

## Instructions

When user provides a task:
1. As Designer: analyze and create design_spec.md
2. Ask user: "Should I proceed to Developer role?"
3. As Developer: implement based on spec
4. Ask user: "Should I proceed to Tester role?"
5. As Tester: verify and report

Switch roles by stating explicitly which role you're taking.
```

Wtedy w terminalu:

```bash
codex

# Wpisujesz:
Build a calculator web app

# Agent czyta AGENTS.md i:
# - Przyjmuje rolę Designer-a
# - Tworzy design_spec.md
# - Pyta czy kontynuować

# Ty odpowiadasz:
yes, proceed to developer

# Agent przełącza się na Developer role
# - Implementuje kod
# - Pyta czy testować

# Ty odpowiadasz:
yes, proceed to tester

# Agent przełącza się na Tester role
# - Sprawdza kod
# - Wypisuje raport
```


## Opcja 3: Ręczne handoff-y między sesjami CLI

Bardziej manualne, ale działa bez Pythona:

```powershell
# Terminal 1 - Designer
codex
# Designer tworzy design_spec.md
# Kopiujesz output

# Terminal 2 - Developer
codex
# Pastelujesz design_spec.md jako kontekst
# Developer implementuje

# Terminal 3 - Tester
codex
# Pastelujesz kod i design
# Tester sprawdza
```

Każdy terminal to osobna sesja, ale ty ręcznie koordynujesz handoff-y.

## Najlepsze podejście dla PowerShell (bez Pythona)

**Opcja 2 z AGENTS.md** to najrealistyczniejsze w Twoim przypadku:

1. ✅ Tworysz `AGENTS.md` w katalogu projektu
2. ✅ Definiujesz role i instrukcje jak się przełączać
3. ✅ Uruchamiasz `codex` w PowerShell
4. ✅ Agent czyta AGENTS.md i sam zarządza rolami
5. ✅ Ty tylko dajesz komendy typu "continue as Developer"

Nie ma true orchestration (bez Pythona/SDK), ale agent **emuluje** multi-agent workflow w ramach jednej konwersacji.[^3_2]

***

**Podsumowanie:**

- Multi-conversation agent control z gatingiem (dokumentacja) → **wymaga Pythona + SDK**
- Multi-role agent w jednej sesji CLI → **Opcja 2 (AGENTS.md), bez Pythona, działa w PowerShell**[^3_2]
- Ręczne handoff-y między sesjami → **działa, ale manualne**

Która opcja Cię interesuje?

<div align="center">⁂</div>

[^3_1]: https://developers.openai.com/codex/cli/slash-commands/

[^3_2]: https://developers.openai.com/codex/guides/agents-md/


---

# załóżmy że użyję opcji z pythonem, czy wtedy będę mógł użyć sesji codexa które już wcześniej utworzyłem i chciałbym kontynuować promptowanie?

Tak, można. To jest dokładnie do tego funkcjonalność **thread/resume** API.[^4_1][^4_2]

## Jak to działa

Gdy Codex CLI tworzy sesję, przypisuje jej **unikatowy ID thread-a**. Możesz go wznowić (resume) z poziomu Pythona:

### Krok 1: Pobierz ID swojej istniejącej sesji

W PowerShell, gdy jesteś w sesji Codex:

```bash
codex
# Patrz na top bar terminala - tam widać coś jak:
# [Thread ID: abc123def456...]
# lub /status pokaże ci ID
```

Albo sprawdź katalog `.codex/` w swoim projekcie — tam są zapisane sesje.

### Krok 2: Resume w Pythonie

```python
import asyncio
import os
from dotenv import load_dotenv
from agents import Agent, Runner, set_default_openai_api
from agents.mcp import MCPServerStdio

load_dotenv(override=True)
set_default_openai_api(os.getenv("OPENAI_API_KEY"))

async def main() -> None:
    async with MCPServerStdio(
        name="Codex CLI",
        params={
            "command": "npx",
            "args": ["-y", "codex", "mcp"],
        },
        client_session_timeout_seconds=360000,
    ) as codex_mcp_server:
        # Twój agent orchestrator
        orchestrator = Agent(
            name="Orchestrator",
            instructions="""
            You can resume existing Codex sessions.
            Use the Codex MCP to continue work in previously created conversations.
            """,
            mcp_servers=[codex_mcp_server],
        )
        
        # Resume istniejącej sesji
        # Wysyłasz prompt który mówi agentowi by wznowił konkretny thread
        result = await Runner.run(
            orchestrator, 
            "Resume thread with ID abc123def456 and continue implementing the feature",
            max_turns=10
        )
```


### Krok 3: Bezpośredni dostęp do thread API (bardziej zaawansowany)

Jeśli pracujesz z TypeScript/SDK, możesz bezpośrednio:

```typescript
const codex = new Codex();

// Resume istniejącego thread-a
const thread = codex.resumeThread("abc123def456");

// Kontynuuj promptowanie
const turn = await thread.run("Now implement the database layer");
const { events } = await thread.runStreamed("another prompt");
```


## Praktycznie dla Ciebie

Scenariusz:

1. Tworzysz sesję w PowerShell i pracujesz nad projektem (np. Codex tworzy plik `index.html`)
2. Zapisujesz sesję i zamykasz terminal
3. Następnego dnia chcesz kontynuować **z tego samego punktu**
4. Tworzysz Python script, który resume-uje stary thread i dodaje nowe instrukcje
```python
async def main() -> None:
    async with MCPServerStdio(
        name="Codex CLI",
        params={"command": "npx", "args": ["-y", "codex", "mcp"]},
        client_session_timeout_seconds=360000,
    ) as codex_mcp_server:
        
        coordinator = Agent(
            name="Coordinator",
            instructions="""
            Resume the existing Codex session and continue the work.
            The context of previous conversation is preserved.
            """,
            mcp_servers=[codex_mcp_server],
        )
        
        # Wznów pracę
        await Runner.run(
            coordinator,
            "Resume thread abc123def456. We had started building a game. "
            "Now add the scoring system and save high scores to localStorage.",
            max_turns=15
        )
```


## Ważne uwagi

| Punkt | Szczegóły |
| :-- | :-- |
| **Thread persistence** | Threads są przechowywane lokalnie w `.codex/` — wciąż dostępne po restarcie PowerShella[^4_1] |
| **Context preservation** | Resume zachowuje pełny kontekst poprzedniej konwersacji — agent zna co robił wcześniej[^4_1] |
| **Handoff + Resume** | Możesz kombinować: Resume thread → Agent pracuje → Handoff do innego agenta → Ten agent også może resumować inny thread[^4_1] |
| **Max turns** | Ustaw odpowiedni `max_turns` aby agent nie pracował w nieskończoność |
| **Session timeout** | `client_session_timeout_seconds=360000` — 100 godzin, wystarczająco na długą sesję |

## Kompletnszy przykład

```python
import asyncio
import os
from dotenv import load_dotenv
from agents import Agent, Runner, set_default_openai_api
from agents.mcp import MCPServerStdio

load_dotenv(override=True)
set_default_openai_api(os.getenv("OPENAI_API_KEY"))

async def main() -> None:
    async with MCPServerStdio(
        name="Codex CLI",
        params={"command": "npx", "args": ["-y", "codex", "mcp"]},
        client_session_timeout_seconds=360000,
    ) as codex_mcp_server:
        
        # Zdefiniuj agentów
        designer = Agent(
            name="Designer",
            instructions="You design the UI/UX for features",
            mcp_servers=[codex_mcp_server],
        )
        
        developer = Agent(
            name="Developer",
            instructions="You implement code based on designs",
            handoffs=[designer],
            mcp_servers=[codex_mcp_server],
        )
        
        designer.handoffs = [developer]
        
        # Resume istniejącego thread-a I kontynuuj
        task = """
        Resume the existing game development session (thread ID: your_thread_id_here).
        The previous context should be preserved.
        Now we need to:
        1. Add sound effects
        2. Improve mobile responsiveness
        3. Add difficulty levels
        
        Designer: Create specs for these features.
        Developer: Implement them.
        """
        
        result = await Runner.run(developer, task, max_turns=20)
        print(result.final_output)

if __name__ == "__main__":
    asyncio.run(main())
```


## Znalezienie ID thread-a

W PowerShell:

```bash
# Jeśli jesteś w sesji Codex
/status
# Wyświetli ID aktualnego thread-a

# Lub sprawdź katalog
ls .codex/
# Tam są foldery z ID thread-ów
```


***

**Odpowiedź na Twoje pytanie: TAK** — możesz:

1. Pracować w PowerShell z Codex (create sesja)
2. Zamknąć terminal
3. Następnie z Pythona resume ten thread
4. Kontynuować promptowanie w tym samym kontekście[^4_2][^4_1]

Czy potrzebujesz konkretnego ID thread-a do testowania, czy mogę pokazać jak go znaleźć?

<div align="center">⁂</div>

[^4_1]: https://developers.openai.com/codex/guides/agents-sdk/

[^4_2]: https://fossies.org/linux/codex-rust/codex-rs/app-server/README.md


---

# Instrukcja: Multi-Agent Workflow z Codex 0.80.0 - Od Zera

Vytworzymy prosty system, gdzie Python script będzie zarządzać agentami pracującymi nad projektem.

***

## ETAP 1: Przygotowanie środowiska (5 min)

### Krok 1.1: Sprawdzenie wersji Codex

Otwórz PowerShell i sprawdź czy masz Codex:

```powershell
codex --version
```

Powinno pokazać: `0.80.0` lub wyższą.

### Krok 1.2: Instalacja Python

Pobierz Python z https://www.python.org/downloads/ (Python 3.10+)

Podczas instalacji **zaznacz checkbox**: `Add Python to PATH`

Sprawdzenie w PowerShell:

```powershell
python --version
```

Powinno pokazać: `Python 3.10.x` lub wyższe.

### Krok 1.3: Sprawdzenie OpenAI API Key

Musisz mieć OpenAI API Key. Pobierz z: https://platform.openai.com/api/keys

***

## ETAP 2: Tworzenie projektu (10 min)

### Krok 2.1: Stwórz folder projektu

W PowerShell:

```powershell
mkdir codex-project
cd codex-project
```


### Krok 2.2: Utwórz plik `.env` z API Key

W PowerShell, będąc w folderze `codex-project`:

```powershell
# Windows PowerShell - utwórz plik .env
@"
OPENAI_API_KEY=sk-xxxxxxxxxxxxxx
"@ | Out-File -Encoding UTF8 .env
```

**Zamień `sk-xxxxxxxxxxxxxx` swoim rzeczywistym API Key!**

### Krok 2.3: Stwórz wirtualne środowisko Pythona

Pozostając w PowerShell w folderze `codex-project`:

```powershell
python -m venv .venv
```

To utworzy folder `.venv` — to izolowane środowisko Pythona.

### Krok 2.4: Aktywuj wirtualne środowisko

```powershell
.venv\Scripts\Activate.ps1
```

Powinno pojawić się `(.venv)` na początku linii w PowerShell — oznacza to, że jesteś w wirtualnym środowisku.

### Krok 2.5: Zainstaluj zależności

```powershell
pip install openai openai-agents python-dotenv
```

To zajmie 1-2 minuty. Czekaj aż się skończy.

**Ważne:** Jeśli pojawi się błąd o `Microsoft Visual C++` — zainstaluj go stąd: https://visualstudio.microsoft.com/visual-cpp-build-tools/

***

## ETAP 3: Tworzenie Python Script-u (5 min)

### Krok 3.1: Otwórz edytor tekstu

Otwórz **VS Code** lub **Notepad++**

### Krok 3.2: Stwórz plik `orchestrator.py`

Skopiuj poniższy kod i zapisz jako `orchestrator.py` **w folderze `codex-project`**:

```python
import asyncio
import os
from dotenv import load_dotenv
from agents import Agent, Runner, set_default_openai_api
from agents.mcp import MCPServerStdio

# Wczytaj API Key z .env
load_dotenv(override=True)
set_default_openai_api(os.getenv("OPENAI_API_KEY"))

async def main() -> None:
    # Uruchom Codex CLI jako MCP server
    async with MCPServerStdio(
        name="Codex CLI",
        params={
            "command": "npx",
            "args": ["-y", "codex", "mcp"],
        },
        client_session_timeout_seconds=360000,
    ) as codex_mcp_server:
        
        # Agent 1: Designer - projektuje UI
        designer_agent = Agent(
            name="Designer",
            instructions=(
                "You are a UI/UX designer. "
                "Your job is to analyze requirements and create a design specification. "
                "Write your design to a file called design_spec.md in the current directory. "
                "When done, summarize what you created and stop."
            ),
            model="gpt-5",
            mcp_servers=[codex_mcp_server],
        )
        
        # Agent 2: Developer - implementuje kod
        developer_agent = Agent(
            name="Developer",
            instructions=(
                "You are a web developer. "
                "Read the design_spec.md file to understand requirements. "
                "Create an index.html file with HTML, CSS, and JavaScript. "
                "Make sure the code works and follows the design specification. "
                "Save your work to index.html in the current directory. "
                "When done, tell the designer what you implemented."
            ),
            model="gpt-5",
            mcp_servers=[codex_mcp_server],
        )
        
        # Konfiguracja handoff-ów (kto może przekazać do kogo)
        designer_agent.handoffs = [developer_agent]
        developer_agent.handoffs = [designer_agent]
        
        # Główne zadanie
        task = """
        Create a simple TODO app with:
        - Input field to add tasks
        - List of tasks
        - Ability to mark tasks as done
        - Delete button for each task
        - Simple and clean design
        
        Process:
        1. Designer: Create design specification
        2. Developer: Implement the HTML/CSS/JavaScript code
        """
        
        # Uruchom workflow
        print("Starting multi-agent workflow...")
        result = await Runner.run(designer_agent, task, max_turns=15)
        print("\n=== WORKFLOW COMPLETED ===")
        print(result.final_output)

if __name__ == "__main__":
    asyncio.run(main())
```

**Gdzie zapisać:** `C:\Users\TWOJA_NAZWA\codex-project\orchestrator.py`

***

## ETAP 4: Uruchomienie (2 min)

### Krok 4.1: Upewnij się, że wirtualne środowisko jest aktywne

W PowerShell powinno być `(.venv)` na początku linii.

Jeśli nie, uruchom:

```powershell
.venv\Scripts\Activate.ps1
```


### Krok 4.2: Uruchom script

```powershell
python orchestrator.py
```

**Co się będzie działo:**

1. Script uruchomi Codex CLI jako serwer
2. Designer agent przeanalizuje zadanie i stwórzy `design_spec.md`
3. Developer agent przeczyta specyfikację i stwórzy `index.html`
4. Oba agenty będą komunikować się poprzez Codex MCP

Będziesz widział output w PowerShell — czekaj aż się skończy.

### Krok 4.3: Sprawdzenie wyników

Po zakończeniu, w folderze `codex-project` powinieneś mieć:

```
codex-project/
├── design_spec.md          ← Specyfikacja od Designer-a
├── index.html              ← Kod od Developer-a
├── orchestrator.py         ← Twój script
├── .env                    ← API Key
└── .venv/                  ← Wirtualne środowisko Python
```

Aby zobaczyć TO-DO app w przeglądarce:

```powershell
# Windows PowerShell
Start-Process index.html
# Lub ręcznie otwórz index.html w przeglądarce
```


***

## ETAP 5: Resume Istniejącej Sesji (Bonus)

Jeśli chcesz kontynuować pracę nad istniejącym projektem:

### Krok 5.1: Znajdź ID sesji

W PowerShell:

```powershell
ls .codex/
```

Zobaczysz folder z ID thread-a (np. `abc123def456`)

### Krok 5.2: Modyfikuj script

Zmień część tasków w `orchestrator.py`:

```python
task = """
Resume the TODO app project.
Now add the following features:
- Save tasks to browser storage (localStorage)
- Load tasks when page refreshes
- Add categories for tasks
"""
```

I uruchom ponownie:

```powershell
python orchestrator.py
```


***

## Troubleshooting

| Problem | Rozwiązanie |
| :-- | :-- |
| `'python' is not recognized` | Zainstaluj Python, zaznaczając `Add to PATH` |
| `ModuleNotFoundError: No module named 'agents'` | Uruchom: `pip install openai openai-agents` (musi być w `.venv`) |
| `OPENAI_API_KEY not found` | Sprawdź `.env` — czy API Key jest prawidłowy |
| `codex: command not found` | Zainstaluj Codex: `npm install -g @openai/codex` |
| `.venv\Scripts\Activate.ps1 cannot be loaded` | Uruchom PowerShell as Administrator |


***

## Struktura projektu — Podsumowanie

```powershell
C:\Users\TWOJA_NAZWA\codex-project\
│
├── .env                          # API Key (nie udostępniaj!)
├── orchestrator.py               # Python script (uruchamiasz to)
├── design_spec.md                # Wygenerowany plik od Designer-a
├── index.html                    # Wygenerowany kod od Developer-a
│
├── .venv\                        # Wirtualne środowisko (tworzysz 1 raz)
│   ├── Scripts\
│   │   └── Activate.ps1          # Używasz do aktywacji
│   └── Lib\                      # Zależności (openai, agents, itd.)
│
└── .codex\                       # Lokalne sesje Codex (auto-created)
    └── thread-id-12345\          # Zapisane konwersacje
```


***

## Krótka instrukcja powtórka (dla następnego razu)

1. Otwórz PowerShell w folderze `codex-project`
2. Aktywuj environment: `.venv\Scripts\Activate.ps1`
3. Uruchom: `python orchestrator.py`
4. Czekaj na wyniki

To wszystko! 🎉

***

Czy wszystko jest jasne? Jeśli masz pytania do konkretnego kroku, pytaj!

