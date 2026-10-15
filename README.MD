# V-Wing bugikorjaus
V-Wing pelin tultua freewareksi havaitsin tässä useita bugeja, joita koitan korjata.

Pelin nykyinen versiontunnus on `1.95 2026 FIX`. `FIX` erottaa tämän
korjatun version alkuperäisestä vuoden 2026 freeware-julkaisusta.

Havaitut ja tässä versiossa korjatut ongelmat:

- Äänet eivät toimi
- Yksinpeli ei toimi oikein
- Moninpeli kentän päättyminen kaataa pelin
- War Mode pelimuoto päättyy liian aikaisin (päättyy toisen pelaajan kuolemaan respawnin sijaan)
- Mission Mode ei käynnistä yhteistä turret- ja H-wing-haastetta moninpelissä

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

## 2026-09-22 – Moninpelin kentänvaihdon korjaus

### Korjattu ongelma

Moninpelissä kenttä ei päättynyt oikein, kenttä ei vaihtunut ja peli jatkui voittoehtojen täyttymisestä huolimatta. `GAME
LENGTH` -asetus ei muuttanut virheellistä käyttäytymistä.

### Toteutetut muutokset

- Yleisen moninpelin pelaajatilojen läpikäynti korjattiin säilyttäen
  alkuperäinen silmukan eteneminen ja pelaajamäärän vertailu.
- Voittoruudulta poistuminen ohjattiin olemassa olevaan pelin
  siivous- ja jatkopolkuun.
- Muutos rakennettiin puhtaasti GUS- ja yksinpelikorjaukset sisältävästä
  versiosta.
- Lopullisessa tiedostossa muuttui 16 tavupaikkaa edelliseen versioon
  verrattuna. Tiedoston koko säilyi ennallaan.

### Testaus

- Moninpeli siirtyy kierroksen ratkettua seuraavaan kenttään oikein.
- `GAME LENGTH` vaikuttaa ottelun pituuteen odotetusti.
- Ottelun viimeisen kierroksen jälkeen näytetään voittoruutu ja palataan
  päävalikkoon pelin sulkeutumisen tai jumittumisen sijasta.
- Yksinpelin aiemmat korjaukset säilyivät toimivina.

## 2026-09-22 – War Moden voittopolun korjaus

### Pelimuodon tausta

War Mode ja Mission eivät olleet mukana V-Wingin alkuperäisessä julkaistussa
versiossa. Ne ovat Simonin peliin myöhemmin tekemiä lisäyksiä, oletettavasti
noin 30 vuoden takaa.

Tuolloin tutkitun binäärin ja pelissä havaittavan käyttäytymisen perusteella tilat
eivät ilmeisesti ehtineet valmiiksi tai kattavasti testatuiksi julkaisua
varten. War Modessa on kuitenkin huomattavasti omaa pelilogiikkaa. Mission
puolestaan näytti lähinnä keskeneräiseltä rungolta,
josta ei ollut löytynyt erillistä tehtävä- tai voittologiikkaa. Myöhempi
Mission-tutkimus ja korjaus kuvataan alla. Simo kuvaa
näitä muutoksia VWINGFreeware.txt tiedostossa hieman.

### Korjattu ongelma

War Modessa pelaajan kuoltua peli aloittaa pitkän kloonin
herättämisvaiheen, jonka aikana ruudulla lukee `WAKING THE CLONE`.
Toisen pelaajan laskeutuminen baselle tämän vaiheen aikana käynnisti
yleisen moninpelin voittopolun. Kierros päättyi silloin ennen kuin kuolleen
pelaajan oli tarkoitus palata peliin.

### Toteutetut muutokset

- War Moden kloonin heräämistila ohittaa yleisen moninpelin
  voittajatarkistuksen.
- Tarkistus jatkaa olemassa olevaan War Moden omaan aluehallintaan
  perustuvaan voittotarkistukseen.
- Yksinpeli sekä muut moninpelitilat jatkavat aiemmin korjattuihin
  voittopolkuihinsa.
- Korjausta varten lisättiin yksi LE-loaderin fixup-tietue.
- 26 tavun apurutiini sijoitettiin ohjelmaosion lopun käyttämättömään
  täytetilaan. Ohjelmasivujen määrä, tiedoston koko ja muiden osioiden
  sijainnit eivät muuttuneet.
- Kehityksen aikaisia diagnostiikkatekstejä tai aikaisempia epäonnistuneita
  kokeita ei sisälly lopulliseen versioon.

### Testaus

- Baselle meneminen ei enää anna voittoa `WAKING THE CLONE` -vaiheen
  aikana.
- Kaikki pelimuodot testattiin eikä niissä havaittu uusia regressioita.
- Binäärin LE-rakenne, sivukartta ja fixup-taulukko tarkistettiin.
  Fixup-taulukosta ei kadonnut eikä muuttunut asiaankuulumattomia tietueita.
- Missioniin ei tehty muutoksia.

## 2026-10-16 – Mission Moden haastepolun palautus

### Korjattu ongelma

Mission käytti usealla pelaajalla tavallisen moninpelin alustus- ja
voittopolkuja. Yksinpelin turret- ja H-wing-haasteen toimintoja jäi tämän
vuoksi saavuttamattomiksi.

