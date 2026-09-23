# V-Wing bugikorjaus
V-Wing pelin tultua freewareksi havaitsin tässä useita bugeja, joita koitan korjata.
havaitut bugit:
- Äänet eivät toimi
- Yksinpeli ei toimi oikein
- Moninpeli kentän päättyminen kaataa pelin
- War Mode pelimuoto päättyy liian aikaisin (päättyy toisen pelaajan kuolemaan respawnin sijaan)
- Mission Mode näyttää olevan keskeneräinen. vaikea sanoa, mikä sen pitäisi olla

# VWINGFIX.EXE – muutosloki

## 2026-09-14 – Gravis UltraSound -samplelatauksen korjaus

Korjaus vuoden 2026 V-Wing-freeware-julkaisun `VWING.EXE`:hen.

### Korjattu ongelma

Alkuperäinen ohjelma tunnisti Gravis UltraSoundin ja luki `SNDADR.DAT`:n,
mutta jätti varsinaiset `.SND`-samplet lataamatta äänikortin muistiin.
Äänentoistorutiinit toimivat, mutta oikeiden äänien sijaan kuului napsahduksia
tai tyhjää sampledataa.

`VWINGFIX.EXE` lataa samplet automaattisesti ennen alkuperäisen DOS4GW- ja
pelikoodin käynnistymistä. Lataus on integroitu samaan EXE-tiedostoon.

### Toteutetut muutokset

- Lisätty 16-bittiseen DOS-käynnistysosaan sisäinen GUS-sampleloader.
- GUS-perusportti luetaan `ULTRASND`-ympäristömuuttujan ensimmäisestä,
  heksadesimaalisesta kentästä. Sama EXE tukee portteja `220h` ja `240h`.
- Kaikki 16 samplea ladataan referenssiloaderin mukaisessa järjestyksessä:
  SHOOT, GRASH, KABOOM, SPLASH, ZIU, MISSILE, BUM, BIRD, ALARM, ARGH,
  ELECTR, CHOP, YEAH, FIUU, FIRE ja SONIC.
- GUS-muistiosoitteet ja 32-bittiset samplepituudet luetaan `SNDADR.DAT`:sta.
  Tiedostojen fyysistä kokoa ei käytetä latauspituutena. Esimerkiksi
  `KABOOM.SND`:stä ladataan 32 000 tavua, vaikka tiedoston koko on 70 618 tavua.
- Sampledata luetaan enintään 4 096 tavun paloissa ja kirjoitetaan GF1:n
  DRAM-muistiin PIO-kirjoituksilla. Jokaiselle tavulle tehdään XOR `80h`
  -muunnos GUSin käyttämään etumerkilliseen samplemuotoon.
- Lisätty virhepolut puuttuvalle tai virheelliselle `ULTRASND`-arvolle,
  puuttuville tiedostoille, vajaille lukutuloksille, lukuvirheille ja
  virheellisille GUS-muistialueille. Loader keskeyttää oman latauksensa,
  sulkee avatun tiedoston ja jatkaa alkuperäiseen käynnistykseen.
- Rekisterit, segmenttirekisterit ja FLAGS palautetaan ennen jatkamista.
  Loader ei muuta DOSin DTA-osoitetta, työhakemistoa eikä ympäristöä.

### EXE-rakenteen muutokset

Korjaus rakennettiin suoraan alkuperäisestä `VWING.EXE`:stä. Alkuperäinen
tiedosto säilytettiin muuttumattomana.

| Kohde | Alkuperäinen | Korjattu |
|---|---:|---:|
| Koko tiedosto | 572 979 tavua | 585 267 tavua |
| MZ-osa otsakkeineen, ilman LE:tä edeltävää täytettä | 11 404 tavua | 23 692 tavua |
| MZ-latauskuva ilman 96 tavun otsaketta | 11 308 tavua | 23 596 tavua |
| `e_ip` | `0x038C` | `0x3600` |
| `e_cp` | 23 | 47 |
| `e_cblp` | 140 | 140 |
| `e_lfanew` | `0x2C90` | `0x5C90` |
| LE-datasivujen absoluuttinen tiedosto-osoite | `0x33A00` | `0x36A00` |

Loader alkaa tiedosto-osoitteesta `0x3660`. Sen koodi, data ja puskuri vievät
yhteensä 4 916 tavua, ja ne sijaitsevat alkuperäisen pinon yläpuolella.

