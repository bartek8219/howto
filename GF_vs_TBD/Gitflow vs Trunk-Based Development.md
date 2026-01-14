## Gitflow vs Trunk-Based Development – porównanie

### Kontekst: wiele repozytoriów, małe zespoły

W waszej konfiguracji (kilkadziesiąt aplikacji backendowych, osobne repo w BitBucket, 1-2 opiekunów na aplikację) oba podejścia będą działać inaczej niż w klasycznych scenariuszach.[^1][^2][^3]

***

## Gitflow – obecne podejście

### Jak wygląda u was:

- **Długotrwałe branche**: `main`, `develop`, branche `feature/*`, `hotfix/*`, `release/*`[^4][^5]
- **Proces**: deweloper tworzy branch feature, pracuje dni/tygodnie, merge do `develop`, następnie przez `release/*` do `main`[^6][^7]
- **Izolacja**: każdy feature ma osobny branch, zmiany są integrowane dopiero po ukończeniu[^8][^4]


### Mocne strony w waszym przypadku:

- **Struktura i przewidywalność**: jasne zasady gdzie i kiedy mergować, łatwe do wytłumaczenia nowym członkom[^5][^4]
- **Równoległy development**: możliwość pracy nad wieloma featurami bez wzajemnego wpływu[^9][^4]
- **Zarządzanie wersjami**: dedykowane branche `release/*` ułatwiają przygotowanie wersji i testy przed produkcją[^4][^6]
- **Hotfixy**: szybkie poprawki bezpośrednio z `main` bez zakłócania trwającego developmentu[^9]
- **Dobre dla compliance**: pełna historia kto, co i kiedy zmergował – ważne w środowiskach regulowanych[^5]


### Słabe strony:

- **Konflikty przy merge**: długotrwałe branche oddalają się od `develop`, co powoduje bolesne konflikty[^10][^7]
- **Opóźnione feedback**: zmiany integrowane późno, błędy wykrywane z dużym opóźnieniem[^8][^10]
- **Overhead**: zarządzanie wieloma typami branchy, ceremony przy release, złożoność w małych zespołach[^4][^5]
- **Problematyczne w multi-repo**: koordynacja zmian wymagających modyfikacji w kilku repo jest trudna, brak narzędzi do synchronizacji[^2][^3][^1]

***

## Trunk-Based Development – alternatywa

### Jak by wyglądał:

- **Jedna główna gałąź**: `main` (trunk) – zawsze gotowa do wdrożenia[^6][^4]
- **Krótkotrwałe branche**: feature branche żyją maksymalnie 1-2 dni, a najlepiej kilka godzin[^11][^12]
- **Częste merge**: commit do trunk przynajmniej raz dziennie[^12][^13]
- **Feature flags**: niedokończone funkcje ukrywane za flagami, kod w trunk ale nieaktywny na produkcji[^14][^15][^16]


### Mocne strony:

- **Mniej konfliktów**: częste małe merge znacząco redukują złożoność konfliktów[^17][^10][^6]
- **Szybkie feedback**: błędy wykrywane wcześniej, CI/CD pełni kluczową rolę[^18][^19][^8]
- **Prostota**: mniej branchy, mniej zasad, szybsze onboardowanie[^6][^4]
- **Continuous Delivery**: naturalnie wspiera wdrożenia kilka razy dziennie[^13][^14][^6]
- **Lepsze dla multi-repo**: każde repo niezależnie, brak potrzeby koordynacji branchy między repozytoriami[^17]


### Słabe strony i problemy, na które możecie natrafić:

#### 1. **Presja na infrastrukturę testową**

- TBD wymaga **komprehensywnych testów automatycznych** (unit, integration, e2e) uruchamianych przy każdym commicie[^19][^20][^18][^8]
- Bez tego ryzyko złamania build jest wysokie[^21][^11][^8]
- **Problem**: musicie zainwestować w build pipeline i pokrycie testami – to nie jest opcjonalne[^20][^18][^19]


#### 2. **Brak izolacji zmian**

- Wszyscy pracują na tym samym branchu, zmiany są natychmiast widoczne dla całego zespołu[^22][^8]
- Może prowadzić do zakłóceń, niezamierzonych efektów ubocznych i "łamania" build[^11][^8]
- **Problem**: wymaga wysokiej dyscypliny, komunikacji i kultury zespołowej – trudne w większych lub mniej doświadczonych zespołach[^8][^11]


#### 3. **Code churn i trudność śledzenia zmian**

- Stały napływ małych commitów utrudnia zrozumienie aktualnego stanu kodu i wpływu ostatnich zmian[^8]
- **Problem**: większe ryzyko pomyłek, szczególnie jeśli review są zbyt szybkie lub powierzchowne[^11]


