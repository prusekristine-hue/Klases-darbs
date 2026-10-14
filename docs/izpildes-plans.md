# Prasību izpildes plāns · Sadarbības uzskaites sistēma

Plāns paredzēts Claude Code. Katrs posms ir viena prasība: **viens zars, viens PR, komita un PR nosaukums sākas ar prasības ID** (piem., `F1: organizāciju saraksts`). Pirms posma sākšanas izlasi šo sadaļu un saistīto prasību failu `docs/<ID>.md`, ja tāds ir.

## Vispārīgie noteikumi visiem posmiem

- **Tehnoloģijas (N4):** Flask + Jinja2 + SQLite. Bez JS ietvariem; JS tikai, ja bez tā nevar.
- **Kods:** `app/` mapē; testi `tests/` mapē (pytest, Flask `test_client`). Pēc katra posma: `make test` jāiziet.
- **UI paraugs:** `prototips/index.html` (krāsas, izkārtojums, teksti latviski). Pārnes stilu uz `app/static/style.css`.
- **Pieejamība (N1, WCAG 2.2 AA):** katram laukam `<label for>`; kļūdas teksts pie lauka un `aria-describedby`; tabulām `<caption>` un `<th scope>`; statuss ne tikai ar krāsu (teksts «Kavēts»); redzams fokuss; «Pāriet uz saturu» saite; strādā ar tastatūru un 200 % mērogā.
- **Personas dati (N2):** glabā tikai prasībās minētos laukus; žurnālos (`logging`) nekad neraksta e-pastu, tālruni, vārdu vai brīvo tekstu — tikai ID.
- **Lomas (F0 aizstāj N3):** `darbinieks` redz un labo tikai organizācijas, kur `owner_id` = viņš; `vaditajs` redz visu un F7. Pārbaude **serverī** katrā maršrutā, ne tikai šablonā.
- **Piekļuves dati:** nekādu atslēgu kodā. `SECRET_KEY` un `ANTHROPIC_API_KEY` no vides mainīgajiem.
- **Neaiztikt:** `tracker/`, `docs/openapi.yaml`, `tools/`.
- Pirms komita pārskati `git diff --cached`.

## Datu modelis (SQLite, `app/schema.sql`)

| Tabula | Lauki |
|---|---|
| `users` | id, email (unikāls, mazie burti), name, role (`darbinieks`/`vaditajs`), password_hash |
| `organizations` | id, name, reg_nr, type (`uznemums`/`nvo`/`pasvaldiba`), owner_id → users |
| `contacts` | id, org_id, name, position, email, phone, legal_basis, basis_date, retain_until |
| `interactions` | id, org_id, kind (`zvans`/`epasts`/`sanaksme`), date, participants, summary, next_step, ai_generated (0/1), approved_by → users, created_by → users |
| `requests` | id, org_id, title, status (`jauns`/`darba`/`atbildets`), due_date, assignee_id → users |
| `tasks` | id, title, due_date, done (0/1), owner_id, org_id (nullable), request_id (nullable) |
| `access_log` | id, user_id, org_id, viewed_at |

**Sākumdati (`flask --app app seed`):** 10 organizācijas, 20 kontakti, 30 mijiedarbības, ~12 pieprasījumi (daļa kavēti, termiņi relatīvi pret šodienu), ~10 uzdevumi; lietotāji `vaditajs@piemers.lv`, `darbinieks@piemers.lv`, `peteris@piemers.lv` ar paroli `Demo1234!`. Visi dati izdomāti.

## Posmi un secība

Laika budžets 1,5 h (N4). **Obligātais minimums** ir posmi 0–5; pārējie, ja paliek laiks.

| # | ID | Saturs | Atkarīgs no | ~min |
|---|---|---|---|---|
| 0 | F0 | Karkass + pieslēgšanās + lomas | — | 20 |
| 1 | F1 | Organizācijas | F0 | 15 |
| 2 | F3 | Mijiedarbību žurnāls | F1 | 10 |
| 3 | F4 | Pieprasījumi | F1 | 10 |
| 4 | F5 | Uzdevumi | F1 | 10 |
| 5 | F9 | Vadības panelis | F1, F3, F4, F5 | 10 |
| 6 | F2 | Kontaktpersonas | F1 | 10 |
| 7 | F6 | Meklēšana | F1, F2, F4 | 10 |
| 8 | F7 | Pārskats vadītājam | F3, F4 | 10 |
| 9 | F8 | Piekļūstamības paziņojums | visi UI posmi | 10 |
| 10 | MI-1 | MI kopsavilkums | F3 | 15 |
| 11 | MI-2 | Tikai klasifikācija (bez koda) | — | 10 |

