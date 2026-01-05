

# ELT proces datasetu OECD International Migration

Tento repozitár predstavuje implementáciu ELT procesu v Snowflake a vytvorenie dátového skladu so Star schémou. Projekt pracuje s [OECD International Migration Database](https://app.snowflake.com/marketplace/listing/GZSUZCN9EC/data-army-intel-migration-statistics-oecd-free?search=intel%20data%20army) datasetom. Projekt sa zameriava na analýzu migračných tokov, identifikáciu kľúčových krajín určenia a pôvodu, a skúmanie demografickej štruktúry migrantov. Výsledný dátový model umožňuje efektívnu multidimenzionálnu analýzu.

1. Úvod a popis zdrojových dát

V tomto projekte analyzujeme dáta o medzinárodnej migrácii. Cieľom je porozumieť:
- dynamike migračných tokov v čase,
- najpopulárnejším destináciám pre migrantov,
- pomeru medzi prílivom (inflows) a odlivom (outflows) obyvateľstva,
- demografickým charakteristikám (pohlavie, občianstvo).

Zdrojové dáta pochádzajú z OECD databázy dostupnej prostredníctvom Snowflake Marketplace (poskytovateľ Data Army Intel). Pôvodný súbor údajov obsahuje 4 denormalizované tabuľky:
INTERNATIONAL_MIGRATION_DATABASE - obsahuje všetky údaje o krajinách, rokoch, demografii a nameraných hodnotách v jednom "plochom" formáte.
INTERNATIONAL_MIGRATION_DATABASE_STOCKS_OF_FOREIGN_BORN_POPULATION - údaje o stave populácie (Stocks), teda koľko ľudí narodených v zahraničí celkovo žije v daných krajinách k určitému dátumu.
LABOUR_MARKET_OUTCOMES_OF_IMMIGRANTS_EMPLOYMENT_RATES_BY_EDUCATIONAL_ATTAINMENT - štatistika trhu práce (zamestnanosť) v rozdelení podľa vzdelania. Ukazuje percentuálny podiel zamestnaných prisťahovalcov v závislosti od ich úrovne vzdelania.
LABOUR_MARKET_OUTCOMES_OF_IMMIGRANTS_EMPLOYMENT_UNEMPLOYMENT_AND_PARTICIPATION_RATES_BY_SEX - celková statistika trhu práce (zamestnanosť, nezamestnanosť) v rozdelení podľa pohlavia (muži/ženy).
<img width="2550" height="1438" alt="erd_schema" src="https://github.com/justnonameuser/project-Aphex/blob/main/img/erd_schema.png"/>

V rámci analýzy bola použitá iba prvá tabuľka, pretože obsahuje najviac údajov a je ideálna pre Star schému.
Účelom ELT procesu bolo tieto surové dáta vyčistiť, normalizovať a transformovať do hviezdicovej schémy vhodnej na analytické účely.

2. Dimenzionálny model
<img alt="star_schema" src="https://github.com/justnonameuser/project-Aphex/blob/main/img/star-schema.png"/>
Pre potreby reportingu bola navrhnutá schéma hviezdy (Star Schema), ktorá obsahuje 1 tabuľku faktov FACT_MIGRATION a 5 dimenzií:
- DIM_COUNTRY: Obsahuje informácie o krajine, ktorá reportuje štatistiku (krajina určenia).
- DIM_ORIGIN: Obsahuje údaje o krajine pôvodu migrantov (miesto narodenia).
- DIM_TIME: Časová dimenzia obsahujúca roky a desaťročia.
- DIM_DEMOGRAPHICS: Kombinácia atribútov pohlavia (Gender) a občianstva (Citizenship).
- DIM_FLOW: Popisuje typ merania (napr. Inflows, Outflows).


3. ELT proces v Snowflake

3.1 Extract (Extrahovanie dát)

Dáta zo zdrojovej databázy boli skopírované do staging tabuľky pre ďalšie spracovanie. Týmto krokom sme oddelili produkčné dáta od analytických operácií.
```sql
create table migration_staging as select * from MIGRATION__STATISTICS__OECD__FREE.MIGRATION_OECD_FREE.INTERNATIONAL_MIGRATION_DATABASE;
```
