# Archiv

## 2026-10-24 – OpenInsider-Screener stillgelegt

- `openinsider-screener.yml` (früher `.github/workflows/`) und `Openinsider_scrap/output/trades.csv` hierher verschoben.
- Grund: Der Workflow ist seit Juni 2026 bei jedem Lauf fehlgeschlagen (`Openinsider_scrap/main.py` und `requirements.txt` fehlten im Repo), die CSV war veraltet.
- Ersatz: Die Degen Zone lädt Insider-, COT-, Fear-&-Greed- und Research-Daten jetzt als JSON aus dem Repo `cfandl-stack/scheduled-tasks`
  (https://cfandl-stack.github.io/scheduled-tasks/data/). Die Daten werden dort per GitHub Actions erzeugt.
