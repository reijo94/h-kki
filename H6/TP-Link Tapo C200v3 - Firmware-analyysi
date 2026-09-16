# TP-Link Tapo C200v3 - Firmware-analyysi ja root-pääsyn tutkiminen

**Kohdelaite:** TP-Link Tapo C200v3 (verkkokamera)
**Firmware-versio:** 1.4.2 Build 250313 Rel.40499n
**Ympäristö:** Kali Linux -terminaali
**Työkalut:** `tp-link-decrypt` (Watchful_IP), `squashfs-tools`, `binwalk`, `strings`, `objdump`, `john`, `ubireader`

---

## 1. Tehtävän tarkoitus

Tehtävänä oli purkaa TP-Linkin salattu kamerafirmware, tutkia sen sisältöä, ja selvittää miten laitteeseen saisi root-tason pääsyn. Tehtävä koostui kuudesta osasta:

1. Pura firmware-imagen salaus
2. Analysoi purettu tiedosto
3. Pura rootfs raa'asta flash-dumpista
4. Pura rootfs firmware-imagesta
5. Etsi laitteen sovellukset
6. Analysoi ja yritä avata root-salasana

---

## 2. Käytettävissä olleet tiedostot

| Tiedosto | Sisältö | Koko |
|---|---|---|
| `Tapo_C200v3_en_1.4.2_..._boot-signed_...bin` | Salattu firmware-paketti (ladattu TP-Linkin palvelimelta) | 8 117 624 tavua |
| `dump-tapo-c200v3-1.4.2.bin` | Raaka SPI-flash-dumppi fyysisesti luetusta laitteesta | 8 388 608 tavua (8 MB) |

---

## 3. Vaihe 1: Firmware-salauksen purku

TP-Link allekirjoittaa ja salaa firmwarensa. `tp-link-decrypt`-työkalu käyttää TP-Linkin julkisesti julkaisemista GPL-lähdekoodipaketeista löytyviä RSA/AES-avaimia salauksen purkuun - avaimet ovat valmistajan itsensä julkaisemia, ei murrettuja.

```bash
git clone https://github.com/tangrs/tp-link-decrypt.git
cd tp-link-decrypt

sudo apt-get install -y openssl libssl-dev build-essential squashfs-tools

# Ladataan TP-Linkin GPL-paketit ja poimitaan avaimet niistä automaattisesti
./extract_keys.sh
make
```

Avaimet löytyivät suoraan TP-Linkin omista julkisista tiedostoista:
```
DES_KEY: 0001020304050607
KEY=9c6ba1d761e4eee17dfde90cfed603bd
IV=8778f31423815ce85e9f186b60507edd
```

**Itse purku:**
```bash
./bin/tp-link-decrypt Tapo_C200v3_en_1.4.2_..._boot-signed_...bin
```

**Tulos:**
```
Tapo firmware header found
RSA-2048
Firmware verification successful
Decrypted firmware written to ...bin.dec
```

RSA-2048-allekirjoitus vahvistui onnistuneesti (aito TP-Link-firmware), ja tiedosto purettiin salauksesta.

---

## 4. Vaihe 2: Puretun tiedoston analyysi

```bash
file Tapo_C200v3_..._up_boot-signed_...bin.dec
hexdump -C Tapo_C200v3_..._up_boot-signed_...bin.dec | head -20
```

Löydökset:
- `0x200` - LZO-pakattua dataa + viittaus `u-boot.bin`-tiedostoon (bootloaderi)
- Squashfs-tunniste (`hsqs`) etsittiin:
  ```bash
  hexdump -C Tapo_C200v3_..._up_boot-signed_...bin.dec | grep '68 73 71 73'
  ```
  löytyi osoitteesta `0x003e0200`

---

## 5. Vaihe 3 & 4: Rootfs:n purku (dumpista ja imagesta)

Molemmista tiedostoista löytyi sama squashfs-tiedostojärjestelmä. Tämä todistaa, että dump vastaa oikeasti laitteessa ajossa olevaa firmwarea.

**Firmware-imagesta:**
```bash
dd if=Tapo_C200v3_..._up_boot-signed_...bin.dec of=rootfs.squashfs bs=1 skip=$((0x003e0200))
unsquashfs rootfs.squashfs
```

**Raa'asta flash-dumpista** (offset hieman eri, `0x00440000`):
```bash
hexdump -C dump-tapo-c200v3-1.4.2.bin | grep '68 73 71 73'
# 00440000  68 73 71 73 ...

dd if=dump-tapo-c200v3-1.4.2.bin of=rootfs_from_dump.squashfs bs=1 skip=$((0x00440000))
unsquashfs rootfs_from_dump.squashfs
```