LE-osa siirrettiin tiedostossa `0x3000` tavua eteenpäin. LE-otsakkeessa
muutettiin vain datasivujen absoluuttinen tiedosto-osoite. Suojatun tilan
pelikoodi, pelidata, objektitaulu, sivukartta ja fixup-taulut säilyivät
tavuntarkasti alkuperäisinä. Myös MZ:n kuusi relocation-merkintää sekä
alkuperäiset `CS`, `SS` ja `SP` säilyivät ennallaan.

Loader palauttaa tallennetun prosessoritilan ja siirtyy lähihypyllä
alkuperäiseen `CS:038C`-käynnistysosoitteeseen.

### Käyttö ja yhteensopivuus

Korjattu EXE voidaan kopioida alkuperäisestä ZIP-paketista purettuun
pelikansioon. Sen voi käynnistää nimellä `VWINGFIX.EXE` tai nimetä
`VWING.EXE`:ksi alkuperäisen tilalle.

Uusia ajonaikaisia lisätiedostoja ei tarvita. Alkuperäisen paketin
`DOS4GW.EXE`, `SNDADR.DAT`, `.SND`-tiedostot ja muut pelitiedostot tarvitaan
edelleen. Erillistä COM-loaderia, wrapperia, käynnistysskriptiä tai
rakennustyökaluja ei tarvita. GUSin on oltava käytössä ja `ULTRASND`:n
vastattava sen asetuksia.

## 2026-09-16 – Yksinpelitilan palautus

### Korjattu ongelma

Vuoden 2026 freeware-version päävalikossa yhden pelaajan valinta käynnisti
tavallisen moninpelitilan yhdellä V-wingillä. Pelistä puuttuivat tällöin
yksinpelin aloitusinfo, varsinainen turret- ja H-wing-haaste, vihollisten
seuranta sekä yksinpelin oikeat kentän päättymisehdot.

Yksinpelin toteutus oli edelleen kokonaisuudessaan EXE-tiedostossa, mutta
viisi sen ohjauspolun tarkistusta luki väärää tilakenttää pelaajamäärän
sijasta. Tämän vuoksi yksinpelirutiini ja sen sisäiset päivitys- ja
voittotarkistukset jäivät osittain saavuttamattomiksi.

### Toteutetut muutokset

- Päävalikon `START GAME` -haara tarkistaa nyt arvon `Players == 1` ja
  kutsuu olemassa olevaa yksinpelirutiinia.
- Kentän yksinpelialustus käyttää samaa pelaajamäärän tarkistusta.
- Yksinpelin tilanne- ja vihollislaskurien päivitys aktivoituu yhdellä
  pelaajalla.
- `AndTheWinnerIs()` siirtyy yhdellä pelaajalla omaan yksinpelin
  voittotarkistukseensa moninpelin voittajalogiikan sijasta.
- Yksinpelin sisäinen kentän päättymistarkistus käyttää pelaajamäärää.
- Turretin tuhoutuminen kasvattaa yksinpelissä `KilledTurrets`-laskuria ja
  lisää tuhoutumisesta saatavat pisteet.
- H-wingin tuhoutuminen kasvattaa yksinpelissä `KilledHWings`-laskuria ja
  lisää tuhoutumisesta saatavat pisteet.
- LE-loaderin fixup-tietueet päivitettiin vastaamaan uusia
  `Players`-viittauksia. Loader ei siten palauta vanhoja osoitteita ohjelman
  käynnistyessä.

Korjaus muuttaa 38 tavua seitsemässä loogisessa kohdassa. Tiedoston koko ja
LE-rakenne eivät muutu. Muutos rakennettiin puhtaasti aiemmasta
GUS-korjatusta `VWINGFIX.EXE`:stä; kehityksen aikaiset diagnostiset base- ja
voittopolkujen ohitukset eivät sisälly lopulliseen tiedostoon.

### Testaus

- Yhden pelaajan tila näyttää ennen kenttää turret-, vaikeusaste- ja muut
  yksinpelitiedot.
- Yksinpelissä turretit ja H-wingit ilmestyvät ja toimivat.
- Kenttä ei pääty baselle menemisestä, jos yksinpelin vihollisia on vielä
  jäljellä.
- Loppuruutu näyttää tuhottujen turrettien ja H-wingien määrät oikein.
  Samalla korjattiin näihin tuhoamisiin liittyvä pistekertymä.
- Moninpeli käynnistyy edelleen normaalisti.

### Korjausten teko

Korjauksissa on hyödynnetty todella paljon tekoälyä. Manuaalisesti näiden korjaaminen olisi lähes mahdotonta alkuperäisen lähdekoodin puuttuessa. 