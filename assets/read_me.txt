# Run

Egyszerű, mobilra optimalizált "ugrálós" böngészős játék (canvas alapú), beépített Spotify-lejátszóval és online ranglistával.

## Funkciók

- Ugrálós/futós játékmenet (koppintás vagy szóköz/fel nyíl az ugráshoz)
- Gyűjthető tárgyak és akadályok
- Testre szabható grafika (saját képek vagy automatikus, beépített helyettesítő rajz, ha a kép nem elérhető)
- Beágyazott Spotify-lejátszó, ami a játék indításakor automatikusan elindul
- Online ranglista (Cloudflare Worker + KV), ahol csak a legjobb eredményed marad meg
- Látogatószámláló (Cloudflare Worker)

---

## Fájlszerkezet

```
/
├── index.html              → maga a játék (HTML + CSS + JS egy fájlban)
├── leaderboard-worker.js   → Cloudflare Worker kód a ranglistához
└── assets/
    ├── bogi_200x200.png       → a futó szereplő képe
    ├── kaktusz_200x200.png    → az akadály képe
    └── mikrofon_200x200.png   → az ugrás közben gyűjthető tárgy képe
```

Ha valamelyik kép hiányzik az `assets` mappából, a játék automatikusan egy beépített, egyszerű SVG-rajzot használ helyette — tehát a játék kép nélkül is működik.

---

## Beállítható konstansok (`index.html`, a `<script>` eleje)

| Konstans | Mit csinál |
|---|---|
| `CHARACTER_IMG_SRC` | a futó karakter képének elérési útja |
| `OBSTACLE_IMG_SRC` | az akadály képének elérési útja |
| `COIN_IMG_SRC` | a gyűjthető tárgy képének elérési útja |
| `VISIT_COUNTER_URL` | a látogatószámláló Worker URL-je |
| `LEADERBOARD_URL` | a ranglista Worker URL-je |
| `SPOTIFY_URI` | melyik Spotify-tartalom szóljon a játék alatt |

### `SPOTIFY_URI` formátuma

```
spotify:track:XXXXXXXXXXXXX     → egy konkrét szám
spotify:album:XXXXXXXXXXXXX     → egy album
spotify:playlist:XXXXXXXXXXXXX  → egy playlist
spotify:artist:XXXXXXXXXXXXX    → egy előadó legnépszerűbb számai
```

Az `XXXXXXXXXXXXX` azonosítót a Spotify megosztási linkjéből lehet kimásolni, pl.:
`https://open.spotify.com/track/XXXXXXXXXXXXX?si=...` → csak az ID kell, a `?si=...` rész nem.

A zene csak az első "Indítás" gombnyomásra indul el (böngésző-korlátozás miatt automatikusan sosem tud elindulni), és "Újra" gombnál nem indul újra, amíg szól. Ha valaki manuálisan megállítja a lejátszóban, a következő gombnyomásra újraindul.

---

## Ranglista beüzemelése (Cloudflare Worker)

1. **Cloudflare Dashboard → Workers & Pages → Create → Create Worker**
2. Nevezd el (pl. `run-leaderboard`), majd a **Edit code** felületen illeszd be a `leaderboard-worker.js` tartalmát → **Deploy**
3. **Workers & Pages → KV** → hozz létre egy új KV namespace-t (pl. `bogirun_leaderboard_kv`)
4. A Worker **Settings → Variables → KV Namespace Bindings**-nél adj hozzá egy bindingot:
   - Variable name: **`LEADERBOARD`** (pontosan így, nagybetűvel)
   - KV namespace: az imént létrehozott namespace
5. Másold ki a Worker URL-jét (pl. `https://run-leaderboard.XXXX.workers.dev`), és írd be az `index.html`-ben a `LEADERBOARD_URL` konstansba

### Worker-viselkedés

- `GET /leaderboard` → visszaadja a top 10 eredményt
- `POST /score` `{ name, score }` → elmenti az eredményt
  - ha ugyanaz a név (kis-nagybetűtől függetlenül) már szerepel a listán, csak akkor frissül, ha a beküldött pontszám jobb a réginél
  - üres név esetén a Worker elutasítja a kérést (400-as hibával)
- legfeljebb 50 eredmény tárolódik el összesen, ebből a top 10 jelenik meg

---

## Ismert korlátok

- A pontszám kliens oldalon (böngészőben) számolódik, ezért technikailag hozzáértő valaki hamis eredményt tudna beküldeni közvetlenül a Worker API-nak, de session token van alkalmazva, illetve Cloudflare Worker oldalon a token kiadásától számítva maximális lehetséges pont kerül meghatározása, ennél magasabbat eldob mentés előtt.
- Ha két különböző ember ugyanazt a nevet adja meg, a legjobb eredményük egy sorba "olvad össze" a ranglistán. Célszerű instanevet kérni. 
