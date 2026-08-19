# Data Download Instructions — Ali (2022) Replication Assignment

Two of the four data sources used in Ali (2022) **ship with the replication package**:

| File | Contents | Key |
|---|---|---|
| `new_deal_data.xlsx` | FHA Title II insurance (`FHA_T2`) and total New Deal loans (`LOANS`) by county, 1935–1939 | `fips`, `new_icpsr` |
| `distances_combined_final.xlsx` | Distance (miles) from each county to its FHA field office (`calculated_distance`) | `county1` (FIPS) |

The other two sources are **not** in the package but we can download them setting up an account. This document explains how to get each one.

---

## A. Free 1940 county controls via IPUMS NHGIS

The paper's *full* Table 1 adds nine standardized 1940 county controls (population, density, Black
population, White population, foreign-born, housing units, employed, seeking work, public emergency
workers). The paper obtained these from **Social Explorer**, which requires a paid subscription.

A **free public substitute** is **IPUMS NHGIS**, which distributes the same underlying 1940 Census county data. Create a free account and download the county data; the numbers are equivalent to Social Explorer's referred to in the paper.

**Expect to end up with eight of the nine.**  It appears the NHGIS 1940 county tables do not report population
density, so build the other eight, estimate with those, and say so in your write-up.

### Option 1 (recommended): download via the `ipumsr` API

Instead of building the extract by hand on the website, you can request and download it
programmatically with the **`ipumsr`** package and the IPUMS API. This is faster to repeat and fully reproducible. A ready-to-edit draft is at **`scripts/03_nhgis_download.R`**.

1. Go to <https://www.nhgis.org/> and click **Register** to create a free account if you don't have an IPUMS account yet.
2. Read the NHGIS API guide: <https://tech.popdata.org/ipumsr/articles/ipums-api-nhgis.html>. Also read the `ipumsr` docs: <https://tech.popdata.org/ipumsr/> 
3. Once you have an IPUMS account, get a free API key at <https://account.ipums.org/api_keys>. In a standalone R session, register your API key (replace YOUR_KEY_HERE with the (long string of numbers and letters) API key you obtained):
   ```r
   library(ipumsr)
   set_ipums_api_key("YOUR_KEY_HERE", save = TRUE)
   ```
   This registers your API key once on your local computer (writes it to your `~/.Renviron`). This is good practice so it is not hard-coded in scripts and potentially shared by mistake later on.

4. The dataset and table codes you need are below. They were current when this was written, but
   NHGIS revises them, so **check them against the live catalog before you submit** rather than
   assuming:
   ```r
   # confirm the 1940 county dataset still exists under this name:
   get_metadata_nhgis(type = "datasets") |>
     dplyr::filter(group == "1940 Census")
   # then list the data tables inside it:
   get_metadata_nhgis(dataset = "1940_cPHAE")$data_tables
   ```
   Dataset **`1940_cPHAE`**, and these six tables:

   | Table | Contents | Controls it supplies |
   |---|---|---|
   | `NT1`   | Total population | `total_pop` |
   | `NT6`   | Population by race and nativity | `black_pop`, `white_pop`, `foreign_born` |
   | `NT43`  | Dwelling/housing units | `housing_units` |
   | `NT25`  | Persons employed | `employed` |
   | `NT29`  | Persons seeking work | `seeking_work` |
   | `NT28B` | Public emergency workers | `emergency_workers` |

5.  In the R code (after loading the **`ipumsr`** package), define and submit the extract at the **county** geographic level:
   ```r
   define_extract_agg(
     collection  = "nhgis",
     description = "GU4911 -- 1940 county controls",
     datasets    = ds_spec("1940_cPHAE",
                           data_tables = c("NT1", "NT6", "NT43",
                                           "NT25", "NT29", "NT28B"),
                           geog_levels = "county"),
     data_format = "csv_no_header"
   ) |>
     submit_extract() |>
     wait_for_extract() |>
     download_extract(download_dir = "data/Social Explorer data")
   ```
   The download folder matters: the prep scripts read the raw extract from
   `data/Social Explorer data/`, named for the paid source NHGIS stands in for, so save it there.
6. Read it with `read_ipums_agg()`, build a 5-digit `fips` from the state/county FIPS columns, and
   rename the cryptic value columns to the names listed under "Format expected by the analysis"
   below.

The draft `scripts/03_nhgis_download.R` walks through all of this, including the metadata-discovery
steps, and `scripts/04_nhgis_dataprep.R` reshapes the raw extract into the format below.

