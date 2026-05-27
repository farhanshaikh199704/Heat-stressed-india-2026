# Heat-Stressed India: The Seen and the Unseen

**Farhan Shaikh · Data Journalism, Hertie School · May 2026**

🔗 **[Read the full article](https://data-journalism-26.github.io/final-project-farhan_final/)**

---

## What this project is about

This data journalism piece investigates the relationship between extreme heat stress and adverse maternal and child health outcomes across India's districts. It brings together two independent research studies — one mapping district-level heat risk, the other establishing the epidemiological link between heat exposure and pregnancy outcomes — to determine hotspot regions at risk, the nature of consequences for mothers and their children, and, lastly, the temporal distribution of stress across the three trimesters.

The central argument: India's heat early warning systems are calibrated to daytime peak temperatures. The atmospheric phenomenon rising fastest — sustained humid nights — is the thing those systems measure least. And it is precisely this form of heat that accumulates across pregnancy trimesters, compounding risk for low birthweight and preterm birth in districts where women are already most vulnerable.


---

## Data sources

| Dataset | Source | Coverage | Use in project |
|---|---|---|---|
| Composite Heat Risk Index | CEEW 2025 — *How Extreme Heat is Impacting India* | 734 districts, 35 indicators, 1982–2022 | District HRI scores, hazard, exposure, vulnerability |
| NFHS-5 District Factsheets | pratapvardhan/NFHS-5 (GitHub) | 340 districts, 21 states, 2019–21 | Underweight prevalence, antenatal coverage, anaemia, institutional births, postnatal care |
| India District Boundaries | datta07/INDIAN-SHAPEFILES | 820 districts, 2011 Census | Choropleth map base layer |
| IMDAA Reanalysis | NCMRWF, Ministry of Earth Sciences | 12-km resolution, 1982–2022 | Underlying climate data for CEEW HRI |

---

## Key findings

- **57% of Indian districts** — home to 76% of India's population — are at high to very high composite heat risk (CEEW 2025)
- **Over 70% of districts** experienced at least five additional very warm nights per summer in the last decade compared to the 1982–2011 baseline — rising faster than hot days
- Each additional consecutive sweltering day in the **first trimester** raises preterm birth odds by **3.6%** (aOR 1.036, 95% CI: 1.009–1.064)
- Each additional day in the **second trimester** raises low birthweight odds by **1%** (aOR 1.010, 95% CI: 1.002–1.018)
- A **December conception** — common in northern India's post-monsoon peak — places the second trimester almost entirely inside March through May, the heart of the heat season
- **16 double-jeopardy districts** were identified across 18 states where HRI ≥ 0.80, child underweight prevalence ≥ 35%, and first-trimester antenatal coverage ≤ 65% simultaneously
- **Nandurbar, Maharashtra**: HRI 0.83 · underweight 57.2% · first-trimester ANC 51%
- **Bharuch, Gujarat**: HRI 0.92 · underweight 45.5% · first-trimester ANC 63%

---

## Repository structure

```
final-project-farhan_final/
│
├── index.html                        # Main article — open this in a browser
├── visual1-map.html                  # Interactive leaflet choropleth map
├── visual2-calendar.html             # Trimester calendar widget
├── visual3-scatter.html              # Interactive scatter plot
│
├── data/
│   ├── ceew_district_hri.csv         # Extracted HRI scores for 724 districts
│   └── scatter_data.csv              # Merged dataset: 277 districts, HRI + NFHS-5
│
├── analysis/
│   └── india_heat_maternal_analysis.R  # Full reproducible analysis script
│
└── Literature/                       # Source papers
    ├── ceew-how-extreme-heat-is-impacting-india.pdf
    └── Vincent_2026_Environ_Res_Commun_8_041009.pdf
```

---

## How to reproduce the analysis

### Requirements

**Python** (for data extraction):
```
pdfplumber
pandas
requests
```

**R** (for map generation):
```r
sf, dplyr, leaflet, htmlwidgets, here
```

### Step 1 — Extract CEEW district HRI data

The CEEW annex PDF is required. Download it from:
```
https://www.ceew.in/sites/default/files/annexure-1-state-wise-handbooks-of-heat-risk.pdf
```

Place it in the `data/` folder, then run the extraction script in the Colab notebook or locally:

```python
import pdfplumber
import pandas as pd

rows = []
with pdfplumber.open("data/annexure-1-state-wise-handbooks-of-heat-risk.pdf") as pdf:
    for page in pdf.pages:
        tables = page.extract_tables()
        for table in tables:
            for row in table:
                if row is None:
                    continue
                row = [str(c).strip() if c else '' for c in row]
                if len(row) >= 5:
                    try:
                        score = float(row[2]) if row[2] else float(row[3])
                        rows.append(row)
                    except ValueError:
                        continue

df_hri = pd.DataFrame(rows)
df_hri.columns = ['state', 'district', 'hri_normalised', 'hri_score', 'hri_category', 'intrastate_rank']
df_hri['district'] = df_hri['district'].str.replace('\n', ' ', regex=False)
df_hri.to_csv("data/ceew_district_hri.csv", index=False)
```

### Step 2 — Fetch NFHS-5 district data and merge

```python
import requests
import pandas as pd
from io import StringIO

state_names = {
    'AN': 'Andaman-and-Nicobar-Island', 'AP': 'Andhra-Pradesh',
    'AS': 'Assam', 'BR': 'Bihar', 'DD': 'Dadra-Nagar-Haveli-and-Daman-Diu',
    'GA': 'Goa', 'GJ': 'Gujarat', 'HP': 'Himachal-Pradesh',
    'JK': 'Jammu-and-Kashmir', 'KA': 'Karnataka', 'KL': 'Kerala',
    'LH': 'Ladakh', 'MH': 'Maharashtra', 'ML': 'Meghalaya',
    'MN': 'Manipur', 'MZ': 'Mizoram', 'NL': 'Nagaland',
    'SK': 'Sikkim', 'TG': 'Telangana', 'TR': 'Tripura', 'WB': 'West-Bengal'
}

dfs = []
for code, name in state_names.items():
    url = f"https://raw.githubusercontent.com/pratapvardhan/NFHS-5/master/district-level/NFHS-5-{code}-{name}.csv"
    r = requests.get(url)
    if r.status_code == 200:
        dfs.append(pd.read_csv(StringIO(r.text)))

nfhs = pd.concat(dfs, ignore_index=True)

# Extract indicators and merge with HRI
# See analysis/india_heat_maternal_analysis.R for full merge pipeline
```

### Step 3 — Generate the interactive map

Download the India district shapefile:
```r
download.file(
  url = "https://raw.githubusercontent.com/datta07/INDIAN-SHAPEFILES/master/INDIA/INDIA_DISTRICTS.geojson",
  destfile = "data/india_districts.geojson",
  method = "wininet"  # Windows; use "libcurl" on Mac/Linux
)
```

Then run `analysis/india_heat_maternal_analysis.R` in RStudio with the working directory set to the project root. The script will generate `visual1-map.html` in the project root.

---

## Methodology notes

- HRI data was extracted from the CEEW state-level annex PDF using `pdfplumber`. 724 of 734 districts were successfully extracted; 10 districts were lost to page-boundary rendering issues.
- NFHS-5 district indicators were fetched directly from the pratapvardhan/NFHS-5 GitHub repository. 277 of 340 districts were matched to CEEW HRI data by state and district name after standardising case.
- The interactive map matches 566 of 724 HRI districts to the datta07 shapefile. Grey districts reflect name mismatches between the 2011 Census delimitation used by CEEW and the shapefile's naming conventions. The 2022 post-reorganisation shapefile was not used due to boundary changes.
- The trimester calendar (Visual 2) illustrates published findings from Vincent et al. 2026 and does not represent original statistical analysis by the author.
- Double-jeopardy districts are defined as those simultaneously meeting all three criteria: HRI ≥ 0.80, child underweight prevalence ≥ 35%, and first-trimester antenatal coverage ≤ 65%.
- The Pearson correlation coefficient shown in Visual 3 is computed on the matched district sample only and reflects geographic co-occurrence, not causal inference.

---

## References

Prabhu, Shravan, Keerthana Anthikat Suresh, Srishti Mandal, Divyanshu Sharma, and Vishwas Chitale. 2025. *How Extreme Heat is Impacting India: Assessing District-level Heat Risk.* New Delhi: Council on Energy, Environment and Water.

Vincent, Vineetha, Darsy Darssan, Nicholas J. Osborne, and Sagnik Dey. 2026. "Effect of extreme heat stress on adverse pregnancy outcomes in India." *Environmental Research Communications* 8: 041009. https://doi.org/10.1088/2515-7620/ae5b4f

National Family Health Survey 5 (NFHS-5), 2019–21. International Institute for Population Sciences, Mumbai. https://dhsprogram.com

IMDAA reanalysis data. National Centre for Medium Range Weather Forecasting, Ministry of Earth Sciences, Government of India. https://rds.ncmrwf.gov.in

---

## Acknowledgements

This project was completed as part of the Data Journalism course at the Hertie School, Berlin (May 2026). The author is grateful to the open-access data policies of CEEW, the NFHS programme, and the pratapvardhan and datta07 GitHub repositories which made district-level analysis possible.

---

*All code and data in this repository are available under the MIT licence. The CEEW Heat Risk Index data is reproduced for non-commercial academic purposes under CC BY-NC 4.0. NFHS-5 data is publicly available from the DHS Programme.*
