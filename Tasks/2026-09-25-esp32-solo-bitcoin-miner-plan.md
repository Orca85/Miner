# ESP-32 Solo Bitcoin "Lottery" Miner

## Feature description

Headless (ingen skärm) Bitcoin solo-mining firmware för två lediga ESP-32-kort.
Korten ansluter till WiFi, pratar Stratum V1 mot en solo-mining-pool (t.ex.
`solo.ckpool.org`), räknar SHA256d över nonce-intervall och skickar in delade
shares. Syftet är hobby/"lottery mining" — extremt liten chans att hitta ett
riktigt block, ingen förväntad ekonomisk vinst. Baseras på det etablerade open
source-projektet **NerdMiner v2**, konfigurerat utan display-beroende.

## Detaljerad plan

1. **Identifiera hårdvara**
   - Avgör exakt chip-variant för båda korten (ESP32 klassisk / S3 / C3) via
     `esptool.py chip_id` när kort är anslutet via USB.
   - Notera flash-storlek (behövs för PlatformIO board-definition).
2. **Verktygskedja**
   - Installera PlatformIO CLI (`pip install platformio`) i användarens
     Python-miljö.
   - Verifiera att PlatformIO kan se enheten (`pio device list`).
3. **Hämta bas-kod**
   - Klona NerdMiner v2 (github.com/BitMaker-hub/NerdMiner_v2) som bas.
   - Identifiera display-relaterad kod och stäng av/mocka den så projektet
     bygger och körs headless (ingen TFT ansluten).
4. **Konfiguration per kort**
   - WiFi SSID/lösenord (via `secrets.h` eller motsvarande, ej hårdkodat i
     committade filer).
   - BTC-mottagaradress (publik adress, inte privat nyckel) för payout.
   - Pool-endpoint: `solo.ckpool.org:3333` (eller motsvarande solo-pool).
   - Unikt worker-namn per kort (t.ex. `esp32-1`, `esp32-2`) så de går att
     särskilja i poolens statistik.
5. **Bygg och flash**
   - Bygg firmware separat för respektive kort-variant/board-definition.
   - Flash via PlatformIO (`pio run -t upload`) till respektive enhet.
6. **Verifiering**
   - Läs serial-loggen och bekräfta: WiFi-anslutning OK, Stratum
     subscribe/authorize OK, `mining.notify` mottagen, hash-loop igång med
     rimlig hashrate (storleksordning kH/s), minst ett godkänt share inskickat.
   - Jämför hashrate mot förväntat värde för chip-typen som sanity check.
7. **Dokumentation**
   - Vid färdigställande: flytta denna plan till
     `/Documentation/Features/` som ren dokumentation, ta bort originalet
     från `/Tasks`.

## TODO checklist

- [x] Identifiera chip-variant och flash-storlek för kort #1 (USB anslutet)
- [x] Identifiera chip-variant och flash-storlek för kort #2
- [x] Installera PlatformIO CLI
- [x] Verifiera `pio device list` ser båda korten
- [x] Klona/vendorisera NerdMiner v2-bas
- [x] Ta bort/inaktivera display-beroende för headless-drift (färdig `ESP32-D0WD-V3-weact`-env, `NO_DISPLAY`)
- [x] Skapa WiFi-konfiguration för kort #1 (via inbyggd captive portal, ingen hemlighet i repo)
- [x] Skapa WiFi-konfiguration för kort #2
- [x] Sätt BTC-adress för kort #1
- [x] Sätt BTC-adress (+ worker-suffix `.esp-32-2`) för kort #2
- [x] Bygg firmware (gemensam binär för båda korten)
- [x] Flash och verifiera serial-logg för kort #1 — WiFi OK, pool subscribe/auth OK, hashar ~350 KH/s
- [x] Flash firmware för kort #2 (samma binär)
- [x] Verifiera serial-logg för kort #2 — WiFi OK (192.168.0.208), pool subscribe/auth OK, hashar ~354 KH/s
- [x] Bekräfta båda korten kör stabilt och hashar parallellt
- [ ] Flytta plan till `/Documentation/Features/` (väntar på bekräftelse från användaren)

## Implementeringsanteckningar

- Kort #1: ESP32-D0WD-V3 (rev v3.1), 4MB flash, MAC `d4:e9:f4:e3:c2:94`, COM6,
  AZ-Delivery ESP32-WROOM-32 devkit (USB-C, CP2102 UART).
- Kort #2: ESP32-D0WD-V3 (rev v3.1), 4MB flash, MAC `d4:e9:f4:e4:3e:44`, COM8,
  identisk modell som kort #1.
