# Germany Data-Jobs Market Pulse: HR-Analytics-Dashboard

<p align="center">
  <img src="assets/Germany Data-Jobs Market Dashboard.png" alt="German Job Market HR dashboard hero" width="900">
</p>

**TL;DR** — Live-Stellenanzeigen der **Bundesagentur für Arbeit** abrufen, **standardisieren und anreichern** (Titel, Arbeitgeber, **PLZ→Stadt/Region**), und ein **Power BI**-Dashboard liefern, das zeigt, **wo und was** eingestellt werden sollte: **Nachfrage-Hotspots**, **aktive Arbeitgeber**, **Freshness der Anzeigen** und **30‑Tage‑Wachstum** nach Jobtitel.

**Tech**: Python (Pandas, Requests) • Power Query (ETL) • Power BI / DAX  
**Daten**: Bundesagentur für Arbeit (BA) Jobsuche-API — öffentliche Stellenanzeigen (Dank an BA)  
**Repo-Struktur**: [`notebooks/`](notebooks) • [`powerbi/`](powerbi) • [`assets/`](assets)

---

## Inhaltsverzeichnis
- [Hintergrund](#hintergrund)
- [Datensatz](#datensatz)
- [Methodik](#methodik)
- [Wichtigste Erkenntnisse](#wichtigste-erkenntnisse)
- [So reproduzierst du es](#so-reproduzierst-du-es)
- [Artefakte (Deliverables)](#artefakte-deliverables)
- [Business-Empfehlungen](#business-empfehlungen)

---

## Hintergrund
Recruiting-Teams brauchen einen **wöchentlichen Puls** des Marktes: Welche **Jobtitel** sind heiß, welche **Arbeitgeber** am aktivsten und welche **Regionen (PLZ/Stadt)** steigen an. Dieses Projekt verwandelt Rohanzeigen in **umsetzbare Recruiting-Moves** — wo sich Outreach lohnt, welche Rollen zu pushen sind und **wie** man das Timing über **Freshness** (Anteil Anzeigen < 7 Tage) steuert.

---
### Geschäftsfrage
Wo konzentriert sich die Nachfrage nach **datenrelevanten Rollen** in Bamberg und Umgebung (**100 km Radius**) — über Arbeitgeber, Jobtitel und Freshness — und **was** sollte HR priorisieren, um Pipelines effizient zu füllen?

---

## Datensatz
- **Quelle**: Bundesagentur für Arbeit (BA) – **Jobsuche/Jobbörse** API (öffentlich).  
- **Entitäten (aus Modell & Queries)**:  
  - `bamberg_jobs_enriched`: Arbeitgeber, Jobtitel, PLZ, Stadt, Region, Veröffentlichungsdatum, Modifikations-Timestamp, abgeleitete KPIs.  
  - **Datumsdimension**: Date, Week, WeekStart, Year, YearWeek.  
- **Zeitraum**: Backfill ab **2017‑08‑28** bis zum aktuellen Snapshot.  
- **Abdeckung (aktueller Snapshot)**: ~**2.803** eindeutige Arbeitgeber, ~**9.993** aktive Anzeigen.  
- **Referenz**: **PLZ→Stadt/Region**-Mappingtabelle zur Anreicherung.

---

## Methodik
**Python (Extraktion)**  
- Verbindung zur BA‑Jobbörse‑API (`/pc/v4/jobs`) mit Demokey `jobboerse-jobsuche`.  
- Abfrage der Stellenanzeigen für **Bamberg + 100 km** (Vollzeit & Teilzeit; `arbeitszeit = vz;tz`).  
- Pagination implementiert bis **100 Seiten** mit kurzem Delay (`time.sleep(0.5)`).  
- Antworten zu **einem JSON** kombiniert und verschachtelte Objekte per `pandas.json_normalize` **geflacht**.
  
  **Datenüberblick & Validierung**  
  - Struktur geprüft (`head`, `shape`, `info`, `describe`), Feldkonsistenz bestätigt.  
  - Dubletten entfernt (`drop_duplicates()`), Missing Values geprüft (`isna().sum()`).  
  - Kernfelder identifiziert: Jobtitel, Beruf, Arbeitgeber, Stadt, PLZ, Region, relevante Timestamps.
    
  **PLZ‑Vervollständigung**  
  - Städte ohne PLZ erkannt (`arbeitsort.plz.isna()`).  
  - **City→PLZ‑Mapping** aus gültigen Zeilen gebaut (häufigste PLZ je Stadt).  
  - Fehlende PLZ gefüllt und Abdeckung nach Imputation verifiziert.
    
  **Filter & Textstandardisierung**  
  - Filter auf vollständige Zeilen (`titel`, `arbeitgeber`, `arbeitsort.ort`, `arbeitsort.region`, `arbeitsort.plz`).  
  - **`clean_title()`** zur Normalisierung von Jobtiteln:  
    - Kleinschreibung, Trimmen.  
    - Entfernen von Gender‑Tags (`m/w/d`, `gn`, `div`).  
    - Interpunktion normalisieren, Umlaute (`äöüß`) erhalten.  
    - Mehrfach‑Leerzeichen kollabieren.
    
  **Feature Engineering**  
  - `aktuelleVeroeffentlichungsdatum`, `modifikationsTimestamp`, `eintrittsdatum` als `datetime` geparst.  
  - `weekday` aus dem Veröffentlichungsdatum extrahiert.  
  - **Posting Age** als Differenz zwischen heute und Veröffentlichungsdatum berechnet (`posting_age_days`).  
  - **Age Buckets** gebildet (`<7`, `7–30`, `30–90`, `>90` Tage) zur Freshness‑Analyse.
    
  **EDA & Visualisierung**  
  - **Top‑Städte**, **Jobtitel**, **Berufe** und **Arbeitgeber** nach Anzeigenzahl gerankt.  
  - **Top‑10‑Arbeitgeber‑Konzentration** (Anteil an allen Anzeigen) berechnet.  
  - Visualisiert:  
    - **Verteilung Posting Age** (Histogramme, gruppierte Balken).  
    - **Anzeigen nach Wochentag** (Publikationszyklen).  
    - **Korrelations‑Heatmap** zwischen numerischen Features (Posting Age, PLZ‑Gruppierung, datumsbasierte Attribute) für Ort↔Recency‑Beziehungen.  
  - Summary‑Tabellen mit Frequenzen und Anteilen.
    
  **Gesamtfazit**  
  - **Kernmuster** zu regionalen Hiring‑Pattern und Dynamik der Stellenanzeigen herausgearbeitet.
    
  **Outputs**  
  - Geflachte Daten als UTF‑8‑CSV exportiert (`bamberg_jobs_100km.csv`).  
  - In Power BI wurde die **Roh‑CSV erneut geladen** und **Clean‑Steps** interaktiv nachvollzogen.

**Power Query**
- Header promotet, Typen gesetzt.  
- Trim/Clean von Texten; Entfernen der Gender‑Marker (m/w/d, gn, gn*).  
- Arbeitgebernamen standardisiert (GmbH, AG, SE, UG).  
- **Staging‑Spalte „Titel (Anzeige)“**: vereinheitlicht `titel` + `beruf`, behandelt Nulls/Leers, normalisiert die Groß/Kleinschreibung.  
- Duplikate entfernt.  
- **PLZ→Stadt/Region** gemerged.  
- Abgeleitete Spalten: Median‑Posting‑Age, **Top Job Title**, **Top Employer**, **Top‑3‑Titles Share %**.  
- Gruppierung nach Woche, Arbeitgeber, Region.

**Modellierung (Power BI)**  
- **Date‑Dimension** per `CALENDAR()` aus min/max `Postingdatum` gebaut.  
- Spalten: `Week`, `WeekStart`, `Year`, `YearWeek` für Time Intelligence.  
- Beziehungen zwischen Date und Posting‑Tabelle angelegt.

- **DAX‑Maße**:  
  - **Stellen (gesamt)** – Summe Anzeigen.  
  - **Arbeitgeber Einz.** – eindeutige Arbeitgeber.  
  - **Wachstum 30T %** – 30‑Tage‑Rollrate.  
  - **Median Anzeigealter (Tage)** – Median der Posting Age.  
  - **Anteil frisch <7 Tage %** – Anteil sehr frischer Anzeigen.  
  - Unterstützend: `Jobs letzte 7 Tage`, `Jobs letzte 7 Tage (Prev)`, `4W gleitender Durchschnitt`, **Top‑3‑Titles‑Anteil**.

**Visualisierung (Power BI Seiten)**  
- **Zusammenfassung** — KPIs, Wachstumstrends, Regionsanteile.  
- **Nachfrage nach Standort** — Karte nach PLZ/Stadt + Titel‑Slicer.  
- **Arbeitgeberlandschaft** — Drilldowns nach Arbeitgeber + Aktivität je Region.  
- **Jobtitel‑Markt** — Anzeigen/Woche, Freshness, gleitende Durchschnitte.

<div style="display: flex; overflow-x: auto; gap: 10px; padding: 10px;">
  <img src="assets/Germany Data-Jobs Market Dashboard.png" alt="Germany Data-Jobs Market Dashboard-Zusammenfassung" width="350">
  <img src="assets/Nachfrage nach Standort.png" alt="Nachfrage nach Standort" width="350">
  <img src="assets/Arbeitgeberlandschaft.png" alt="Arbeitgeberlandschaft" width="350">
  <img src="assets/Jobtitel-Markt.png" alt="Jobtitel-Markt" width="350">
</div>

---

## Wichtigste Erkenntnisse

### Aus Python (Jupyter Notebook)

#### Datenumfang
- Rund **10.000** Live‑Stellenanzeigen über die BA‑API erfasst.  
- Geografischer Fokus: **~100 km** um Bamberg, v. a. Franken & Nordbayern.

#### Nachfrage nach Standort
- **Nürnberg (Mittelfranken)** ist der Haupt‑Hotspot (~**1.300** Anzeigen), gefolgt von **Bamberg (~1.100)** und **Erlangen (~850)**.  
- Sekundärcluster: **Forchheim, Bayreuth, Würzburg, Fürth, Höchstadt a. d. Aisch, Hirschaid, Coburg**.  
- Deutung: Nachfrage konzentriert sich entlang des fränkischen **Industrie‑ & Logistik‑Korridors**.

#### Gefragte Rollen
- Nach Beruf: **Verkäufer/in, Gabelstaplerfahrer/in, Maschinen‑ & Anlagenführer/in (o. Schwerpunkt), Helfer/in – Reinigung, Industriemechaniker/in, Fachkraft – Lagerlogistik, Mechatroniker/in**.  
- Nach Jobtitel: **Produktionsmitarbeiter (m/w/d), Staplerfahrer (m/w/d), Maschinenbediener (m/w/d), Produktionshelfer (m/w/d), Lagermitarbeiter (m/w/d)**.  
- Deutung: Markt ist stark **operations‑getrieben** (Produktion, Lager, technische Gewerke). IT/Admin kleinerer Anteil.

#### Posting‑Takt & Freshness
- Wöchentliche Anzeigen **Peak Mitte September 2025** (~**4.300** in der Woche).  
- Wochentage: **Mo–Mi** am aktivsten; **Wochenende** sehr wenig.  
- **Posting Age**: front‑loaded (0–6 Tage) mit langem Tail **>90 Tage** (lange offen/republiziert).  
- Empfohlene KPIs: **Median Posting Age**, **Anteil <7 Tage**.

#### Regional‑ & Rollenmix
- Mehrheit der Anzeigen in **Bayern**, gefolgt von **Thüringen** und **Baden‑Württemberg**.  
- In Bayern dominieren **Produktion & Lager**, konsistent mit der Industriestruktur.

#### Korrelationen
- Keine starken linearen Zusammenhänge.  
- Schwach negative Korrelation (**~ −0,35**) zwischen Posting Age und Entfernung vom Suchzentrum — vermutlich Radius‑Erweiterung; **nicht entscheidungsrelevant**.

### Aus dem Power‑BI‑Dashboard

#### Executive KPIs (Summary)
- **Eindeutige Arbeitgeber**: **2.688**  
- **Aktive Anzeigen gesamt**: **9.964**  
- **30‑Tage‑Wachstum**: **+47,15 %**  
- **Median Posting Age**: **19 Tage**  

#### Top‑Jobtitel
- **Industriemechaniker** am häufigsten (~**1.049**).  
- Weitere starke Rollen: **Maschinen‑ & Anlagenführer, Mechatroniker, Helfer Reinigung/Lagerlogistik**.

#### Standort‑Highlights
- **Top‑Standort**: **Nürnberg, Mittelfranken**.  
- **Top‑PLZ**: **90312**.  
- Karte zeigt dichte Konzentration in **Nordbayern/Franken**.

#### Arbeitgeberlandschaft
- Markt fragmentiert, **Personaldienstleister** führen.  
- **Top‑Arbeitgeber (letzte 7 Tage)**: AlphaConsult, Unique Personalservice, Papp Personal, Randstad Deutschland, ABSOLUT Personalservice, König Fachpersonal, FERCHAU, DIE JOBMACHER.  
- Regional dominiert **Bayern**, gefolgt von **Thüringen, Baden‑Württemberg, Sachsen, Hessen**.

#### Freshness & Aktivität
- **Hohe Freshness** in den meisten Jobtiteln; vielfach **>95 %** unter 7 Tagen.  
- Deutet auf **schnelle Turnover** und aktive Recruiting‑Pipelines.

#### Wachstumstrends
- **30‑Tage‑Wachstum** nach Titel zeigt stetige Zunahmen in industriell/technischen Rollen.  
- **+47 % monatlich** signalisiert starke Einstellungsdynamik.

### Zusammenfassung
Der regionale Arbeitsmarkt um **Franken & Nordbayern** wird von **Industrie, Logistik, Produktion** getragen. **Personaldienstleister** spielen eine Schlüsselrolle. **Hohe Freshness** und starkes **30‑Tage‑Wachstum** zeigen aktive Zyklen — besonders bei industriellen/mechanischen Rollen. Fokus für HR: **Mo–Mi sourcen**, **Median Posting Age** und **<7‑Tage‑Anteil** beobachten.

---

## So reproduzierst du es
1. Repo klonen.  
2. **Python‑Ingestion** (optional für Refresh):  
   - `notebooks/jobs_market_pulse.ipynb` öffnen.  
   - API‑Extract ausführen → speichert `data/jobs_raw.csv` → `data/jobs_clean.csv`.  
3. **Power BI öffnen**:  
   - `powerbi/germany_jobs_market_pulse.pbix` laden.  
   - Datenquellenpfad prüfen → **Refresh**.  
4. Dashboards erkunden.

---

## Artefakte (Deliverables)

- **Notebook:** [Germany data-jobs market pulse (HR analytics angle).ipynb](./Germany%20data-jobs%20market%20pulse%20(HR%20analytics%20angle).ipynb)
- **Dashboard:** [Germany data-jobs market Dashboard.pbix](./Germany%20data-jobs%20market%20Dashboard.pbix)
- **Assets (Diagramme & Visualisierungen):**
  - [Germany Data-Jobs Market Dashboard.png](./assets/Germany%20Data-Jobs%20Market%20Dashboard.png)
  - [Arbeitgeberlandschaft.png](./assets/Arbeitgeberlandschaft.png)
  - [Jobtitel-Markt.png](./assets/Jobtitel-Markt.png)
  - [Nachfrage nach Standort.png](./assets/Nachfrage%20nach%20Standort.png)
- **Datensatz:** [Bamberg_jobs_100km.csv](./Bamberg_jobs_100km.csv)


---

## Business-Empfehlungen

### 1) Auf Kernregionen fokussieren
Recruiting & Planung in **Nürnberg, Bamberg, Erlangen** priorisieren (höchste Volumina).  
Selektiv in **Bayreuth, Würzburg, Fürth** erweitern, um zusätzliche Talentpools zu erschließen.

### 2) Gefragte Rollen targeten
Pipelines für **Industriemechaniker, Mechatroniker, Lagerlogistik, Gabelstaplerfahrer, Produktionsmitarbeiter** stärken — sie dominieren die Nachfrage.  
**Trainings/Apprenticeships** erwägen, um Engpässe in Produktion & Lager zu adressieren.

### 3) Timing optimieren
Stellen & Outreach auf **Montag–Mittwoch** legen (höchste Aktivität).  
**Median Posting Age** und **Anteil <7 Tage** tracken, um in einem schnellen Markt wettbewerbsfähig zu bleiben.

### 4) Personaldienstleister nutzen
Strategische Partnerschaften mit großen **Staffing‑Firms** (z. B. AlphaConsult, Randstad, Unique Personalservice).  
Für **schnellen Zugang** zu Arbeitskräften in Logistik, Produktion, Technik nutzen.

### 5) Wachstums‑KPIs monitoren
**30‑Tage‑Wachstumsrate (+47,15 %)** und **Median Posting Age (19 Tage)** als **Leading Indicators** verfolgen.  
Mit Power BI **rollen‑ & regionenspezifisch** monitoren, um Workforce‑Planung agil zu steuern.
