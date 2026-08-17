# Potere d'acquisto reale — Comuni e Province d'Italia

Mappa interattiva del **potere d'acquisto reale**: il reddito medio IRPEF corretto per il costo della vita territoriale, per ciascuno dei 7.904 comuni italiani e delle 107 province.

```
potere_acquisto_reale = reddito_medio × [costo_vita_italia_media / costo_vita_stimato]
```

Un reddito nominale alto in una zona molto cara può risultare in un potere d'acquisto reale **inferiore** a un reddito più basso in una zona economica, e viceversa.

**Demo locale**: apri `index.html` con un server che supporti le [Range request](#nota-tecnica-server-locale) (vedi sotto — `python3 -m http.server` **non funziona**).

---

## Le due viste

La mappa offre due viste, con due basi di calcolo diverse per il costo della vita — perché nessuna fonte unica è insieme *reale* e *sufficientemente granulare*:

| | **Vista Comuni** (default) | **Vista Province** |
|---|---|---|
| Costo della vita | Valore immobiliare medio OMI, come differenziale rispetto alla media nazionale | NIC Istat ufficiale (indice prezzi al consumo), dove rilevato |
| Natura del dato | **Stima** (proxy), ma con variazione geografica reale (225–10.767 €/mq) | **Dato diretto**, nessuna stima |
| Copertura | 7.890 comuni | 77 province su 107 (dato reale); le altre 30 restano grigie, senza fallback (27 mai rilevate da Istat, 2 con dato mancante nel mese corrente, 1 provincia abolita) |
| Correzione tipica | Forte: azzera quasi il divario nominale Nord-Sud (~25% → ~2,5%, vedi [Risultati](#risultati-e-verifiche-fatte)) | Minima: il NIC varia solo 2,4 punti in tutta Italia, quindi la vista riflette soprattutto il reddito nominale |

**Perché non un'unica vista "giusta"**: il NIC è un indice di *variazione* dei prezzi nel tempo (ogni città riparte da 100 nel proprio anno base), non di *livello* comparabile fra città — non è quindi un proxy valido del costo della vita assoluto a livello comunale, anche se è l'unico dato Istat "reale" disponibile. L'OMI ha invece una variazione geografica enorme (quasi 50×) ma è comunque una stima indiretta (il costo della vita non è fatto solo di casa). La mappa mostra onestamente entrambe, etichettate per quello che sono.

---

## Fonti dei dati

| Dato | Fonte | Periodo | Copertura |
|---|---|---|---|
| Reddito medio IRPEF | [MEF – Dipartimento delle Finanze](https://www.finanze.gov.it/it/statistiche-fiscali/open-data-comunale-principali-variabili-irpef/) | 2024 | 7.897 comuni |
| Valore immobiliare (OMI) | [Agenzia delle Entrate – Osservatorio Mercato Immobiliare](https://www.agenziaentrate.gov.it/portale/schede/fabbricatiterreni/omi/forniture-dati-omi) | 2° sem. 2025 | 7.511 comuni (richiede login Fisconline/Entratel, non scaricabile via script) |
| NIC (prezzi al consumo) | [Istat SDMX](https://esploradati.istat.it/SDMXWS/rest), dataflow `IT1:167_745` (`DCSP_NIC1B2025`) | Luglio 2026 (base 2025=100) | 77 province + dato nazionale |
| Confini amministrativi | Istat, `Limiti01012022_g` | 1/1/2022 | 7.904 comuni, 107 province |
| Crosswalk codici comune | Istat, [Elenco-comuni-italiani](https://www.istat.it/storage/codici-unita-amministrative/Elenco-comuni-italiani.xlsx) | corrente | flag capoluogo, codici storici |

Licenza dati derivati: **CC BY 4.0**.

---

## Pipeline dati

Stack: **DuckDB** (+ estensione `spatial`) → **GeoParquet** → **tippecanoe** → **PMTiles**. Nessun database persistente, nessun backend: tutta l'elaborazione è offline, l'output finale è statico.

```
scripts/
├── 01_download_mef_redditi.py   Scarica/legge CSV MEF redditi comunali, valida, esporta parquet
├── 02_process_omi.py            Elabora CSV OMI (zone/tipologia → indice comunale), crosswalk Sardegna
├── 02b_process_nic.py           Elabora CSV NIC Istat (SDMX) per gli 80 capoluoghi con rilevazione diretta
├── 03_build_potere_acquisto.py  Join MEF+OMI+confini comunali, calcolo formula, classi quantili → vista Comuni
├── 04_export_pmtiles.py         GeoParquet → GeoJSON → PMTiles (tippecanoe), per comuni e/o province
└── 05_build_province.py         Aggrega MEF per provincia, join NIC reale, calcolo formula → vista Province
```

### Eseguire da zero

```bash
# 1. Redditi MEF (auto-scarica se non già presente in data/raw/mef/)
python3 scripts/01_download_mef_redditi.py --anno 2024

# 2. OMI (richiede download manuale preventivo — vedi nota sotto — in data/raw/omi/QI_<periodo>_VALORI.csv)
python3 scripts/02_process_omi.py

# 2b. NIC Istat (richiede CSV SDMX preventivo in data/raw/istat/nic_<periodo>.csv — vedi nota sotto)
python3 scripts/02b_process_nic.py

# 3. Vista Comuni: join + formula + classi
python3 scripts/03_build_potere_acquisto.py

# 5. Vista Province: aggregazione + NIC reale
python3 scripts/05_build_province.py

# 4. Export tiles (entrambe le viste)
python3 scripts/04_export_pmtiles.py tutti
```

Requisiti: Python 3 con `duckdb` e `pandas`, più i binari `tippecanoe`, `pmtiles`, `ogr2ogr`/`ogrinfo` (GDAL) nel `PATH`.

```bash
pip install duckdb pandas
```

### Note sulle fonti che richiedono download manuale

- **OMI**: il bulk nazionale è disponibile solo nell'area riservata Fisconline/Entratel (login richiesto). Non esiste un mirror pubblico aggiornato — l'unico storico libero ([`ondata/quotazioni-immobiliari-agenzia-entrate`](https://github.com/ondata/quotazioni-immobiliari-agenzia-entrate)) si ferma al 2° semestre 2018. Va scaricato manualmente e posizionato in `data/raw/omi/QI_<periodo>_VALORI.csv` (+ `_ZONE.csv`).
- **NIC**: scaricabile via [Istat SDMX](https://esploradati.istat.it/databrowser) (dataflow `167_745`, base 2025=100) o dal databrowser web, in `data/raw/istat/nic_<periodo>.csv` (più `nic_ref_area_nomi.json`, generabile dal codelist `CL_ITTER107`).

### Punti critici della pipeline (e come sono stati gestiti)

- **Codici comune**: tre fonti diverse (MEF, OMI, confini ISTAT) usano incarnazioni leggermente diverse del codice ISTAT comune (`pro_com`, 6 cifre zero-padded). La Sardegna è il caso peggiore: OMI e NIC usano ancora i codici provincia pre-riforma 2016 per diversi comuni/capoluoghi (es. Villasimius, Olbia, Sassari). Risolto con crosswalk a cascata sulle colonne storiche dell'elenco ISTAT (`Codice Comune numerico con 107/110 Province`).
- **Confini in proiezione errata**: lo shapefile ISTAT `*_WGS84.shp` è in realtà in **EPSG:32632** (UTM32N), non EPSG:4326 nonostante il nome — richiede `ST_Transform` esplicito prima dell'export, altrimenti tippecanoe scarta metà delle geometrie.
- **Micro-comuni rumorosi**: comuni con pochi contribuenti possono avere una media reddituale statisticamente instabile (un solo contribuente ad alto reddito può spostarla sensibilmente). Non filtrati, ma segnalati in nota metodologica.
- **OMI mancante o zero**: comuni con mercato immobiliare fermo (es. aree post-sisma: Accumoli, Preci, Gibellina) hanno OMI legittimamente nullo. Fallback a cascata: media provinciale → media nazionale, con la fonte usata tracciata esplicitamente nel campo `fonte_costo_vita` di ogni comune.
- **NIC come proxy comunale — scartato**: un primo tentativo usava `NIC_stimato(comune) = NIC(capoluogo) × [OMI(comune)/OMI(capoluogo)]^k`. Verificato empiricamente che il NIC varia troppo poco fra capoluoghi (102,3–104,7, spread 2,4 punti) per essere un proxy utile del *livello* di costo della vita — è un indice di *variazione* nel tempo, non di livello comparabile fra città. Abbandonato in favore delle due viste separate.

---

## Frontend

`index.html`, autosufficiente: **MapLibre GL JS** + **PMTiles** (via CDN, con Subresource Integrity), basemap **CartoDB Positron** (chiara), nessuna build step.

- Toggle Comuni/Province in alto a sinistra
- Choropleth a 5 classi quantili, palette diverging rosso→verde (ColorBrewer RdYlGn)
- Popup al click con dettaglio per comune/provincia, inclusa la fonte del dato costo-vita
- Banner di avviso permanente nella vista Province (correzione minima)
- Pannello "Note metodologiche" completo (formula, fonti, limiti, licenza)

### Nota tecnica: server locale

`pmtiles.js` richiede il supporto alle **HTTP Range request** (risposta `206 Partial Content`) per leggere header/directory/tile del file `.pmtiles`. `python3 -m http.server` **non le supporta** (risponde sempre `200` con l'intero file) e la mappa non caricherà alcun dato, senza errori evidenti in console. Usare invece:

```bash
npx http-server -p 8080 --cors
```

---

## Struttura del repository

```
data/
├── raw/            Dati grezzi scaricati (MEF, OMI, Istat NIC, confini, crosswalk comuni)
└── processed/      Output intermedi e finali (parquet, geoparquet)
dist/geo/           PMTiles pubblicati, serviti dal frontend
docs/               Piano di progetto originale
scripts/            Pipeline DuckDB, eseguibile in sequenza (01 → 02/02b → 03/05 → 04)
index.html          Frontend, nessuna dipendenza da build
```

---

## Risultati e verifiche fatte

Aggregando la vista Comuni per macroarea (dati 2024/2025):

| Area | Reddito medio | Potere d'acquisto reale | Correzione |
|---|---|---|---|
| Nord | 24.550 € | 26.190 € | +6,7% |
| Centro | 22.024 € | 25.978 € | +18% |
| Sud e Isole | 19.686 € | 25.555 € | +29,8% |

Il divario nominale Nord-Sud (~25%) si riduce a **~2,5%** dopo la correzione per costo della vita — coerente con la letteratura economica (Banca d'Italia, SVIMEZ) sul restringimento del divario reale Nord-Sud una volta corretto per il costo della vita locale, pur senza ribaltarsi completamente.

---

## Timeline e stato

Piano originale in `docs/piano-mappa-potere-acquisto-comuni.md`. Stato attuale: pipeline dati e frontend completi e funzionanti (punti 1–4 del piano). Da fare: automazione GitHub Actions per l'aggiornamento periodico (punto 3.4 del piano — dati IRPEF/OMI annuali/semestrali, non urgente), pubblicazione GitHub Pages.
