# Zgradite svoj strežnik za pošiljanje pošte SMTP

## preambula

SMTP lahko neposredno kupi storitve od prodajalcev oblakov, kot so:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Ali potisna e-pošta v oblak](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

Prav tako lahko zgradite svoj poštni strežnik - neomejeno pošiljanje, nizki skupni stroški.

Spodaj vam korak za korakom pokažemo, kako zgraditi lasten poštni strežnik.

## Izbira strežnika

Samostojni strežnik SMTP zahteva javni IP z odprtimi vrati 25, 456 in 587.

Običajno uporabljeni javni oblaki so ta vrata privzeto blokirali in morda jih je mogoče odpreti z izdajo delovnega naloga, vendar je navsezadnje zelo težavno.

Priporočam nakup pri gostitelju, ki ima ta vrata odprta in podpira nastavitev povratnih imen domen.

Tukaj priporočam [Contabo](https://contabo.com) .

Contabo je ponudnik gostovanja s sedežem v Münchnu v Nemčiji, ustanovljen leta 2003 z zelo konkurenčnimi cenami.

Če kot valuto nakupa izberete evro, bo cena nižja (strežnik z 8 GB pomnilnika in 4 procesorji stane približno 529 juanov na leto, začetna namestitev pa je brezplačna eno leto).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Pri oddaji naročila navedite `prefer AMD` in strežnik s CPE AMD bo imel boljšo zmogljivost.

V nadaljevanju bom vzel Contabojev VPS kot primer, da pokažem, kako zgraditi lasten poštni strežnik.

## Konfiguracija sistema Ubuntu

Operacijski sistem tukaj je Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Če strežnik na ssh prikaže `Welcome to TinyCore 13!` (kot je prikazano na spodnji sliki), to pomeni, da sistem še ni bil nameščen. Prekinite povezavo ssh in počakajte nekaj minut, da se znova prijavite.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Ko se pojavi `Welcome to Ubuntu 22.04.1 LTS` , je inicializacija končana in lahko nadaljujete z naslednjimi koraki.

### [Izbirno] Inicializirajte razvojno okolje

Ta korak ni obvezen.

Zaradi udobja sem namestitev in sistemsko konfiguracijo programske opreme ubuntu postavil na [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Zaženite naslednji ukaz za namestitev z enim klikom.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Kitajski uporabniki, namesto tega uporabite naslednji ukaz in jezik, časovni pas itd. bodo samodejno nastavljeni.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo omogoča IPV6

Omogočite IPV6, da lahko SMTP pošilja tudi e-pošto z naslovi IPV6.

uredi `/etc/sysctl.conf`

Spremenite ali dodajte naslednje vrstice

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Nadaljujte z [vadnico contabo: Dodajanje povezave IPv6 v vaš strežnik](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Uredite `/etc/netplan/01-netcfg.yaml` , dodajte nekaj vrstic, kot je prikazano na spodnji sliki (privzeta konfiguracijska datoteka Contabo VPS že vsebuje te vrstice, samo odkomentirajte jih).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Nato `netplan apply` , da bo spremenjena konfiguracija začela veljati.

Ko je konfiguracija uspešna, lahko uporabite `curl 6.ipw.cn` za ogled naslova ipv6 vašega zunanjega omrežja.

## Klonirajte ops konfiguracijskega repozitorija

```
git clone https://github.com/wactax/ops.soft.git
```

## Ustvarite brezplačen SSL certifikat za vaše ime domene

Pošiljanje pošte zahteva potrdilo SSL za šifriranje in podpisovanje.

Za ustvarjanje potrdil uporabljamo [acme.sh.](https://github.com/acmesh-official/acme.sh)

acme.sh je odprtokodno avtomatizirano orodje za podpisovanje potrdil,

Vnesite konfiguracijsko skladišče ops.soft, zaženite `./ssl.sh` in mapa `conf` bo ustvarjena v **zgornjem imeniku** .

Poiščite svojega ponudnika DNS v [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , uredite `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Nato zaženite `./ssl.sh 123.com` , da ustvarite potrdila `123.com` in `*.123.com` za vaše ime domene.

Prvi zagon bo samodejno namestil [acme.sh](https://github.com/acmesh-official/acme.sh) in dodal načrtovano nalogo za samodejno obnovitev. Vidite lahko `crontab -l` , taka vrstica je naslednja.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Pot za ustvarjeno potrdilo je nekaj podobnega `/mnt/www/.acme.sh/123.com_ecc。`

Obnova potrdila bo poklicala skript `conf/reload/123.com.sh` , uredite ta skript, lahko dodate ukaze, kot je `nginx -s reload` , da osvežite predpomnilnik potrdil sorodnih aplikacij.

## Zgradite strežnik SMTP s chasquid

[chasquid](https://github.com/albertito/chasquid) je odprtokodni strežnik SMTP, napisan v jeziku Go.

Kot nadomestek za starodavne poštne programe, kot sta Postfix in Sendmail, je chasquid preprostejši in enostavnejši za uporabo, lažji pa je tudi za sekundarni razvoj.

Zaženi `./chasquid/init.sh 123.com` bo nameščen samodejno z enim klikom (zamenjaj 123.com z imenom domene pošiljatelja).

## Konfigurirajte DKIM za e-poštni podpis

DKIM se uporablja za pošiljanje e-poštnih podpisov, da se pisma ne obravnavajo kot neželena pošta.

Ko se ukaz uspešno izvede, boste pozvani, da nastavite zapis DKIM (kot je prikazano spodaj).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Samo dodajte zapis TXT v svoj DNS (kot je prikazano spodaj).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Oglejte si stanje storitve in dnevnike

 `systemctl status chasquid` Oglejte si status storitve.

Stanje normalnega delovanja je prikazano na spodnji sliki

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` ali `journalctl -xeu chasquid` si lahko ogleda dnevnik napak.

## Povratna konfiguracija imena domene

Obratno ime domene omogoča, da se naslov IP razreši v ustrezno ime domene.

Če nastavite ime obratne domene, lahko preprečite, da bi bila e-poštna sporočila prepoznana kot vsiljena pošta.

Ko je pošta prejeta, bo prejemni strežnik izvedel analizo povratnega imena domene na naslovu IP strežnika pošiljatelja, da potrdi, ali ima strežnik pošiljatelj veljavno ime povratne domene.

Če strežnik pošiljatelj nima imena povratne domene ali če se ime povratne domene ne ujema z naslovom IP strežnika pošiljatelja, lahko sprejemni strežnik e-pošto prepozna kot vsiljeno pošto ali jo zavrne.

Obiščite [https://my.contabo.com/rdns](https://my.contabo.com/rdns) in konfigurirajte, kot je prikazano spodaj

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Po nastavitvi povratnega imena domene ne pozabite konfigurirati posredovane ločljivosti imena domene ipv4 in ipv6 na strežnik.

## Uredite ime gostitelja chasquid.conf

Spremenite `conf/chasquid/chasquid.conf` na vrednost imena povratne domene.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Nato zaženite `systemctl restart chasquid` , da znova zaženete storitev.

## Varnostno kopirajte conf v repozitorij git

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Na primer, mapo conf varnostno kopiram v svoj proces github, kot sledi

Najprej ustvarite zasebno skladišče

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Vnesite imenik conf in oddajte v skladišče

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Dodaj pošiljatelja

teči

```
chasquid-util user-add i@wac.tax
```

Lahko doda pošiljatelja

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Preverite, ali je geslo pravilno nastavljeno

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Po dodajanju uporabnika bo `chasquid/domains/wac.tax/users` posodobljen, ne pozabite ga poslati v skladišče.

## DNS dodaj zapis SPF

SPF (Sender Policy Framework) je tehnologija za preverjanje e-pošte, ki se uporablja za preprečevanje goljufij po e-pošti.

Preveri identiteto pošiljatelja e-pošte tako, da preveri, ali se naslov IP pošiljatelja ujema z zapisi DNS imena domene, za katerega se predstavlja, s čimer prepreči prevarantom pošiljanje lažnih e-poštnih sporočil.

Če dodate zapise SPF, lahko čim bolj preprečite, da bi bila e-poštna sporočila prepoznana kot vsiljena pošta.

Če vaš domenski imenski strežnik ne podpira vrste SPF, preprosto dodajte zapis vrste TXT.

Na primer, SPF za `wac.tax` je naslednji

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF za `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Upoštevajte, da imam tukaj `include:_spf.google.com` , to je zato, ker bom pozneje konfiguriral `i@wac.tax` kot naslov za pošiljanje v Googlovem nabiralniku.

## DNS konfiguracija DMARC

DMARC je okrajšava za (Domain-based Message Authentication, Reporting & Conformance).

Uporablja se za zajemanje odklonov SPF (morda zaradi konfiguracijskih napak ali pa se nekdo drug pretvarja, da ste vi, da bi poslal vsiljeno pošto).

Dodajte zapis TXT `_dmarc` ,

Vsebina je naslednja

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

Pomen vsakega parametra je naslednji

### p (Politika)

Označuje, kako ravnati z e-pošto, ki ne prestane preverjanja SPF (Sender Policy Framework) ali DKIM (DomainKeys Identified Mail). Parameter p je mogoče nastaviti na eno od treh vrednosti:

* brez: Izvedeno ni bilo nobeno dejanje, samo rezultat preverjanja se vrne pošiljatelju prek mehanizma za poročanje po e-pošti.
* Karantena: pošto, ki ni prestala preverjanja, postavite v mapo z vsiljeno pošto, vendar je ne boste neposredno zavrnili.
* zavrni: Neposredno zavrnite e-poštna sporočila, ki ne prestanejo preverjanja.

### fo (možnosti napake)

Podaja količino informacij, ki jih vrne mehanizem poročanja. Nastavite ga lahko na eno od naslednjih vrednosti:

* 0: Poročilo o rezultatih preverjanja za vsa sporočila
* 1: Prijavite samo sporočila, ki ne prestanejo preverjanja
* d: poročajte samo o napakah pri preverjanju imena domene
* s: poročajte samo o napakah pri preverjanju SPF
* l: Sporoči samo napake pri preverjanju DKIM

### rua & ruf

* rua (URI poročanja za združena poročila): E-poštni naslov za prejemanje združenih poročil
* ruf (URI poročanja za forenzična poročila): e-poštni naslov za prejemanje podrobnih poročil

## Dodajte zapise MX za posredovanje e-pošte v Google Mail

Ker nisem mogel najti brezplačnega službenega poštnega predala, ki podpira univerzalne naslove (Catch-All, lahko prejema vsa e-poštna sporočila, poslana na to ime domene, brez omejitev glede predpon), sem uporabil chasquid za posredovanje vseh e-poštnih sporočil v svoj nabiralnik Gmail.

**Če imate svoj plačljivi poslovni nabiralnik, ne spreminjajte MX in preskočite ta korak.**

Uredite `conf/chasquid/domains/wac.tax/aliases` , nastavite poštni predal za posredovanje

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` označuje vsa e-poštna sporočila, `i` je predpona e-poštnega naslova uporabnika pošiljatelja, ustvarjenega zgoraj. Za posredovanje pošte mora vsak uporabnik dodati vrstico.

Nato dodajte zapis MX (tukaj pokažem neposredno na naslov imena povratne domene, kot je prikazano v prvi vrstici na spodnji sliki).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Ko je konfiguracija končana, lahko uporabite druge e-poštne naslove za pošiljanje e-pošte na `i@wac.tax` in `any123@wac.tax` , da preverite, ali lahko prejemate e-pošto v Gmailu.

Če ne, preverite dnevnik chasquid ( `grep chasquid /var/log/syslog` ).

## Pošljite e-pošto na i@wac.tax z Google Mail

Ko je Google Mail prejel pošto, sem seveda upal, da bom odgovoril z `i@wac.tax` namesto i.wac.tax@gmail.com.

Obiščite [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) in kliknite »Dodaj drug e-poštni naslov«.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Nato vnesite kodo za preverjanje, ki ste jo prejeli po e-pošti, ki je bila posredovana na.

Končno ga lahko nastavite kot privzeti naslov pošiljatelja (skupaj z možnostjo odgovora z istim naslovom).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

Na ta način smo zaključili vzpostavitev poštnega strežnika SMTP in hkrati uporabljamo Google Mail za pošiljanje in prejemanje elektronske pošte.

## Pošljite testno e-pošto, da preverite, ali je konfiguracija uspešna

Vnesite `ops/chasquid`

Zaženi `direnv allow` namestitev odvisnosti (direnv je bil nameščen v prejšnjem procesu inicializacije z enim ključem in lupini je bil dodan kavelj)

potem teči

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

Pomen parametrov je naslednji

* uporabnik: uporabniško ime SMTP
* pass: geslo SMTP
* za: prejemnik

Lahko pošljete testno e-pošto.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Priporočljivo je, da uporabite Gmail za prejemanje testnih e-poštnih sporočil, da preverite, ali so konfiguracije uspešne.

### Standardno šifriranje TLS

Kot je prikazano na spodnji sliki, obstaja ta majhna ključavnica, kar pomeni, da je bilo potrdilo SSL uspešno omogočeno.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Nato kliknite »Pokaži izvirno e-pošto«

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Kot je prikazano na spodnji sliki, izvirna poštna stran Gmaila prikazuje DKIM, kar pomeni, da je konfiguracija DKIM uspešna.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Preverite Prejeto v glavi izvirnega e-poštnega sporočila in videli boste, da je naslov pošiljatelja IPV6, kar pomeni, da je tudi IPV6 uspešno konfiguriran.
