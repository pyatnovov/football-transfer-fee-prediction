# football-transfer-fee-prediction
# Čistenie dát — Prediction of Football Transfer Fees

Stručný popis fázy čistenia a prípravy pôvodných dát z datasetu **Football Data from Transfermarkt** (Kaggle) pre následný Feature Engineering a trénovanie ML modelov na predikciu prestupovej sumy futbalistov.

## Prehľad výsledkov

| Tabuľka | Pôvodné dáta (riadky × stĺpce) | Po čistení | Výstupný súbor |
| :--- | :---: | :---: | :--- |
| `transfers` | 175 165 × 10 | 16 624 × 7 | `clean_transfers.csv` |
| `players` | 50 149 × 26 | 9 779 × 7 | `clean_players.csv` |
| `appearances` | 1 894 350 × 13 | 883 515 × 9 | `clean_appearances.csv` |
| `clubs` | 796 × 17 | 755 × 8 | `clean_clubs.csv` |
| `competitions` | 65 × 11 | 65 × 7 | `clean_competitions.csv` |

---

## Kroky čistenia podľa tabuliek

### 1. `transfers`
* Vybraných 7 kľúčových atribútov: `player_id`, `transfer_date`, `transfer_season`, `from_club_id`, `to_club_id`, `transfer_fee`, `market_value_in_eur`.
* Stĺpec `transfer_date` konvertovaný do formátu `datetime`.
* Odstránené úplné duplikáty a anomálne záznamy, kde `from_club_id == to_club_id`.
* Ponechané iba platené prestupy (`transfer_fee > 0`); odstránené prestupy voľných hráčov, hosťovania a chýbajúce hodnoty cieľovej premennej.
* Obmedzený časový rozsah: **`2013-07-01` — `2026-07-06`** (odstránené budúce/plánované dátumy a staršie prestupy pred rokom 2013, pre ktoré v `appearances` chýba minimálne jedna celá sezóna štatistík).
* Chýbajúce hodnoty v `market_value_in_eur` (886 riadkov, 5,3 %) boli zámerne ponechané pre flexibilitu pri modelovaní.

### 2. `players`
* Vybraných 7 statických atribútov: `player_id`, `country_of_citizenship`, `date_of_birth`, `position`, `sub_position`, `foot`, `height_in_cm`.
* **Prevencia Data Leakage (úniku dát):** odstránené stĺpce odrážajúce stav hráča k dátumu stiahnutia datasetu, nie k dátumu prestupu (`market_value_in_eur`, `highest_market_value_in_eur`, `contract_expiration_date`, `international_caps`, `international_goals`, `current_club_id`).
* Stĺpec `date_of_birth` konvertovaný do formátu `datetime`; odstránené duplikáty podľa `player_id`.
* Odstránené záznamy bez dátumu narodenia a riadok s nekorektnou pozíciou (`position == 'Missing'`).
* Textové chýbajúce hodnoty (`country_of_citizenship`, `sub_position`, `foot`) nahradené hodnotou `'Unknown'`.
* Skontrolované anomálie výšky (`< 140` cm alebo `> 220` cm nahradené hodnotou `NaN`).

### 3. `appearances`
* Vybraných 9 atribútov: `player_id`, `player_club_id`, `date`, `competition_id`, `yellow_cards`, `red_cards`, `goals`, `assists`, `minutes_played` (odstránený stĺpec `player_current_club_id` pre riziko Data Leakage).
* Stĺpec `date` konvertovaný do formátu `datetime`; odstránené úplné duplikáty.
* Vyfiltrované záznamy s `minutes_played > 0` a nezápornými hodnotami gólov, asistencií a kariet.
* Pokryté obdobie zápasov: **`2012-07-05` — `2026-06-28`**, bez chýbajúcich hodnôt.

### 4. `clubs`
* Vybraných 8 atribútov: `club_id`, `domestic_competition_id`, `squad_size`, `average_age`, `foreigners_percentage`, `national_team_players`, `stadium_seats`, `net_transfer_record` (odstránený úplne prázdny stĺpec `total_market_value`).
* Odstránené duplikáty podľa `club_id`.
* Stĺpec `net_transfer_record` prevedený z textového formátu (napr. `+€5.90m`, `€-25.00m`, `+-0`) na číselný typ `float` (v eurách).
* Nulové anomálie v `squad_size`, `average_age` a `stadium_seats` nahradené hodnotou `NaN`.

### 5. `competitions`
* Vybraných 7 atribútov: `competition_id`, `name`, `sub_type`, `type`, `country_name`, `confederation`, `total_clubs`.
* Odstránené duplikáty podľa `competition_id`.
* Chýbajúce hodnoty v `country_name` (medzinárodné turnaje) nahradené hodnotou `'International'`; chýbajúce hodnoty v `total_clubs` (15 medzinárodných/pohárových súťaží) ponechané bez zmeny.

---

## Finálna synchronizácia a poznámky pre ďalšie fázy
* Všetky tabuľky boli vzájomne zosúladené podľa kľúčov `player_id` a `club_id` voči finálnemu datasetu `clean_transfers.csv`.
* **Číselné chýbajúce hodnoty (`NaN`)** v `market_value_in_eur`, `height_in_cm` a vybraných stĺpcoch `clubs` neboli v tejto fáze zámerne dopĺňané — ich imputáciu je potrebné vykonať až v rámci ML pipeline **po rozdelení dát na `train / test`**.
* Vo fáze Feature Engineering je nevyhnutné agregovať dáta z `clean_appearances.csv` striktne za podmienky `appearances.date < transfers.transfer_date` a rozdelenie na trénovaciu a testovaciu množinu realizovať chronologicky (Time-Based Split).