---

### 0 · F0 — karkass, pieslēgšanās, lomas

**Zars:** `f0-flask-karkass` · **Prasības:** `docs/F0.md`

1. `requirements.txt`: `flask`; `pip install -r requirements.txt`.
2. `app/__init__.py` (`create_app`), `app/db.py` (savienojums, `init-db`, `seed` komandas), `app/schema.sql`.
3. `app/templates/base.html`: `lang="lv"`, augšējā josla (vārds, loma, «Iziet»), «Pāriet uz saturu», kājenē saite uz `/pieklustamiba`.
4. `/login` un `/register` vienā lapā ar cilnēm (kā prototipā); paroles ar `werkzeug.security`; sesija; «Atcerēties mani» = `session.permanent`.
5. Reģistrācija vienmēr piešķir `darbinieks`; vadītājs tikai no sākumdatiem.
6. Dekoratori `login_required`, `role_required('vaditajs')`; palīgfunkcija `can_access_org(user, org_id)`.
7. Pēc pieslēgšanās pāradresē uz `/` (F9; līdz 5. posmam — vienkārša sveiciena lapa).

**Testi:** veiksmīga un neveiksmīga pieslēgšanās; kļūdas paziņojums neatklāj, kurš lauks nepareizs; reģistrācija → loma `darbinieks`; dublikāta e-pasts; nepieslēgts lietotājs → `/login`; «Iziet» notīra sesiju.

### 1 · F1 — organizācijas

**Zars:** `f1-organizacijas`

- `/organizacijas`: saraksts (tabula ar `caption`, `th scope`) + meklēšana pēc nosaukuma/reģ. nr.; darbiniekam tikai savas.
- `/organizacijas/jauna`, `/organizacijas/<id>/labot`: nosaukums, reģ. nr., veids (uzņēmums / NVO / pašvaldība), atbildīgais darbinieks. Servera validācija; kļūdas teksts pie lauka.
- `/organizacijas/<id>`: **kartīte** — pamatdati + vietas F2, F3, F4, F5 sadaļām. Katrs skatījums → ieraksts `access_log` (N2).

**Testi:** darbinieks nevar atvērt svešu organizāciju (403/404); vadītājs var; tukšs nosaukums → kļūda; skatījums rada `access_log` ierakstu.

### 2 · F3 — mijiedarbību žurnāls

**Zars:** `f3-mijiedarbibas`

- Forma kartītē: veids (zvans / e-pasts / sanāksme), datums, dalībnieki, kopsavilkums, nākamais solis.
- Kartītē hronoloģija, jaunākā augšā.

**Testi:** pievienošana; hronoloģiskā secība; nav piekļuves svešai organizācijai.

### 3 · F4 — pieprasījumi

**Zars:** `f4-pieprasijumi`

- Statusi `Jauns → Darbā → Atbildēts` (tikai uz priekšu); termiņš.
- Kavētie (termiņš < šodiena un nav «Atbildēts») ar **tekstu** «Kavēts N d.», ne tikai krāsu.
- `/pieprasijumi` saraksts ar filtru pēc statusa + sadaļa kartītē.

**Testi:** statusa pāreja; nepareiza pāreja noraidīta; kavējuma aprēķins; teksts «Kavēts» ir HTML.

### 4 · F5 — uzdevumi

**Zars:** `f5-uzdevumi`

- `/uzdevumi` — «Mani uzdevumi šodien»; atzīmēt kā izpildītu (POST forma, strādā bez JS).
- Uzdevums piesaistāms organizācijai vai pieprasījumam.

**Testi:** tikai paša uzdevumi; atzīmēšana; svešu uzdevumu nevar atzīmēt.

### 5 · F9 — vadības panelis

**Zars:** `f9-vadibas-panelis` · **Prasības:** `docs/F9.md`