Molemmista purkautui identtinen tiedostojärjestelmä: 76 inodea, kansiot `bin/`, `lib/`, `usr/`, `etc/`, `config/`.

---

## 6. Vaihe 5: Sovellusten etsintä

```bash
find squashfs-root -type f -executable
ls -la squashfs-root/bin/ squashfs-root/usr/sbin/
```

| Sovellus/tiedosto | Tarkoitus |
|---|---|
| `bin/main` | Kameran pääsovellus - API:t, videosyöte, WiFi-hallinta |
| `bin/gdbserver` | Etä-debug-työkalu (epätavallinen tuotanto-firmwaressa) |
| `bin/impdbg` | Ingenic-valmistajan sisäinen debug-työkalu |
| `usr/sbin/hostapd` | WiFi-tukiasemademoni |
| `libaudioProcess.so`, `libudt.so` | Ääni-/videoprosessointi, UDT-verkkoprotokolla |
| Kernel-moduulit (`.ko`) | `tx-isp-t31.ko` (kuvasensori), `motor_driver.ko` (PTZ-moottori), `esp32.ko` (WiFi-siru) |

Laitteen SoC tunnistettiin Ingenic T31 -MIPS-prosessoriksi.

---

## 7. Vaihe 6: Root-salasanan analysointi

### 7.1 Staattinen etsintä

```bash
find squashfs-root -iname "*passwd*" -o -iname "*shadow*"
cat squashfs-root/etc/passwd    # ei löytynyt
cat squashfs-root/etc/shadow    # ei löytynyt
```

Salasanatiedostoa ei löytynyt puretusta tiedostojärjestelmästä kummastakaan lähteestä.

### 7.2 Piilotettu konfiguraatioarkisto

Dumpin hex-analyysissä löytyi piilotettu gzip-pakattu tar-arkisto (`base-files.tar.gz`) kohdasta `0x30100`, joka ei näkynyt normaalissa squashfs-purussa:

```bash
dd if=dump-tapo-c200v3-1.4.2.bin bs=1 skip=$((0x30100)) count=196608 | zcat > gzip_extracted.bin
tar -xf gzip_extracted.bin
```

Sisälsi laitekohtaisia asetustiedostoja (`encrypt_key`, `oem.config`, `ams.config`), mutta ei salasanaa - löytyi DES-avain `17CD41F251DB9B5F`.

### 7.3 Todellinen root-pääsyn menetelmä: U-Boot/UART-ohitus

Dumpista löytyi U-Boot SPL 2013.07 -bootloaderi, jonka käynnistys voidaan keskeyttää sarjaportin (UART) kautta:

```
U-Boot SPL 2013.07 (Apr 26 2023 - 10:09:24)
bootargs=console=ttyS1,115200n8 ... root=/dev/mtdblock6 rootfstype=squashfs
                                   ... init=/etc/preinit
```

**Selitys:** Käynnistyessään laitteella on lyhyt aika-ikkuna jonka aikana U-Boot odottaa näppäinpainallusta. Fyysisellä sarjaporttiliitännällä (UART, kolme johtoa piirilevyllä) siihen ehtimällä voidaan muuttaa käynnistyskomentoa niin, että laite käynnistää suoraan root-komentorivin (`/bin/sh`) tavallisen käyttöjärjestelmän sijaan - ilman salasanaa.

```bash
# UART-konsolissa (esim. minicom/screen 115200 baud):
setenv bootargs 'console=ttyS1,115200n8 ... init=/bin/sh'
boot
# -> suora root-shell, ei salasanaa tarvita
```

Tämä on fyysiseen pääsyyn perustuva haavoittuvuus (laite pitää avata ja liittää johdot piirilevylle).

### 7.4 Mahdollinen komennon injektio verkon yli

`bin/main`-binäärin disassemblointi paljasti mielenkiintoisen ketjun:

```bash
apt-get install -y binutils-mipsel-linux-gnu
mipsel-linux-gnu-objdump -d bin/main > main_disasm.txt
grep -n "jal.*<system\|jal.*<popen" main_disasm.txt
```

Datavirta konekoodista jäljitettynä:

1. JSON-muotoinen verkkopyyntö tulee sisään (esim. `{"method": 14, "params": "..."}`)
2. `jso_obj_get_string_origin()` lukee `"params"`-kentän arvon täysin raakana
3. Arvoa ei suodateta mitenkään
4. Arvo liitetään suoraan komentoon: `snprintf("%s %s %s", "wlan_operate get_oemid", <käyttäjän_syöte>, ...)`
5. Rakennettu komento ajetaan `popen()`-kutsulla - suora kuoritulkin suoritus