> **If a function is not found:** `define_extract_agg(collection = "nhgis", ...)` and
> `read_ipums_agg()` are the current `ipumsr` functions. Older versions called them
> `define_extract_nhgis()` and `read_nhgis()` and took the same arguments. If R cannot find
> `define_extract_agg()`, update the package rather than falling back to the old name.

### Option 2: build the extract in the NHGIS Data Finder (point and click)

1. Go to <https://www.nhgis.org/> and click **Register** to create a free account.
2. Open the **Data Finder** (the "Get Data" / Select Data tool).
3. Apply these filters:
   - **Geographic levels:** County
   - **Years:** 1940
   - **Topics:** Total Population; Race; Housing (Dwelling/Housing Units); Labor Force / Employment;
     Nativity (Foreign-born). There is no density topic to select for 1940 — see the note above.
4. Select the source tables listed in Option 1, step 4: `NT1`, `NT6`, `NT43`, `NT25`, `NT29` and
   `NT28B`, all from the 1940 county dataset `1940_cPHAE`. The Data Finder shows these codes
   alongside each table's description, so you can match on either.
5. Add the tables to your **Data Cart**, choose the **county** geographic level, and submit the
   extract. NHGIS emails you when it is ready.
6. Download the CSV into `data/Social Explorer data/`, which is where the prep scripts look for it.
   NHGIS files include GISJOIN and state/county FIPS codes; build a 5-digit county FIPS
   (`STATEFP` + `COUNTYFP`, zero-padded) so it matches the assignment's `fips` key.

### Format expected by the analysis

Whichever option you used, you should end up with one row per county: a 5-digit character column
`fips` plus these eight numeric columns.

```
fips, total_pop, black_pop, white_pop, foreign_born,
housing_units, employed, seeking_work, emergency_workers
```

Standardize each of the eight (subtract the mean, divide by the standard deviation across counties)
before putting them in the regression, since that is how the paper enters them. Adding the controls
should move the first-stage distance coefficient toward the paper's −0.339.

> **Note on column matching:** the table codes above get you the right tables; the *columns inside*
> them still arrive under cryptic names like `BXXXXX001`. Use the codebook (`.txt`) that downloads
> with every NHGIS extract to map those to the eight variables above — that mapping is the
> data-cleaning exercise, and it is not mechanical. Several of the eight are not a single column
> but the sum of two, because the 1940 tables break the count into parts (white population, for
> instance, is split by nativity). Read each column's label and universe in the codebook before you
> rename or add anything, and sanity-check the result before moving on. Three checks that must hold
> for every county: `white_pop + black_pop <= total_pop`, `foreign_born <= white_pop` (foreign-born
> white is a subset of the white population in these tables), and `employed <= total_pop`.

---

## B. IPUMS USA microdata for the paper's main result (Table 2)

Table 2 — the paper's central result — is estimated on **individual-level census microdata**. This
is **free** but requires an account. 

### Option 1 (recommended): download via the `ipumsr` API

Instead of building the extract by hand on the website, you can request and download it
programmatically with the **`ipumsr`** package and the IPUMS API. This is faster to repeat and fully reproducible. A ready-to-edit draft is provided at **`scripts/01_ipums_download.R`**.

1. Read the `ipumsr` docs: <https://tech.popdata.org/ipumsr/> and the microdata API guide,
   <https://tech.popdata.org/ipumsr/articles/ipums-api-micro.html>.
2. Install the package and get a **free API key** at <https://account.ipums.org/api_keys>.
3. Register the key once in an separate R session (writes it to your `~/.Renviron` so it is not hard-coded in scripts):
   ```r
   library(ipumsr)
   set_ipums_api_key("YOUR_KEY_HERE", save = TRUE)
   ```
   Ignore prior three steps if you did already did this for the IPUMS NHGIS data download.