- `/` pēc pieslēgšanās. Rādītāji: organizācijas, atvērtie, kavētie pieprasījumi, neizpildītie uzdevumi šodien.
- Bloki: «Mani uzdevumi šodien», «Pēdējās mijiedarbības» (5), «Atvērtie pieprasījumi pēc termiņa».
- Darbiniekam — tikai savas organizācijas; vadītājam — visas + «Aktivitāte pa darbiniekiem».
- Izkārtojums kā `prototips/index.html`.

**Testi:** skaitļi darbiniekam un vadītājam atšķiras pareizi; vadītāja bloks darbiniekam netiek renderēts.

### 6 · F2 — kontaktpersonas

**Zars:** `f2-kontaktpersonas`

- Lauki **tikai**: vārds, amats, e-pasts, tālrunis, saziņas pamats, pamata datums (VDAR minimizācija).
- `retain_until` = pamata datums + noteikts termiņš (konstante, piem., 3 gadi); kartītē redzams.
- «Dzēst pēc pieprasījuma» — neatgriezeniska dzēšana ar apstiprinājumu; žurnālā tikai ID.

**Testi:** obligātie lauki; dzēšana; žurnālā nav e-pasta/tālruņa.

### 7 · F6 — meklēšana

**Zars:** `f6-meklesana`

- Viens lauks augšējā joslā → `/meklet?q=` pāri organizācijām, kontaktiem, pieprasījumiem; rezultāti tabulā ar `caption` un `th scope`; ievēro lomu tiesības.

**Testi:** atrod katrā tipā; darbinieks neredz svešus rezultātus.

### 8 · F7 — pārskats vadītājam

**Zars:** `f7-parskats`

- `/parskats` tikai `vaditajs` (citiem 403): aktivitāte pa darbiniekiem, kavētie pieprasījumi, aktīvākās organizācijas.
- Diagramma (SVG serverī vai `<progress>`/CSS joslas) **un tā pati informācija tabulā**.

**Testi:** darbiniekam 403; tabulas skaitļi atbilst datiem.

### 9 · F8 — piekļūstamības paziņojums

**Zars:** `f8-pieklustamiba`

- Palaid axe (piem., `npx @axe-core/cli` vai pārlūka paplašinājums) galvenajā ceļā: pieslēgšanās → panelis → organizācija → mijiedarbība.
- Izlabo critical/serious atradumus. Lapā `/pieklustamiba` ieraksti **reālus** testa atradumus un datumu; neizdomā rezultātus. Ja axe nevar palaist, apstājies un ziņo lietotājam.

### 10 · MI-1 — mijiedarbības kopsavilkums

**Zars:** `mi-1-kopsavilkums`

- Pirms sākšanas ielādē `claude-api` skill un izvēlies aktuālo modeli.
- Pogas «Ģenerēt kopsavilkumu» → no brīvā teksta 3 teikumi + nākamā soļa ieteikums.
- Rezultāts **nekad netiek saglabāts automātiski**: darbinieks to rediģē un apstiprina; saglabā `ai_generated=1`, `approved_by`; kartītē marķējums «Ģenerēts ar MI».
- Atslēga no `ANTHROPIC_API_KEY`; ja tās nav — poga atspējota ar paskaidrojumu. Testos MI izsaukumu aizstāj ar viltojumu (mock).

### 11 · MI-2 — tikai klasifikācija

**Zars:** `mi-2-klasifikacija` · **Kodu neraksta.**

- `docs/MI-2-klasifikacija.md`: kas tiek vērtēts, kādi dati, kādas sekas organizācijai/personai, riska kategorija pēc MI akta, lēmums «būvēt / nebūvēt» ar pamatojumu. Galīgo lēmumu atstāj cilvēkam.

---

## Uzdevuma piemērs Claude Code

```
Īsteno posmu "1 · F1 — organizācijas" no @docs/izpildes-plans.md.
Izveido zaru f1-organizacijas, palaid make test, komita nosaukums sākas ar "F1:".
```

## Gatavības pārbaude katram posmam

- [ ] `make test` iziet
- [ ] Lomu tiesības pārbaudītas serverī un ar testu
- [ ] Formām `label`, kļūdas pie laukiem, tabulām `caption` + `th scope`
- [ ] Žurnālos nav personas datu
- [ ] `git diff --cached` pārskatīts; nav atslēgu
- [ ] Komits un PR sākas ar prasības ID
