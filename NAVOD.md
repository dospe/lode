# Lodě na VPS (https://lode.pdostalek.cz)

Klasické lodě v prohlížeči: solo proti AI, nebo dva hráči v reálném čase přes
sdílený odkaz na místnost. Bez účtů, bez instalace. Zdroj:
[Hackatoan/battleship](https://github.com/Hackatoan/battleship) (Node.js +
Express + Socket.io, hotový image na GHCR).

Stejný model jako radar: porty 80/443 drží Caddy audiobookshelfu
(`/opt/audiobookshelf`). Hra nemá vlastní veřejný port, jen položí
`lode.caddy` do `/etc/caddy/sites.d` a svůj kontejner `lode` připojí do
sdílené docker sítě `web`.

```
internet ─443─> audiobookshelf-caddy ──(síť web)──> lode:3025
                  (HTTPS, certifikát)               (Express + Socket.io)
```

## 1. DNS
`lode.pdostalek.cz` A záznam → veřejná IP VPS. Na firewallu nic navíc.

**Cloudflare:** „DNS only“ (šedý mráček), stejně jako u radaru. Přes proxy by
hra sice fungovala (Cloudflare WebSockety umí), ale certifikát si Caddy
vyřídí sám a proxy nic nepřidá.

## 2. Předpoklad
Sdílené weby v Caddy audiobookshelfu už jsou zapnuté (krok 2 v NAVOD.md
radaru): existuje `/etc/caddy/sites.d` i docker síť `web`. Ověření:
```
docker network ls | grep web
ls /etc/caddy/sites.d/
```

## 3. VPS – hra
```
sudo mkdir -p /opt/lode && cd /opt/lode
# nakopíruj docker-compose.yml a caddy/lode.caddy
docker compose up -d
docker logs lode                                  # má vypsat, že poslouchá na 3025
```
Žádný `.env` není potřeba. Kontejner běží jako `nobody`, read-only, bez
capabilities, s limitem paměti. Kdyby po startu padal s `EACCES`, image se
změnil a potřebuje zapisovat – pak odeber řádek `user:` a `read_only:`.

Rychlý test zevnitř sítě `web` (ještě před Caddy):
```
docker run --rm --network web curlimages/curl -sI http://lode:3025/ | head -1   # -> HTTP/1.1 200 OK
```

## 4. Zveřejnění přes sdílený Caddy
```
sudo cp /opt/lode/caddy/lode.caddy /etc/caddy/sites.d/
docker exec audiobookshelf-caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
docker exec audiobookshelf-caddy caddy reload --config /etc/caddy/Caddyfile
docker logs -f audiobookshelf-caddy               # počkej na "certificate obtained" pro lode.pdostalek.cz
```
`validate` vždy před `reload` (stejný důvod jako u radaru: chybný soubor ve
`sites.d` by při restartu shodil i audiobookshelf). Hru odstraníš smazáním
`/etc/caddy/sites.d/lode.caddy`, reloadem a `docker compose down` v `/opt/lode`.

## 5. Ověření
Otevři https://lode.pdostalek.cz, zvol multiplayer → vznikne odkaz na
místnost. Otevři ho v druhém prohlížeči (nebo pošli kamarádovi) – oba
rozmístí lodě a hra začne. Pokud se druhý hráč nepřipojí, zkontroluj v
konzoli prohlížeče, zda projde WebSocket na `/socket.io/` (Caddy ho má
přeposílat automaticky).

## Žebříček (volitelné)
Hra umí sdílený žebříček přes Postgres (`DATABASE_URL`). Bez něj běží
normálně, jen bez žebříčku – pro hru s kamarády zbytečné. Přihlášení přes
Google v menu hry míří na Firebase projekt autora, u nás nic nedělá; dá se
ignorovat.

## Aktualizace
Stejný cron jako radar, jen jiná složka:
```
# /etc/cron.d/lode-update – každou neděli ve 4:40 stáhne nový image a restartuje
40 4 * * 0  root  cd /opt/lode && docker compose pull -q && docker compose up -d && docker image prune -f
```
Při aktualizaci se rozehrané partie ztratí (stav je jen v paměti serveru) –
proto neděle ve 4:40.

## Bezpečnostní shrnutí
- Kontejner bez portu na hostiteli, dosažitelný jen ze sítě `web`.
- Běží jako `nobody`, read-only FS, `cap_drop ALL`, `no-new-privileges`,
  limity paměti a procesů.
- HTTPS (Let's Encrypt, sdílený Caddy), HSTS, nosniff, bez rámování.
- Hra nemá účty ani heslo; místnost zná jen ten, kdo dostal odkaz. Server
  drží jen stav partií v paměti, nic neukládá na disk.
- Bez `DATABASE_URL` neodchází z VPS žádná data; s Google přihlášením by
  prohlížeč hráče kontaktoval Firebase/Google.
