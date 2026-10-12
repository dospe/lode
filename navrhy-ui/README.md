# Návrhy nového UI hry (říjen 2026)

Statické mockupy tří variant vzhledu hry bez patičky a odkazů, s kódy polí
na pozadí a odlišnou barvou potopených lodí. Nejde o funkční hru – jen o
předlohu pro úpravu forku `dospe/battleship` (public/index.html, style.css,
game.js).

- `soucasny-stav.png` – současný vzhled pro srovnání
- `navrh-A.png` – „Námořní“ (tmavá, vývoj současného vzhledu)
- `navrh-B.png` – „Radar“ (zelený terminál, kódy uprostřed polí, bez vnějších popisků)
- `navrh-C.png` – „Papír“ (světlá, tužka a papír)
- `mock.html` – zdroj mockupů; otevřít v prohlížeči s `?v=A|B|C&screen=game|menu`

**Zvoleno:** A jako tmavý režim + C jako světlý režim, přepínatelné tlačítkem
vpravo nahoře. Implementováno ve forku `dospe/battleship`, větev `ui-cz`.
