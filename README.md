# bank_sync – bankové transakcie do Google Sheetu

Každý deň stiahne cez bankové API transakcie za **predchádzajúci aj aktuálny deň** zo všetkých nastavených účtov
a doplní ich do tabuľky v Google Sheets. Transakcie, ktoré už v tabuľke sú (podľa stĺpca
„ID transakcie“), sa znova nezapíšu, takže opakované spustenie ani spätné dotiahnutie viacerých dní
nevytvorí duplicity.

Podporované banky:

| Provider        | Banky                                                         | Prístup                                   |
|-----------------|---------------------------------------------------------------|-------------------------------------------|
| `fio`           | Fio banka                                                     | API token z internetbankingu              |
| `enablebanking` | Tatra banka, SLSP, VÚB, ČSOB SK, 365.bank, UniCredit a ďalšie | PSD2 cez [Enable Banking](https://enablebanking.com) |

Stĺpce v hárku: `Dátum, Účet protistrany, Protistrana, Suma, VS, Popis, IBAN účtu, ID transakcie,
Stiahnuté`. IBAN účtu je IBAN vášho účtu, z ktorého bola transakcia stiahnutá.
Suma sa zobrazuje s desatinnou čiarkou (`1 234,50`) – nástroj nastaví tabuľke slovenské jazykové
nastavenie (`locale: sk_SK`, dá sa zmeniť v konfigurácii).
ID transakcie má tvar `<názov účtu>:<ID z banky>` a slúži na rozpoznanie už zapísaných transakcií.
Vlastné stĺpce pridané za tieto stĺpce nástroj nechá tak. Dátum je skutočný dátum vo formáte `DD.MM.YYYY` (dá sa podľa neho
triediť a filtrovať). Odchádzajúce platby majú zápornú sumu. Ukladajú sa len zaúčtované
transakcie (nie čakajúce/blokované).

## 1. Google Sheet

1. V [Google Cloud Console](https://console.cloud.google.com/) vytvorte projekt, zapnite
   **Google Sheets API** a vytvorte **Service account** → Keys → *Add key* → JSON.
2. Tabuľku zdieľajte (Share) s e-mailom service accountu (`...@...iam.gserviceaccount.com`)
   s právom **Editor**.
3. ID tabuľky je časť URL medzi `/d/` a `/edit`.

Hárok (predvolene `Transakcie`) sa vytvorí aj s hlavičkou automaticky.

## 2. Banky

### Fio banka
Internetbanking → Nastavenia → API → Pridať nový token, oprávnenie **len sledovanie účtu**.
Token uložte do premennej prostredia, ktorej názov uvediete v `token_env`.

### Ostatné banky (Enable Banking)
1. Zaregistrujte sa na <https://enablebanking.com/cp/applications> a vytvorte aplikáciu
   (pre vlastné účty stačí bezplatný režim *restricted production*). Uložte si `application_id`
   a stiahnutý súkromný kľúč `.pem`. Ako redirect URL môžete nechať `https://enablebanking.com/`.
2. Doplňte `enablebanking` sekciu do `config.yaml` a potom:

```bash
export ENABLEBANKING_PRIVATE_KEY="$(cat moj-kluc.pem)"
python -m bank_sync eb-banks SK                 # presné názvy bánk
python -m bank_sync eb-auth "Tatra banka" SK    # vypíše odkaz na prihlásenie do banky
# po prihlásení skopírujte z adresy presmerovania hodnotu ?code=...
python -m bank_sync eb-session <code>           # vypíše account_uid jednotlivých účtov
```

3. `account_uid` vložte k účtu do `config.yaml`.

> ⚠️ Súhlas s prístupom (PSD2 consent) platí podľa banky 90–180 dní. Po vypršaní synchronizácia
> daného účtu zlyhá s chybou a treba zopakovať `eb-auth` + `eb-session` (uid účtu sa môže zmeniť).

## 3. Lokálne spustenie

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
cp config.example.yaml config.yaml    # a upravte

export GOOGLE_APPLICATION_CREDENTIALS=service-account.json
export FIO_TOKEN_1=...

python -m bank_sync sync --dry-run     # len vypíše, čo by zapísal
python -m bank_sync sync               # včerajšok a dnešok
python -m bank_sync sync --days 7      # posledných 7 dní vrátane dneška (už zapísané sa preskočia)
python -m bank_sync sync --from 2026-01-01 --date 2026-06-30   # spätné načítanie
python -m bank_sync sync --account "Fio firemný"               # len jeden účet
```

Každý účet môže mať vlastný hárok (`worksheet:` pri účte); bez neho sa zapisuje do
`google_sheet.worksheet`. Chýbajúci hárok sa vytvorí automaticky.

Ak niektorý účet zlyhá, ostatné sa aj tak zapíšu a príkaz skončí s návratovým kódom 1.

## 4. Denné spúšťanie: cron-job.org → GitHub Actions

Synchronizáciu vykonáva workflow `.github/workflows/daily-sync.yml`. Sám sa nespúšťa – každé ráno
o 6:00 ho cez GitHub API spustí služba [cron-job.org](https://cron-job.org) (tá pozná časové pásmo
Europe/Bratislava a spúšťa presne načas). Ručne sa dá spustiť aj cez Actions → *Run workflow*
(s voliteľným počtom dní).

### GitHub token pre cron-job.org
GitHub → *Settings → Developer settings → Personal access tokens → Fine-grained tokens →
Generate new token*:
- **Repository access:** *Only select repositories* → `tsunami-bank-transactions`
- **Permissions → Repository permissions → Actions:** *Read and write* (nič iné netreba)
- **Expiration:** čo najdlhšia; dátum si poznačte – po vypršaní treba token v cron-job.org vymeniť

### Úloha v cron-job.org
*Cronjobs → Create cronjob*:

| Pole | Hodnota |
|---|---|
| URL | `https://api.github.com/repos/M99-claude/tsunami-bank-transactions/actions/workflows/daily-sync.yml/dispatches` |
| Execution schedule | každý deň o **06:00**, časové pásmo **Europe/Bratislava** |
| Advanced → Request method | `POST` |
| Advanced → Headers | `Authorization: Bearer <GitHub token>`<br>`Accept: application/vnd.github+json`<br>`X-GitHub-Api-Version: 2022-11-28` |
| Advanced → Request body | `{"ref": "claude/sleepy-babbage-1ex9dt"}` |

`ref` je vetva s kódom (predvolená vetva repozitára). Voliteľne sa dá poslať aj počet dní:
`{"ref": "claude/sleepy-babbage-1ex9dt", "inputs": {"days": "3"}}`.

Úspešné spustenie vráti **HTTP 204** (prázdna odpoveď). Tlačidlom *Test run* v cron-job.org si
to overíte – v GitHube sa hneď objaví nový beh v záložke Actions.

### Secrets
V repozitári nastavte *Settings → Secrets and variables → Actions*:

| Secret                        | Obsah                                              |
|-------------------------------|----------------------------------------------------|
| `CONFIG_YAML`                 | celý obsah vášho `config.yaml`                     |
| `GOOGLE_SERVICE_ACCOUNT_JSON` | obsah JSON kľúča service accountu                  |
| `ENABLEBANKING_PRIVATE_KEY`   | obsah `.pem` súboru (ak používate Enable Banking)  |
| `FIO_TOKEN_1` …               | token každého Fio účtu – názov podľa `token_env`   |

Pri pridaní ďalšieho Fio účtu doplňte jeho premennú aj do sekcie `env` vo workflow.
GitHub pri zlyhaní behu pošle e-mail – tak sa dozviete napr. o vypršanom súhlase banky.
Ak by zlyhalo samotné spustenie (napr. vypršaný GitHub token), upozorní vás cron-job.org
(v nastaveniach úlohy zapnite notifikácie pri zlyhaní).

Alternatívne stačí cron na vlastnom serveri:

```cron
0 7 * * *  cd /opt/bank_sync && .venv/bin/python -m bank_sync sync >> sync.log 2>&1
```

## Testy

```bash
pip install pytest && python -m pytest
```