#### 4. **Zarządzanie release i środowiskami**

- W waszym przypadku (komunikacja z BPM, prawdopodobnie środowiska dev/test/prod) potrzebujecie jasnej strategii deploymentu[^23][^24]
- **Problem**: jeśli macie długotrwałe branche per środowisko – to nie jest trunk-based[^24][^23]
- Rozwiązanie: osobne repo z definicjami wersji artefaktów per środowisko + pipeline triggerowany zmianami w tym repo[^23]


#### 5. **Feature flags – nowy layer złożoności**

- Niedokończone funkcje ukrywane za feature flags (np. if-y w kodzie)[^15][^16][^14]
- **Problem**: jeśli flagi nie są szybko usuwane, generują **technical debt** i komplikują kod[^14][^15]
- Wymaga narzędzi do zarządzania flagami (np. Unleash, LaunchDarkly) i dyscypliny w ich usuwaniu[^15][^14]


#### 6. **Code review – zmiana procesu**

- W Gitflow: review po ukończeniu feature (jeden duży PR)[^25]
- W TBD: review po pierwszym commicie, często przed ukończeniem feature (wiele małych PR)[^26][^25]
- **Problem**: review muszą być przetwarzane **bardzo szybko** (minuty, nie godziny), inaczej blokują flow i zwiększają cycle time[^27][^26]


#### 7. **Wymaga doświadczonego zespołu**

- TBD wymaga umiejętności dzielenia pracy na małe batche, pisania testów, zarządzania flagami[^13][^11][^8]
- **Problem**: dla osób przyzwyczajonych do długotrwałych branchy to znacząca zmiana mindset[^13][^11]


#### 8. **Ryzyko w środowisku multi-repo**

- Choć TBD lepiej się skaluje w multi-repo niż Gitflow, nadal macie ryzyko niesynchronizowanych zmian między aplikacjami[^3][^2]
- **Problem**: jeśli zmiana wymaga modyfikacji w kilku repo, musicie mieć proces synchronizacji (np. wersjonowanie API, semantic versioning, testy kontraktowe)[^28]

***

## Kiedy rozważyć TBD w waszym przypadku

**Dobrze pasuje jeśli**:[^4][^6][^13]

- Chcecie praktykować CI/CD i wdrażać często
- Zespoły są wystarczająco doświadczone lub gotowe na zmianę
- Macie lub jesteście gotowi zbudować solidną infrastrukturę testową
- Szybkie feedback i redukcja merge conflicts to priorytet

**Zostańcie przy Gitflow jeśli**:[^5][^6][^4]

- Macie strukturalne cykle release (np. raz na miesiąc/kwartał)
- Potrzebujecie zarządzać wieloma wersjami równocześnie
- Brakuje wam zasobów/czasu na zbudowanie infrastruktury testowej
- Zespół nie jest gotowy na zmianę kultury pracy

***

## Rekomendacja na początek

**Nie przeskakujcie od razu całkowicie**. Rozważcie:

1. **Pilot w 1-2 repozytoriach**: wybierzcie aplikacje z dobrym pokryciem testami i doświadczonymi opiekunami[^12]
2. **Zainwestujcie w testy i CI**: bez tego TBD nie ma szans[^18][^19][^20]
3. **Feature flags jako pierwszy krok**: wprowadźcie je jeszcze w Gitflow, żeby zespół się przyzwyczaił[^14][^15]
4. **Skróćcie życie feature branches**: zamiast tygodni → dni, bez pełnego przejścia na TBD[^12][^13]

***

**Kluczowe pytania na spotkanie**:

- Jak wygląda obecne pokrycie testami automatycznymi?
- Ile czasu zespół traci na rozwiązywanie konfliktów przy merge?
- Jak często wdrażacie na produkcję i czy chcecie to przyspieszyć?
- Czy zespół jest gotowy na zmianę sposobu pracy i naukę nowych praktyk?
<span style="display:none">[^29][^30][^31][^32][^33][^34][^35][^36][^37][^38][^39][^40][^41][^42][^43][^44][^45][^46][^47][^48][^49][^50][^51][^52][^53]</span>

<div align="center">⁂</div>

[^1]: https://stackoverflow.com/questions/16754597/using-git-flow-with-multiple-git-repositories-per-project-app

[^2]: https://gist.github.com/Riebart/02a69371c100385f453c9d8eb01dd585

[^3]: https://georgestocker.com/2020/03/04/please-stop-recommending-git-flow/

[^4]: https://graphite.dev/guides/trunk-vs-gitflow

[^5]: https://get.assembla.com/blog/trunk-based-development-vs-git-flow/

[^6]: https://www.appsilon.com/post/comparison-of-gitflow-and-trunk-based-development

