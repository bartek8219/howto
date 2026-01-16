---

# Instrukcja: Multi-Agent Workflow z Codex 0.80.0 - Od Zera

Utworzymy prosty system, gdzie Python script będzie zarządzać agentami pracującymi nad projektem.

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

