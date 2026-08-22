# TORCH

Treenipäiväkirja ja treenikaverit. Yksi HTML-tiedosto, ei riippuvuuksia, ei build-vaihetta. Toimii selaimessa ja tallentaa treenit paikallisesti.

**Live:** https://lehtisenelmeri.github.io/Setti/

## Ominaisuudet

- **Treenin kirjaus** - liikkeet, sarjat, painot ja toistot. Volyymi ja arvioitu 1RM lasketaan automaattisesti.
- **Lepoajastin** - countdown-rengas sarjojen välissä, kesto säädettävissä.
- **Kardio** - sekuntikello (aloita, pysäytä, jatka) painojen sijaan.
- **Splitit** - valmiit mallit (PPL, Upper/Lower, Arnold, Bro, Full Body) tai oma. Kiinnitä liikkeet viikonpäiville, sovellus ehdottaa päivän treenin.
- **Kehitys** - kuukausikalenteri, eniten kehittynyt liike, lihastasapaino.
- **Kaverit** - aktiviteettifeedi, treenistriikit, salikutsut. Demodatalla; backend-sauma on `Social`-objektissa.
- **Kuva treenistä** - sisäänrakennettu kamera (etu/taka, zoom, salama) tai tiedostovalinta.
- **Varmuuskopio** - vie ja tuo kaikki data JSON-tiedostona.

## Käyttö

Avaa `index.html` selaimessa. Ei asennusta, ei palvelinta.

Kamera vaatii HTTPS-origin, eli se toimii vain julkaistussa osoitteessa tai oikealla puhelimella, ei `file:`-protokollalla.

## Tallennus

`localStorage`, avainprefix `setti.*` (historiallinen, säilytetty jotta vanhat treenit eivät katoa). Ei ulkoisia palveluita, ei seurantaa.

## Tekniikka

Yksi tiedosto: HTML + CSS + vanilla JS. Ei kirjastoja, ei framework-riippuvuuksia. Fontit Google Fontsista (Barlow Condensed, Figtree).

Asennettavissa kotinäytölle: PWA-manifesti ja ikoni ovat inline `data:`-URI:na.