[^7]: https://www.atlassian.com/continuous-delivery/continuous-integration/trunk-based-development

[^8]: https://docs.aws.amazon.com/prescriptive-guidance/latest/choosing-git-branch-approach/advantages-and-disadvantages-of-the-trunk-strategy.html

[^9]: https://www.glukhov.org/post/2025/06/gitflow-steps-and-alternatives/

[^10]: https://www.bagile.co.uk/branching-killing-delivery-trunk-based-development/

[^11]: https://www.featbit.co/articles2025/trunk-based-development-frustrations-solutions/

[^12]: https://codilime.com/blog/trunk-based-development/

[^13]: https://dora.dev/capabilities/trunk-based-development/

[^14]: https://docs.getunleash.io/feature-flag-tutorials/use-cases/trunk-based-development

[^15]: https://www.featbit.co/articles2025/trunk-based-development-feature-flags-2025/

[^16]: https://martinfowler.com/articles/feature-toggles.html

[^17]: https://www.aviator.co/blog/trunk-based-development-in-microservices/

[^18]: https://www.harness.io/blog/a-complete-guide-to-trunk-based-development

[^19]: https://dev.to/hsmall/how-to-implement-trunk-based-development-a-practical-guide-56e7

[^20]: https://www.aviator.co/blog/managing-continuous-delivery-with-trunk-based-development/

[^21]: https://www.baeldung.com/ops/vcs-trunk-based-development

[^22]: https://www.aviator.co/blog/trunk-based-development/

[^23]: https://www.reddit.com/r/devops/comments/1gilg78/how_to_handle_releases_for_different_environments/

[^24]: https://codefresh.io/blog/stop-using-branches-deploying-different-gitops-environments/

[^25]: https://team-coder.com/code-reviews-in-trunk-based-development/

[^26]: https://trunkbaseddevelopment.com/continuous-review/

[^27]: https://stackoverflow.com/questions/62952712/code-reviews-with-trunk-based-development

[^28]: https://www.reddit.com/r/devops/comments/wpdku5/how_do_you_manage_microservices_api_versions_and/

[^29]: https://www.datacamp.com/tutorial/git-branching-strategy-guide

[^30]: https://learnautomatedtesting.com/course/git-for-test-engineers-from-basics-to-team-collaboration/module-9/lesson-4-multi-repository-workflow-managing-tests-across-microservices/

[^31]: https://www.toptal.com/software/trunk-based-development-git-flow

[^32]: https://softwareskill.pl/trunk-based-development-ci-cd

[^33]: https://justjoin.it/blog/escape-from-merge-hell-why-i-prefer-trunk-based-development-over-feature-branching-and-gitflow

[^34]: https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow

[^35]: https://www.reddit.com/r/programming/comments/17xom10/why_i_prefer_trunkbased_development/

[^36]: https://stackoverflow.com/questions/42135533/what-are-the-pros-and-cons-of-using-a-trunk-based-vs-feature-based-workflow-in-g

[^37]: https://www.aviator.co/blog/monorepo-a-hands-on-guide-for-managing-repositories-and-microservices/

[^38]: https://www.reddit.com/r/devops/comments/1da01vm/dev_testing_with_trunk_based_development/

[^39]: https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/implement-a-gitflow-branching-strategy-for-multi-account-devops-environments.html

[^40]: https://www.freecodecamp.org/news/what-is-trunk-based-development/

[^41]: https://trunkbaseddevelopment.com

[^42]: https://dev.to/okoye_ndidiamaka_5e3b7d30/advanced-git-workflows-and-best-practices-in-version-control-systems-49dk

[^43]: https://www.architecture-weekly.com/p/dont-oversell-ideas-trunk-based-development

[^44]: https://dev.to/victoria/git-branching-for-small-teams-2n64

[^45]: https://www.reddit.com/r/git/comments/x7yhru/does_this_look_like_a_ok_git_flow_for_small_team/

[^46]: https://trunkbaseddevelopment.com/feature-flags/

[^47]: https://stackoverflow.com/questions/25681521/which-git-flow-is-the-best-for-a-small-team-where-all-developers-work-on-all-the

[^48]: https://news.ycombinator.com/item?id=12941997

[^49]: https://bettersoftwaredesign.pl/podcast/o-dostarczaniu-kodu-na-produkcje-z-uzyciem-feature-toggles-z-mateuszem-kwasniewskim/

[^50]: https://developer.harness.io/docs/feature-flags/get-started/trunk-based-development

[^51]: https://www.abtasty.com/blog/git-branching-strategies/

[^52]: https://www.youtube.com/watch?v=0jM2c16DjuA

[^53]: https://adevait.com/software/creating-branching-strategy