Aikaisempien Git-versioiden binäärivertailu osoitti, että yksinpelikorjauksen
seitsemän muutoskohtaa olivat aiemmin tarkistaneet pelimuotoa. Mission
käytti siis samoja haastepolkuja, jotka yksinpelikorjaus rajasi yhden
pelaajan käyttöön. Nyt tarkistukset hyväksyvät molemmat tilanteet:
`Players == 1` tai `Game Mode == Mission`.

### Toteutetut muutokset

- Mission käynnistää 2–4 pelaajalla olemassa olevan haastepelin. Tavoitteena
  on tuhota turretteja ja H-wingejä yhdessä.
- Päävalikon käynnistys, kentän haastealustus, bonuksen päivitys ja haasteen
  päättymistarkistukset käyttävät yhteistä yksinpeli-/Mission-ehtoa.
- Turrettien ja H-wingien tuhot kartuttavat yhteisiä tuholaskureita ja
  pisteitä. Yksi tuhottu turret kasvattaa laskuria yhdellä pelaajamäärästä
  riippumatta.
- Mission käyttää olemassa olevaa stage-etenemistä ja tulosruutua.
  Taukoruudulla näkyvät `STAGE`, `TURRETS`, `H-WINGS`, `BONUS` ja `SCORE`.
- Yhden pelaajan haastepolku, Normal-moninpeli ja War Moden aikaisempi
  voittopolun korjaus säilytettiin.
- Pelaajien keskinäistä vahinkoa ei muutettu.

Korjaus rakennettiin War Mode -korjauksen sisältävästä versiosta
(`ffd4abc`). Seitsemään kohtaan lisättiin kutsut yhteiseen ehtotarkistukseen
ja sen apurutiineihin. Apukoodi vie 46 tavua ohjelmaosion lopun
käyttämättömästä täytetilasta. LE-loaderille lisättiin yksi fixup-tietue;
kaikki 25 345 vanhaa tietuetta säilyivät ennallaan. Tiedostokoko on edelleen
585 267 tavua, eivätkä objektien koot tai datasivujen sijainnit muuttuneet.

### Pakeneminen ja asetukset

Alkuperäinen eject-/base-sääntö säilytettiin: yhden pelaajan eject ja
baselle pääsy voi näyttää `ESCAPED!` ja siirtää yhteisen pelin seuraavaan
kenttään, vaikka muut pelaajat voisivat vielä jatkaa taistelua. Kaikkien
pelaajien yhteistä pakenemisehtoa ei lisätty.

Haastepeli asettaa omat sääntö- ja vaikeusasetuksensa yksinpelin tavoin.
Missionin tilastojen jatkuvaa piirtoa tai basen korjauksen pistevähennystä
ei laajennettu moninpelille; taukoruudun tilastot ovat käytössä.

### Testaus

- Muutetuille x86-koodijaksoille suoritettiin 768 onnistunutta tarkistusta.
  Mukana olivat pelaajamäärät 1–4, kaikki kolme pelimuotoa ja War-tilalipun
  molemmat arvot kahdella eri objektien muistiasettelulla.
- Tarkistettiin haarakohteet, rekisterien säilyminen, pinon tasapaino,
  turretin tuholaskuri ja pisteet sekä H-wingin määrän vähennys,
  tuholaskuri ja pisteet.
- LE-fixupit ja binäärin muutosalueet tarkistettiin erikseen.
- Käyttäjä kokeili Missionia pelissä ja hyväksyi toiminnan nykyisellä
  eject-säännöllä. Testattu `VWTEST.EXE` otettiin uudeksi `VWINGFIX.EXE`:ksi.

Mission-korjauksen jälkeisen version SHA-256:
`efd17229edbb41cd38fbcec090000ad4f433146fd75d27695c8c16e37bfc0a5f`.

## 2026-10-16 – Korjatun version tunnus

Versionumeroksi vaihdettiin `1.95 2026 FIX` DOS-käynnistystekstissä,
päävalikossa ja alun tekstivierityksessä. Päävalikon pidempi versionrivi
keskitettiin uudelleen. Alkuperäiset tekijä- ja copyright-merkinnät
säilytettiin.

Kaksi pidempää tekstiä sijoitettiin ohjelmaosion käyttämättömään
täytetilaan ja niiden kaksi LE-fixup-kohdetta päivitettiin. Vieritysteksti
mahtui alkuperäiseen tilaansa. Tiedostokoko ja sivumäärät säilyivät
ennallaan, eikä pelilogiikan käskyihin tehty muutoksia. Kaikki 25 346
fixup-tietuetta tarkistettiin ja tekstien osoitteiden toiminta varmistettiin
kahdella objektien muistiasettelulla. Tekstien leveydet tarkistettiin
pelin käyttämien 8 ja 16 pikselin merkkileveyksien mukaan.

Nykyisen `VWINGFIX.EXE`:n SHA-256:
`0ce98d061216d25ab9cfe75c98deef48842c1b88c2e58bd6fcea2961be7215da`.

## Korjausten teko

Korjauksissa on hyödynnetty todella paljon tekoälyä. Manuaalisesti näiden korjaaminen olisi lähes mahdotonta alkuperäisen lähdekoodin puuttuessa.