**Selitys:** Jos ohjelma rakentaa ajettavan komentorivin liittämällä siihen käyttäjän lähettämää tekstiä sellaisenaan, hyökkääjä voi lisätä sekaan omia lisäkomentoja (esim. `; avaa takaovi`). Tätä kutsutaan komennon injektioksi, ja se voisi toimia suoraan verkon yli - ei vaatisi fyysistä pääsyä kuten U-Boot-menetelmä.

Huom: tämä on staattisesta analyysistä tehty löydös. Täysi vahvistus vaatisi testauksen oikealla laitteella muokatulla verkkopyynnöllä.

### 7.5 Vaihtoehtoisten tiedostojärjestelmien poissulkeminen

Kokeiltiin myös `ubireader`-työkalua UBI/UBIFS-tiedostojärjestelmän varalta:

```bash
pip install ubi_reader --break-system-packages
ubireader_display_info dump-tapo-c200v3-1.4.2.bin
# "Could not determine start offset" kaikilla yleisillä lohkokoilla
```

Tulos: ei UBI-rakennetta. Odotettu tulos, koska laite käyttää squashfs-tiedostojärjestelmää (vain luku, ei tarvitse UBI:n kulumisen tasausta). Toisen tyyppisen laitteen mahdollinen UBI-käyttö johtuu todennäköisesti eri mallista.

---

## 8. Yhteenveto tehtävistä

| # | Tehtävä | Status | Menetelmä |
|---|---|---|---|
| 1 | Pura firmware-image | Valmis | `tp-link-decrypt` + TP-Linkin julkiset GPL-avaimet |
| 2 | Analysoi image-tiedosto | Valmis | `hexdump` + magic byte -tunnistus |
| 3 | Pura rootfs dumpista | Valmis | `dd` + `unsquashfs`, offset `0x440000` |
| 4 | Pura rootfs image-tiedostosta | Valmis | `dd` + `unsquashfs`, offset `0x3e0200` |
| 5 | Etsi sovellukset | Valmis | `find`/`ls` puretussa rootfs:ssä |
| 6 | Avaa root-salasana | Analysoitu | Ei staattista salasanaa - oikea reitti on U-Boot/UART-ohitus (fyysinen), lisäksi mahdollinen komennon injektio (verkko) |

---

## 9. Pääjohtopäätökset

- TP-Link käyttää samoja RSA/AES-avaimia julkisesti eri laitemalleissa - tämä on itsessään heikkous, koska yksi avainvuoto vaikuttaa moneen tuotteeseen.
- Root-salasanaa ei ole tallennettu selkokielisenä laitteeseen. Pääsy saadaan keskeyttämällä käynnistysprosessi fyysisesti (UART).
- Firmwaresta löytyi merkkejä mahdollisesta komennon injektio -haavoittuvuudesta, joka voisi toimia myös etänä verkon yli.
- Eri Tapo-mallit voivat käyttää eri tiedostojärjestelmiä (squashfs vs. UBI/UBIFS), joten jokainen laite pitää analysoida erikseen.

---

## 10. Käytetyt työkalut

```bash
# Perusanalyysi
file, hexdump, strings, dd, find, grep

# Purku/pakkaus
tp-link-decrypt      # TP-Link-firmwaren RSA/AES-purku
unsquashfs            # Squashfs-tiedostojärjestelmän purku
tar, gzip             # Piilotettujen arkistojen purku

# Binäärianalyysi
objdump (mipsel-linux-gnu-objdump)  # MIPS-konekoodin disassemblointi
readelf, nm                          # ELF-tiedoston rakenteen tutkiminen

# Salasanan murto (kokeiltu, ei tuottanut tulosta tällä datalla)
john (John the Ripper)

# Vaihtoehtoisten tiedostojärjestelmien tunnistus
ubireader / ubi_reader
```

---

## Loppuhuomautus

Suurin osa tästä analyysistä (komentojen valinta, tulosten tulkinta, konekoodin jäljittäminen ja haavoittuvuuksien tunnistaminen) on tehty Claude-tekoälyn avustuksella, mihin opettaja on antanut luvan. Oma osaamiseni painottuu laitteistopuolelle (mm. iPhone-mikrojuotokset ja kitarapedaalien rakentelu), joten piirilevytason "hakkerointi" (esim. UART-liitäntöjen juottaminen ja fyysinen laitteen avaaminen) olisi huomattavasti luontevampaa kuin pelkkä terminaalissa tehtävä binääri- ja konekoodianalyysi.
