# Übergabe: RK3568-Weiterbildung (TRM ↔ Linux ↔ U-Boot)

Dieses Dokument ist der Einstieg für eine neue Sitzung. Erst lesen, dann
`trainingsplan-rk3568-uboot-linux.md`, dann beim "Nächsten Schritt" weitermachen.

## Ziel und Arbeitsweise

Martin lernt den RK3568 Kapitel für Kapitel anhand der TRM und will zu jedem
Abschnitt den Gegenpart im Linux-Kernel und in U-Boot verstehen. Ziel ist, die
Vorgehensweise selbst zu beherrschen; Durchsatz ist nicht das Ziel. Erkläre den
Weg: warum diese Quelle, warum dieser Test, warum diese Reihenfolge.

Regeln:

- Jede Aussage mit Quelle: `datei:zeile`, TRM-Seite, Commit-Hash oder Messwert.
- Status kennzeichnen: **gemessen**, **im Quelltext gelesen**, **abgeleitet**.
  Abgeleitetes nie als Befund darstellen.
- Hypothese und unterscheidenden Test vorher nennen, samt Deutung jedes
  möglichen Ausgangs. Immer nur eine Variable ändern.
- Befehle für Martin vollständig und kopierbar, mit Erwartung und Bedeutung
  **jedes** möglichen Ausgangs. Ein ausbleibender Treffer ist ein Befund.
- Kein Zugriff auf Laborhardware (serielle Konsole, Flash, Messgeräte): Befehle
  formulieren, Martin führt sie aus. Kein `git push`, keine Branch-Wechsel in
  seinen Arbeits-Repos.
- Große Aktionen (Subagenten, viele Dateien, lange Ausgaben) vorher ankündigen.
- Antworten auf Deutsch.

## Dateien und TRM

Im Repo liegt nur `trainingsplan-rk3568-uboot-linux.md` (Trainingsplan, nach
offenen Baustellen gegliedert, Stufen 0–6).

Die TRM liegt **nicht** im Repo; Martin lädt sie als PDF in die Sitzung hoch:

- Part 1, V1.3 (2022-09-30): 852 Seiten, Kap. 1–27.
- Part 2, V1.1 (2021-03-01): 2583 Seiten, Kap. 1–33. Part 2 ist älter als
  Part 1; Registerangaben können zwischen den Teilen abweichen.

**Lesbarkeit zu Beginn prüfen**, bevor Aussagen zur TRM als belegt gelten:

1. Part 1, Tab. 1-1 (S. 11): Startadresse von `SYS_SGRF`. Erwartet
   `0xFDD18000` (im Quelltext gelesen).
2. Letzte Zeile von Kap. 27 in Part 1: prüft, ob das Ende des PDFs erreichbar
   ist.
3. Part 2, Basisadresse des eMMC-Controllers (Kap. 7): erwartet `0xFE310000`
   (Part 1, Tab. 1-1).

Deutung: Stimmt (1), fehlt (2) → Part 1 nur teilweise lesbar. Falsche oder
erfundene Adresse bei (1) → das PDF wird nicht wirklich gelesen; weitere
TRM-Aussagen sind unbelegt, Sitzung abbrechen. Scheitert (3) am Upload-Limit
→ Part 2 in Teilen oder als Text bereitstellen (`pdftotext -layout`, Schwäche:
mehrzeilige Tabellenzellen werden zerlegt; bei Bitfeldern im PDF nachsehen).

## Quelltext-Referenzen

Für die Grundlagen reicht Mainline (**gemessen** per `git describe` /
`git tag --merged` in Martins lokalen Bäumen):

- Linux: `https://github.com/torvalds/linux`, Tag `v6.19.14` (Stable-Tag; ist
  er nicht erreichbar, auf `v6.19` ausweichen und das sagen).
- U-Boot: `https://github.com/u-boot/u-boot`, Tag `v2026.01`.
- Referenzboard: Radxa ROCK 3A (`configs/rock-3a-rk3568_defconfig`,
  `rk3568-rock-3a.dts`). Es bootet wie Martins Board per SPI-NOR
  (`ROCKCHIP_SPI_IMAGE`, `SPL_SPI_LOAD`). Weitere Referenzen: `evb-rk3568`,
  `odroid-m1-rk3568`, `nanopi-r5s-rk3568`, `generic-rk3568`.

Die Befunde unten wurden an Martins lokalen Fork-Branches des Boards erhoben,
die auf diesen Tags aufsetzen. Zeilennummern können in den Mainline-Tags um
einige Zeilen abweichen; **vor dem Zitieren im Tag nachsehen**. Die
RK3568-Basisdateien (`rk356x-base.dtsi`, `rk3568.c`, `bootrom.h`, `spl.c`)
sind Mainline-Code; dass sie im Fork unverändert sind, wurde nicht geprüft
(**abgeleitet**).

Das Board von Martin (Board-DTS, Board-Defconfig, `-u-boot.dtsi`) existiert
nur im Fork und wurde in Mainline nicht gesucht. Die Sitzung hat darauf keinen
Zugriff. Aussagen dazu stammen ausschließlich aus den Befunden unten; nicht so
tun, als wäre der Fork eingesehen worden.

## Plan

Einstieg über Kap. 1 und die Boot-Kette, dann Kap. 5, 7, 27 (Part 1) und
Kap. 7 (Part 2, eMMC); danach CRU, GRF und übrige Peripherie in
TRM-Reihenfolge. Pro Abschnitt: (1) Hardware laut TRM, (2) Linux,
(3) U-Boot, (4) Unterschiede, (5) Selbsttest.

## Stand: Kapitel 1 (System Overview, Part 1 S. 10–18) erledigt

