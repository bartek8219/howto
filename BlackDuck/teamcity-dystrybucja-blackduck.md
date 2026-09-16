# Dystrybucja pipeline'u skanów Black Duck w TeamCity — opcje

Kontekst: cienki adapter uruchamiający trzy obrazy dockerowe (cała logika w obrazach),
parametry i secrety (Black Duck, LiteLLM, Jira) na poziomie adaptera.
Cel: udostępnienie rozwiązania całej organizacji.
Ograniczenie: **Versioned Settings wyłączone na serwerze**.

## Porównanie opcji

| | Szablon (root project) | Recipe | Biblioteka DSL |
|---|---|---|---|
| Wymaga Versioned Settings | nie | nie | **tak** |
| Wymaga TeamCity 2025.03+ | nie | **tak** | nie |
| Dystrybucja org-wide | dziedziczenie | dziedziczenie | per repo zespołu |
| Model dystrybucji | push | push | pull |
| Wersjonowanie w Git | nie | tak (YAML) | tak |
| Zakres | kroki + triggery + features + parametry + wymagania agenta | tylko kroki + inputs | wszystko |

## 1. Biblioteka DSL jako zależność w POM

- Jar z rozszerzeniami TeamCity DSL (np. `fun BuildType.blackDuckScan(...)`), publikowany do Artifactory
  lub wgrywany przez Administration | DSL Libraries.
- Konsumpcja: `.teamcity/pom.xml` w repo zespołu (plik generowany przez TeamCity, **nie** pom aplikacji)
  + wywołania w `settings.kts`.
- `<dependency>` wskazuje koordynaty, `<repositories>` wskazuje źródło; credentiale do Artifactory
  w pliku `mavenSettingsDsl.xml` wgranym raz na Root project | Maven Settings.
- **Blokada**: użycie `settings.kts` wymaga włączonych Versioned Settings — obecnie niedostępne.
- Kod biblioteki jest centralny, ale konsumpcja rozproszona: brak dziedziczenia (każdy zespół musi
  wciągnąć zależność świadomie), brak zasięgu (zespoły spoza config-as-code nie skorzystają),
  aktualizacja = PR z bumpem wersji do N repozytoriów. `LATEST`/`SNAPSHOT` odświeża się samo,
  ale zmienia konfigurację CI wszystkim bez ich wiedzy.

## 2. Extract recipe

- Recipes (od TeamCity 2025.03, następca meta-runnerów) = własne kroki builda.
- Ekstrakcja z istniejącej konfiguracji przez menu Actions → recipe w XML. Wariant YAML pisze się od zera.
- Struktura YAML: `name`, `title`, `version`, `description`, `container`, `inputs`, `steps` —
  pasuje 1:1 do przypadku (kontener + inputs na secrety i parametry).
- Wgranie: Project Settings | Recipes | Upload private recipe. Prywatne recipes **dziedziczą się
  do podprojektów** → Root project = cała organizacja. Edytowalne bez ponownej ekstrakcji.
- YAML w Git → review, historia, wersjonowanie. Wgrywanie na serwer automatyzowalne przez REST API.
- Ograniczenia: brak triggerów i build features; VCS rooty nie są zapisywane w recipe
  (konfiguracja bez odpowiedniego VCS root się wywali); secrety i tak z parametrów projektu.

## Rekomendacja

1. **Sprawdzić wersję serwera.** Jeśli Versioned Settings są wyłączone od 2022, instancja może nie mieć
   2025.03 i recipes nie istnieją. Fallback: meta-runnery (ten sam mechanizm, XML, też dziedziczone).
2. **Hybryda, nie zamiana.** Recipe (YAML w Git) na trzy kroki dockerowe + szablon na root project,
   który używa recipe i dokłada triggery, failure conditions, parametry i secrety.
   Recipe nie zastąpi szablonu.
3. **Perspektywa.** Cała logika siedzi w wersjonowanych obrazach dockerowych przechodzących normalny
   review — warstwa TeamCity to kilkanaście linijek glue. Recipe daje realny zysk (diff, PR, historia
   zmian glue), ale to higiena, nie różnica architektoniczna. Nie warto pod to przebudowywać całości
   ani czekać na włączenie Versioned Settings.
4. **Do adminów.** Versioned Settings zostały wyłączone przez upgrade, nie decyzją architektoniczną.
   Włączenie ich choćby na jednym projekcie pilotażowym otwiera opcję 1 na przyszłość.