- Board-definition i PlatformIO för båda: `esp32dev` (standard 38-pin
  ESP32-WROOM-32, 4MB flash) — ingen S3/C3-anpassning behövs.
- PlatformIO-environment: `ESP32-D0WD-V3-weact` i NerdMiner v2:s
  `platformio.ini` — sätter `DEVKITV1`, vilket via
  `src/drivers/devices/esp32DevKit.h` definierar `NO_DISPLAY`. Ignorerar
  `TFT_eSPI` helt i bygget. Exakt vad vi behöver, ingen kodändring krävdes.
- Konfiguration (WiFi/pool/BTC-adress) sker via inbyggd WiFiManager captive
  portal (AP-SSID `NerdMinerAP`, lösenord `MineYourCoins`) — inget att
  hårdkoda eller committa som hemlighet.
- **Känt problem (kort #1, löst):** Brownout-loop
  (`Brownout detector was triggered` → ständig `SW_CPU_RESET`) vid start,
  eftersom WiFi-radionsströmtoppar (~300–500mA) inte kunde levereras av
  ursprunglig USB-kabel/port. Löstes genom byte till bättre kabel/direkt
  USB-port på datorn (inte hubb). Om samma problem dyker upp på kort #2,
  applicera samma fix.
- Kort #1 verifierat: WiFi anslutet, `public-pool.io` subscribe+authorize OK,
  mottar `mining.notify`, hashar stabilt ~350 KH/s (förväntad nivå för
  klassisk ESP32).
- Kort #2 verifierat: WiFi anslutet (192.168.0.208), samma pool, worker-namn
  `<btc-adress>.esp-32-2` för att skiljas från kort #1 i poolstatistiken,
  hashar stabilt ~354 KH/s.
- Total kombinerad hashrate: ~700-710 KH/s för båda korten.

## Verifieringsresultat

| Kort | COM-port | MAC | WiFi | Pool auth | Hashrate | Status |
|------|----------|-----|------|-----------|----------|--------|
| #1   | COM6     | d4:e9:f4:e3:c2:94 | 192.168.0.200 | OK (subscribe+authorize) | ~350 KH/s | Kör stabilt |
| #2   | COM8     | d4:e9:f4:e4:3e:44 | 192.168.0.208 | OK (subscribe+authorize) | ~354 KH/s | Kör stabilt |

Båda korten mottar `mining.notify`-jobb från `public-pool.io` kontinuerligt
och räknar SHA256d-hashar utan krascher under observationsperioden.

## Tillägg: statistik-dashboard

Byggde en fristående Claude Artifact ("Nonce Watch") som visar samlad
lotteristatistik för de två korten, hämtat från public-pool.io:s publika API
(`https://public-pool.io:40557/api/client/<address>`).

- Artifact-sidor kan inte göra `fetch()` mot godtyckliga externa domäner
  (CSP-sandboxad). Löst genom att sidan läser från artifact-databasen
  (`db`-capability) istället, med live `onSnapshot`-uppdatering.
- Försökte automatisera datainsamlingen med en schemalagd molnrutin
  (varje timme). Misslyckades: molnsandboxens nätverksproxy tillåter inte
  utgående anslutningar till icke-standardporten 40557, och poolen saknar
  variant på port 443. Rutinen är pausad (`trig_01UcznJEzVT63v2cr4zLtJqP`).
- Löst istället med manuell uppdatering: användaren säger "uppdatera
  dashboarden" i chatten, Claude hämtar lokalt (fungerar, verifierat) och
  skriver till artifact-databasen via ArtifactData.
- Länk: https://claude.ai/artifact/QFkWw1sENEPZCQDejABX1w (privat, ägarens
  konto).

## Sammanfattning

Två AZ-Delivery ESP32-WROOM-32-kort (ESP32-D0WD-V3, 4MB flash) är flashade
med NerdMiner v2, environment `ESP32-D0WD-V3-weact`, helt headless (ingen
skärm/TFT kompilerad in). Konfiguration av WiFi, pool och BTC-adress sker
via inbyggd captive portal (WiFiManager) — inget hårdkodat i firmware eller
committat i repo. Båda korten kör mot `public-pool.io:3333` som solo-miners
("lottery mining"), med unika worker-suffix för att skiljas åt i poolens
statistik. Kombinerad hashrate ~700 KH/s. Ett brownout-problem pga
otillräcklig USB-strömförsörjning upptäcktes och löstes genom byte till
bättre USB-kabel/port — värt att komma ihåg om korten någon gång flyttas
till en annan strömkälla (powerbank, väggadapter) i framtiden.