Gelesen: 1.1 Address Mapping, 1.2 System Boot. **Nicht gelesen:** 1.3
Interrupt-Tabelle (kommt beim GIC-Kapitel, Part 1 Kap. 9).

TRM (im Quelltext gelesen):

- Flacher 36-Bit-Adressraum; Peripherie in `0xF0000000 … 0xFE8CFFFF` und
  `0x3_0000_0000 …` (PCIe-Fenster).
- Remap: BootROM (20 KB) liegt bei Reset auf `0xFFFF0000`.
  `PMU_SGRF_SOC_CON1[12:11]` blendet PMU_SRAM (`01`) oder SYSTEM_SRAM (`10`)
  dort ein; danach ist das BootROM unerreichbar.
- BootROM prüft einen ID-Block der Reihe nach auf: SPI-NOR, SPI-NAND, NAND,
  eMMC, SDMMC, danach USB-OTG-Download (Abb. 1-1).

Linux (im Quelltext gelesen, `arch/arm64/boot/dts/rockchip/rk356x-base.dtsi`):
GIC `0xfd400000` (Z. 279), pmugrf `fdc20000` (347), grf `fdc60000` (361),
pmucru `fdd00000` (386), cru `fdd20000` (393), i2c0 `fdd40000` (406),
uart0 `fdd50000` (419), usb_host0_xhci `fcc00000` (245). Alle stimmen mit
der TRM überein. Linux splittet den GIC-Block in GICD `fd400000` und GICR
`fd460000` (die GICR-Adresse steht nicht in Kap. 1). Kein Eintrag für
Remap/SGRF im Kernel: `grep 'sram|sgrf'` in `rk356x-base.dtsi` blieb leer
(negativer Befund; Deutung "Kernel läuft nach BL31" ist abgeleitet).

U-Boot (im Quelltext gelesen):

- `arch/arm/mach-rockchip/rk3568/rk3568.c`: MMU-Regionen RAM bis
  `0xF0000000`, Device `0xF0000000`–`0x100000000` und `0x300000000`–
  `0x3C0C00000`; deckt beide TRM-Fenster. `SGRF_BASE = 0xFDD18000` (Z. 24) =
  SYS_SGRF laut TRM; `SGRF_SOC_CON4` setzt HPROT-Secure für eMMC/SDMMC0.
- Board-Defconfig im Fork: `DEBUG_UART_BASE=0xFE660000` (UART2),
  SPI-Flash-Pfad `/spi@fe300000/flash@0` (FSPI). Beide stimmen mit der TRM
  überein.
- `arch/arm/include/asm/arch-rockchip/bootrom.h`: Boot-Quellen NAND=1,
  EMMC=2, SPINOR=3, SPINAND=4, SD=5, USB=10; BootROM legt die Quelle bei
  `CFG_IRAM_BASE + 0x10` ab; `arch/arm/mach-rockchip/spl.c:36` liest sie.
- SPI-Boot: `SPL_SPI_LOAD`, `SYS_SPI_U_BOOT_OFFS=0x60000`,
  `ROCKCHIP_SPI_IMAGE`; in `rk356x-u-boot.dtsi` hängt `mkimage -T rksd` mit
  `offset = <0x8000>` den ID-Block an.
- Kein `CONFIG_TPL*` im Board-Defconfig des Forks und im ROCK-3A-Defconfig;
  die DDR-Init kommt aus
  dem Rockchip-Blob (`ROCKCHIP_TPL`, `doc/board/rockchip/rockchip.rst:296-298`).

Auffälligkeiten:

1. Zwei Reihenfolgen: BootROM fest SPI-NOR → SPI-NAND → NAND → eMMC → SD;
   U-Boot-SPL `u-boot,spl-boot-order = "same-as-spl", &sdmmc0, &sdhci`
   (`rk356x-u-boot.dtsi`), also SD vor eMMC. Kein Widerspruch (zwei Stufen),
   aber wichtig für die Boot-Sel-Taster (Stufe 0).
2. Board-Anpassung im Fork: ohne `flash@0` mit `bootph-pre-ram` im SPL-DT
   fällt der SPL von SPI-NOR auf MMC zurück (Kommentar im `-u-boot.dtsi` des
   Boards). In Mainline vergleichen, wie `rk3568-rock-3a-u-boot.dtsi` das löst.

## Offene Punkte

- **Welcher U-Boot läuft auf Martins Board?** Seine Notizen sagen: Stock-U-Boot
  aus dem Yocto-Build. Die Befunde oben stammen aber aus einem Fork-Branch.
  Klären, bevor U-Boot-Details als "Martins Board" verbucht werden.
- Wie der Rockchip-DDR-Blob in das ID-Block-Image gelangt (idbloader-Aufbau,
  TPL ohne `CONFIG_TPL`): noch nicht nachvollzogen (Stufe 0).
- Wert von `CFG_IRAM_BASE` für RK3568 nicht ermittelt; Selbsttest
  `md.l <CFG_IRAM_BASE+0x10> 1` (3 = SPI-NOR, 2 = eMMC, 5 = SD, 0xa = USB)
  nicht ausgeführt.
- Ob die BootROM-Quellen-Zelle bis in U-Boot proper erhalten bleibt: abgeleitet,
  nicht gemessen.
- Kap. 13 in Part 2 fehlt in der Kapitelsuche (vermutlich Display/VOP2 mit
  abweichender Überschrift); beim Erreichen prüfen.

## Nächster Schritt

Part 1, Kap. 5 (Embedded SRAM) und Kap. 7 (PMU): Was sind die zwei SRAMs aus
Tab. 1-2, warum der Remap, und wie schließt das die Lücke zwischen BootROM und
SPL. Danach Kap. 27 (FSPI) mit Stufe 5 des Trainingsplans.
