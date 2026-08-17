# SETTI

Minimalistinen treenisovellus — yksi HTML-tiedosto, ei riippuvuuksia. Toimii selaimessa, tallentaa treenit paikallisesti.

## Ominaisuudet

- **Treenin logaus** — liikkeet, sarjat, painot ja toistot. Volyymi ja ennätykset lasketaan automaattisesti.
- **Millisekuntikello** — treenin kesto reaaliajassa (`H:MM:SS.mmm`).
- **Lepoajastin** — countdown, kesto valittavissa (1:00–3:00), "Valmiina" ajan päättyessä.
- **Treenipohjat** — tallenna treeni pohjaksi ja toista se myöhemmin.
- **Viikko-ohjelmointi** — kiinnitä pohja viikonpäiville, sovellus ehdottaa päivän treeniä.
- **Historia** — kaikki tallennetut treenit, volyymit ja sarjat.

## Käyttö

Avaa `index.html` selaimessa. Ei asennusta, ei palvelinta.

Tai käytä GitHub Pagesin kautta (Settings → Pages → Deploy from branch → `main` / root).

## Tallennus

Data tallentuu `window.storage`-rajapintaan; jos ei saatavilla, muistiin istunnon ajaksi. Ei ulkoisia palveluita, ei seurantaa.

## Tekniikka

Yksi tiedosto: HTML + CSS + vanilla JS. Fontit Google Fonts -importilla (Anton, Archivo, JetBrains Mono).