4. In the R code (after loading the **`ipumsr`** package), the extract is defined, submitted, and downloaded with a short pipeline:
   ```r
   target_samples <- c("us1920a", "us1930a", "us1940a",
                       "us1950a", "us1960b", "us1970d")

   target_variables <- list(
     "YEAR", "OWNERSHP", "VALUEH",
     var_spec("RELATE", case_selections = "1"),   # 1 = Head/Householder
     "SEX", "AGE", "MARST", "RACE", "HISPAN", "SCHOOL", "EMPSTAT", "OCCSCORE",
     "STATEICP", "COUNTYICP", "STATEFIP", "COUNTYFIP"
   )

   define_extract_micro(
     collection  = "usa",
     description = "GU4911 -- Ali (2022) replication",
     samples     = target_samples,
     variables   = target_variables
   ) |>
     submit_extract()   |>
     wait_for_extract() |>
     download_extract(download_dir = "data/IPUMS data")
   ```
   Three things in that block matter more than they look. `var_spec("RELATE", case_selections = "1")`
   asks IPUMS to send **household heads only**, which is the level the analysis runs at; without it
   you download every person in every household and discard most of them after the fact. Mixing plain
   variable names with a `var_spec()` object requires `list()`, not `c()`. And save into
   `data/IPUMS data/`, which is where `02_ipums_dataprep.R` looks for the `.xml` codebook.
5. **Verify the sample codes before submitting.** Confirm the six IDs above against the live catalog
   with `get_sample_info("usa")` rather than trusting a copied list — codes change over time, and the
   1970 "Form 1 / Form 2 Metro" samples have several variants that are easy to confuse. The draft
   script does this check for you and warns if a code is not found. See "Why six samples and not the
   paper's eight" below for what is deliberately left out and why.

A draft script with the above is filled in and commented in `scripts/01_ipums_download.R`; you mainly need to paste
your API key and confirm data you want to download.

### Option 2: build the extract on the IPUMS website

1. Go to <https://usa.ipums.org/usa/> and create a free account.
2. Click **Select Samples** and check these **six** samples:

   | Census year | Sample | IPUMS code |
   |---|---|---|
   | 1970 | Form 2 Metro | `us1970d` |
   | 1960 | 5% | `us1960b` |
   | 1950 | 1% | `us1950a` |
   | 1940 | 1% | `us1940a` |
   | 1930 | 1% | `us1930a` |
   | 1920 | 1% | `us1920a` |

3. Add these variables (from the README): `YEAR`, `OWNERSHP`, `VALUEH`, `RELATE`, `SEX`, `AGE`,
   `MARST`, `RACE`, `HISPAN`, `SCHOOL`, `EMPSTAT`. Also keep the default technical variables IPUMS
   attaches (`SAMPLE`, `SERIAL`, `PERWT`, and the geography variables `STATEICP`, `COUNTYICP`,
   `STATEFIP`, `COUNTYFIP` — the original code uses these to build county codes). Add `OCCSCORE`,
   which the analysis code also uses.
4. Click **Select Cases**, choose `RELATE`, and keep only **1 — Head/Householder**. Do not skip
   this. The analysis is at the household-head level, and this one setting is the difference
   between downloading a few million records and a few tens of millions.
5. Create the extract, choose output format **.dat (fixed-width) with the DDI XML codebook**
   (this is what the `ipumsr` R package reads), and submit. IPUMS emails you when it is ready.
6. Download **both** the data file (`usa_XXXXX.dat.gz`) and the DDI codebook (`usa_XXXXX.xml`) into
   `data/IPUMS data/`. Do **not** unzip the `.dat.gz`; `ipumsr::read_ipums_micro()` reads it
   directly.

### Why six samples and not the paper's eight

The paper's README lists eight, adding **1960 1%** and **1970 Form 1 Metro**. Leave both out:

- **1960 1%** carries no county identifiers, so not one of its records can be merged to the
  county-level FHA and distance data. It would contribute nothing to either table.
- **1970 Form 1 Metro** does not include `SCHOOL`, which is one of the individual controls, so
  every record from it drops out of a complete-case regression anyway.

Downloading them costs you time and disk for no additional usable observations. This is also why
your sample will be smaller than the paper's — say so in your write-up rather than leaving the
reader to wonder.

### Size / compute warning

Even restricted to household heads, these six samples run to millions of person-records. The
restriction in step 4 (or the `case_selections` argument in Option 1) is what keeps this manageable;
without it the extract is roughly an order of magnitude larger, and both the IPUMS-side processing
and your download take correspondingly longer.

## Sources

- [IPUMS NHGIS](https://www.nhgis.org/)
- [Overview of NHGIS Datasets](https://www.nhgis.org/overview-nhgis-datasets)
- [NHGIS Data Availability](https://www.nhgis.org/data-availability)
- [IPUMS USA: select samples](https://usa.ipums.org/usa-action/samples)
- [IPUMS USA: sample descriptions](https://usa.ipums.org/usa/sampdesc.shtml)
- [IPUMS USA: OWNERSHP](https://usa.ipums.org/usa-action/variables/OWNERSHP)
