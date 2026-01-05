

# ELT proces datasetu OECD International Migration

Tento repozitár predstavuje implementáciu ELT procesu v Snowflake a vytvorenie dátového skladu so Star schémou. Projekt pracuje s [OECD International Migration Database](https://app.snowflake.com/marketplace/listing/GZSUZCN9EC/data-army-intel-migration-statistics-oecd-free?search=intel%20data%20army) datasetom. Projekt sa zameriava na analýzu migračných tokov, identifikáciu kľúčových krajín určenia a pôvodu, a skúmanie demografickej štruktúry migrantov. Výsledný dátový model umožňuje efektívnu multidimenzionálnu analýzu.

## 1. Úvod a popis zdrojových dát

V tomto projekte analyzujeme dáta o medzinárodnej migrácii. Cieľom je porozumieť:
- dynamike migračných tokov v čase,
- najpopulárnejším destináciám pre migrantov,
- pomeru medzi prílivom (inflows) a odlivom (outflows) obyvateľstva,
- demografickým charakteristikám (pohlavie, občianstvo).

Zdrojové dáta pochádzajú z OECD databázy dostupnej prostredníctvom Snowflake Marketplace (poskytovateľ Data Army Intel). Pôvodný súbor údajov obsahuje 4 denormalizované tabuľky:
- INTERNATIONAL_MIGRATION_DATABASE - obsahuje všetky údaje o krajinách, rokoch, demografii a nameraných hodnotách v jednom "plochom" formáte.
- INTERNATIONAL_MIGRATION_DATABASE_STOCKS_OF_FOREIGN_BORN_POPULATION - údaje o stave populácie (Stocks), teda koľko ľudí narodených v zahraničí celkovo žije v daných krajinách k určitému dátumu.
- LABOUR_MARKET_OUTCOMES_OF_IMMIGRANTS_EMPLOYMENT_RATES_BY_EDUCATIONAL_ATTAINMENT - štatistika trhu práce (zamestnanosť) v rozdelení podľa vzdelania. Ukazuje percentuálny podiel zamestnaných prisťahovalcov v závislosti od ich úrovne vzdelania.
- LABOUR_MARKET_OUTCOMES_OF_IMMIGRANTS_EMPLOYMENT_UNEMPLOYMENT_AND_PARTICIPATION_RATES_BY_SEX - celková statistika trhu práce (zamestnanosť, nezamestnanosť) v rozdelení podľa pohlavia (muži/ženy).
<img width="2550" height="1438" alt="erd_schema" src="https://github.com/justnonameuser/project-Aphex/blob/main/img/erd_schema.png"/>

V rámci analýzy bola použitá iba prvá tabuľka, pretože obsahuje najviac údajov a je ideálna pre Star schému.
Účelom ELT procesu bolo tieto surové dáta vyčistiť, normalizovať a transformovať do hviezdicovej schémy vhodnej na analytické účely.
---
## 2. Dimenzionálny model
<img alt="star_schema" src="https://github.com/justnonameuser/project-Aphex/blob/main/img/star-schema.png"/>
Pre potreby reportingu bola navrhnutá schéma hviezdy (Star Schema), ktorá obsahuje 1 tabuľku faktov FACT_MIGRATION a 5 dimenzií:
- DIM_COUNTRY: Obsahuje informácie o krajine, ktorá reportuje štatistiku (krajina určenia).
- DIM_ORIGIN: Obsahuje údaje o krajine pôvodu migrantov (miesto narodenia).
- DIM_TIME: Časová dimenzia obsahujúca roky a desaťročia.
- DIM_DEMOGRAPHICS: Kombinácia atribútov pohlavia (Gender) a občianstva (Citizenship).
- DIM_FLOW: Popisuje typ merania (napr. Inflows, Outflows).

---
## 3. ELT proces v Snowflake

### 3.1 Extract (Extrahovanie dát)

Dáta zo zdrojovej databázy boli skopírované do staging tabuľky pre ďalšie spracovanie. Týmto krokom sme oddelili produkčné dáta od analytických operácií.
```sql
create table migration_staging as select * from MIGRATION__STATISTICS__OECD__FREE.MIGRATION_OECD_FREE.INTERNATIONAL_MIGRATION_DATABASE;
```
---

### 3.2 Transform (Transformácia dát - Dimenzie)

V tejto fáze boli vytvorené dimenzie pomocou príkazu SELECT DISTINCT na extrakciu unikátnych záznamov a ROW_NUMBER() na generovanie náhradných kľúčov (surrogate keys).

DIM_COUNTRY:
```sql
CREATE OR REPLACE TABLE DIM_COUNTRY AS SELECT
    ROW_NUMBER() OVER (ORDER BY REFERENCE_AREA_CODE) as country_id,
    REFERENCE_AREA_CODE as country_code,
    REFERENCE_AREA_DESCRIPTION as country_name
FROM (SELECT DISTINCT REFERENCE_AREA_CODE, REFERENCE_AREA_DESCRIPTION FROM MIGRATION_STAGING);
```
DIM_ORIGIN:
```sql
CREATE OR REPLACE TABLE DIM_ORIGIN AS SELECT 
    ROW_NUMBER() OVER (ORDER BY PLACE_OF_BIRTH_CODE) as iddim_origin,
    PLACE_OF_BIRTH_CODE as origin_code,
    PLACE_OF_BIRTH_DESCRIPTION as origin_name
FROM (SELECT DISTINCT PLACE_OF_BIRTH_CODE, PLACE_OF_BIRTH_DESCRIPTION FROM MIGRATION_STAGING);
```
DIM_TIME:
```sql
create or replace TABLE DIM_TIME as SELECT 
ROW_NUMBER() OVER (ORDER BY TIME_PERIOD) as time_id,
CAST(TIME_PERIOD as INTEGER) as year
FROM (SELECT DISTINCT TIME_PERIOD from MIGRATION_STAGING);
```
Dimenzia DIM_DEMOGRAPHICS kombinuje pohlavie a občianstvo do jedného ID pre optimalizáciu modelu:
```sql
create or replace TABLE DIM_DEMOGRAPHICS AS SELECT
ROW_NUMBER() OVER (ORDER BY SEX_CODE, CITIZENSHIP_CODE) AS iddim_demographics,
SEX_CODE,
SEX_DESCRIPTION as gender,
CITIZENSHIP_CODE,
CITIZENSHIP_DESCRIPTION as citizenship from (select DISTINCT SEX_CODE, SEX_DESCRIPTION, CITIZENSHIP_CODE, CITIZENSHIP_DESCRIPTION FROM MIGRATION_STAGING);
```
DIM_FLOW:
```sql
create or replace TABLE DIM_FLOW AS SELECT 
ROW_NUMBER() OVER (ORDER BY MEASURE_CODE) as flow_id,
MEASURE_CODE AS measure_code,
MEASURE_DESCRIPTION as measure_desc FROM (SELECT DISTINCT MEASURE_CODE, MEASURE_DESCRIPTION FROM MIGRATION_STAGING);
```
---
### 3.3 Load (Načítanie faktov)
Faktová tabuľka FACT_MIGRATION vznikla spojením (JOIN) staging tabuľky so všetkými novovytvorenými dimenziami.

Dôležitou súčasťou transformácie bolo pretypovanie stĺpca OBSERVATION_VALUE na INT a použitie Window funkcie LAG(), ktorá je požiadavkou zadania. Táto funkcia pridáva stĺpec prev_year_count (hodnota z minulého roka) pre analýzu medziročných zmien.
```sql
CREATE OR REPLACE TABLE FACT_MIGRATION AS SELECT
ROW_NUMBER() OVER (order by MIGRATION_STAGING.TIME_PERIOD) as migration_id, DIM_COUNTRY.country_id, DIM_ORIGIN.IDDIM_ORIGIN, DIM_TIME.time_id, DIM_DEMOGRAPHICS.IDDIM_DEMOGRAPHICS, DIM_FLOW.flow_id,CAST(MIGRATION_STAGING.OBSERVATION_VALUE AS int) as migrant_count,

LAG(CAST(migration_staging.OBSERVATION_VALUE AS INT)) OVER (PARTITION BY DIM_COUNTRY.country_id, DIM_ORIGIN.IDDIM_ORIGIN, DIM_DEMOGRAPHICS.IDDIM_DEMOGRAPHICS, DIM_FLOW.flow_id ORDER BY DIM_TIME.year) as prev_year_count
FROM MIGRATION_STAGING 

JOIN DIM_COUNTRY ON MIGRATION_STAGING.REFERENCE_AREA_CODE = DIM_COUNTRY.country_code
JOIN DIM_ORIGIN ON MIGRATION_STAGING.PLACE_OF_BIRTH_CODE = DIM_ORIGIN.ORIGIN_CODE
JOIN DIM_TIME ON CAST(MIGRATION_STAGING.TIME_PERIOD AS INTEGER) = DIM_TIME.YEAR
join DIM_DEMOGRAPHICS ON MIGRATION_STAGING.SEX_CODE = DIM_DEMOGRAPHICS.SEX_CODE AND MIGRATION_STAGING.CITIZENSHIP_CODE = DIM_DEMOGRAPHICS.CITIZENSHIP_CODE
JOIN DIM_FLOW ON MIGRATION_STAGING.MEASURE_CODE = DIM_FLOW.MEASURE_CODE;
```

## 4. Vizualizácia dát
Dashboard obsahuje 5 kľúčových vizualizácií.
### 1) Dynamika migrácie
```sql
SELECT DIM_TIME.YEAR,SUM(FACT_MIGRATION.MIGRANT_COUNT) as TOTAL_MIGRANTS
FROM FACT_MIGRATION
JOIN DIM_TIME ON FACT_MIGRATION.TIME_ID = DIM_TIME.TIME_ID
JOIN DIM_FLOW ON FACT_MIGRATION.FLOW_ID = DIM_FLOW.FLOW_ID
WHERE DIM_FLOW.MEASURE_DESC LIKE '%Inflows%'
GROUP BY DIM_TIME.YEAR
ORDER BY DIM_TIME.YEAR;
```
<img alt="dynamika_migracie.png" src="https://github.com/justnonameuser/project-Aphex/blob/main/img/dynamika_migracie.png"/>

### 2) Odhadovaná migrácia mužov (total - female)
```sql
SELECT DIM_TIME.year,SUM(CASE 
WHEN DIM_DEMOGRAPHICS.GENDER = 'Total' 
THEN FACT_MIGRATION.migrant_count
END) - SUM(CASE WHEN DIM_DEMOGRAPHICS.GENDER = 'Female' 
THEN FACT_MIGRATION.migrant_count
END) AS MALE_COUNT
FROM FACT_MIGRATION
JOIN DIM_TIME 
    ON FACT_MIGRATION.time_id = DIM_TIME.time_id
JOIN DIM_DEMOGRAPHICS 
    ON FACT_MIGRATION.iddim_demographics = DIM_DEMOGRAPHICS.iddim_demographics
JOIN DIM_FLOW 
    ON FACT_MIGRATION.flow_id = DIM_FLOW.flow_id
WHERE DIM_FLOW.measure_code = 'B11'
GROUP BY DIM_TIME.year
ORDER BY DIM_TIME.year;
```
<img alt="migracia_muzov.png" src="https://github.com/justnonameuser/project-Aphex/blob/main/img/migracia_muzov.png"/>

### 3) Top krajín, odkiaľ prichádzajú
```sql
SELECT DIM_DEMOGRAPHICS.CITIZENSHIP as ORIGIN_COUNTRY,
SUM(FACT_MIGRATION.migrant_count) as TOTAL_PEOPLE
FROM FACT_MIGRATION
JOIN DIM_DEMOGRAPHICS ON FACT_MIGRATION.iddim_demographics = DIM_DEMOGRAPHICS.iddim_demographics
JOIN DIM_FLOW ON FACT_MIGRATION.flow_id = DIM_FLOW.flow_id
WHERE DIM_FLOW.measure_code = 'B11' 
AND DIM_DEMOGRAPHICS.CITIZENSHIP NOT IN ('Total', 'Stateless', 'World', 'World unspecified','European Union (15 countries)')
AND FACT_MIGRATION.migrant_count IS NOT NULL
GROUP BY DIM_DEMOGRAPHICS.CITIZENSHIP
ORDER BY TOTAL_PEOPLE DESC LIMIT 10;
```
<img alt="odkial_prichadzaju.png" src="https://github.com/justnonameuser/project-Aphex/blob/main/img/odkial_prichadzaju.png"/>

### 4) Top krajín, kam prichádzajú
```sql
SELECT DIM_COUNTRY.country_name, SUM(FACT_MIGRATION.migrant_count) as TOTAL_INFLOW
FROM FACT_MIGRATION 
JOIN DIM_COUNTRY ON FACT_MIGRATION.country_id = DIM_COUNTRY.country_id
JOIN DIM_FLOW ON FACT_MIGRATION.flow_id = DIM_FLOW.flow_id
WHERE DIM_FLOW.measure_code = 'B11' AND FACT_MIGRATION.migrant_count IS NOT NULL
GROUP BY DIM_COUNTRY.country_name
ORDER BY TOTAL_INFLOW DESC LIMIT 10;
```
<img alt="kam_prichadzaju.png" src="https://github.com/justnonameuser/project-Aphex/blob/main/img/kam_prichadzaju.png"/>

### 5) Príchod proti odchodu
```sql
SELECT DIM_FLOW.measure_desc,SUM(FACT_MIGRATION.migrant_count) as TOTAL_COUNT
FROM FACT_MIGRATION
JOIN DIM_FLOW ON FACT_MIGRATION.flow_id = DIM_FLOW.flow_id
WHERE DIM_FLOW.measure_code IN ('B11', 'B12') 
GROUP BY DIM_FLOW.measure_desc;
```
<img alt="prichod_odchod.png" src="https://github.com/justnonameuser/project-Aphex/blob/main/img/prichod_odchod.png"/>

---
**Autor:** Maksym Zukh
