# [Framework Name] — Research Template

> **Uputa za korištenje:** Ovaj template popuni za svaki framework koji istražuješ.  
> Sva polja su obavezna. Za polja koja ne možeš pronaći, napiši `N/A — nije dokumentirano`.  
> Datum snimke (snapshot) je važan za metodologiju — uvijek ga zabilježi.

---

**Framework name:**  
**Repository:** (GitHub URL ili "N/A — closed source")  
**Official website:**  
**Snapshot date:** YYYY-MM-DD  
**License:** (MIT / Apache-2.0 / AGPL / Commercial / Closed)  
**Language:** (Python / TypeScript / Both / Other)  
**Developer/Organisation:**  
**First public release:** (godina ili "nije poznato")  

---

## Architecture

> Opiši kako framework interno funkcionira. Što su glavne komponente? Kako izgledaju agentski loop ili pipeline? Postoji li više agenata koji surađuju?

[Tekst ovdje]

---

## Perception type

> Kako agent "vidi" web stranicu? Odaberi jedan ili više:
> - **DOM-based** — parsira HTML/DOM stablo, koristi element indekse ili selektore
> - **Vision-based** — šalje screenshot VLM-u, radi na pixel koordinatama
> - **Hybrid** — kombinira DOM i screenshots
> - **Accessibility tree** — koristi a11y stablo umjesto raw HTML
> - **Code generation** — LLM piše kod koji sam pristupa stranici

[Tekst ovdje]

---

## LLM compatibility

> Koji LLM modeli rade s ovim frameworkom? Je li LLM-agnostičan ili vezan uz specifičnog providera?

| Provider | Modeli | Napomena |
|---|---|---|
| | | |

---

## Action interface

> Koji set akcija agent može izvršiti? Kako se zadaje zadatak? Postoji li programatski API?

[Tekst ovdje — nabrojaj ključne akcije i API pozive]

---

## Dependencies

> Koje su glavne ovisnosti? Što treba biti instalirano da bi radio?

- 
- 
- 

---

## Browser control method

> Što framework koristi ispod haube za kontrolu browsera?
> (Playwright / Selenium / CDP direktno / Chrome Extension API / Puppeteer / Vlastito)

[Tekst ovdje]

---

## Strengths

- 
- 
- 

---

## Weaknesses

- 
- 
- 

---

## Benchmark results

> Koji su poznati rezultati na standardnim benchmarkovima? Uvijek navedi koji benchmark, koji datum, koji LLM.

| Benchmark | Score | LLM korišten | Datum | Izvor |
|---|---|---|---|---|
| WebVoyager | | | | |
| WebArena | | | | |
| Mind2Web | | | | |
| Steel Leaderboard | | | | |

---

## Typical use cases

> Za što je ovaj framework idealan? Za što nije prikladan?

**Idealno za:**
- 

**Nije prikladno za:**
- 

---

## Setup complexity

> Koliko je teško postaviti i pokrenuti?  
> Skala: 1 (pip install + api key) → 5 (Docker + baza + cloud account)

**Ocjena:** /5  
**Minimalni setup:** [opis najjednostavnijeg načina pokretanja]

---

## Open source vs. commercial

> Je li open-source? Postoji li cloud varijanta? Koje su cijene?

- Open source: DA / NE
- Self-hostable: DA / NE  
- Cloud/SaaS varijanta: DA / NE (URL)
- Cijena cloud: [ako postoji]

---

## Community & maintenance

> Koliko je aktivna zajednica? Kada je zadnji commit?

- GitHub stars: (aproximativno)
- Last commit: (datum ili "aktivno razvijano")
- Contributors: (broj)
- Discord/Slack: DA / NE

---

## Additional notes for thesis

> Sve što je specifično relevantno za tvoj završni rad — usporedbe, metodološke napomene, zanimljive arhitekturalne detalje, upozorenja na netočne podatke u sekundarnim izvorima.

[Tekst ovdje]

---

*Template verzija: 1.0 | Zadnja izmjena: 2026-03-18*
