# Trainingsplan: U-Boot & Linux Details am RK3568

Referenz sind Mainline-Linux (Tag `v6.19.14`) und Mainline-U-Boot (Tag
`v2026.01`) mit dem Radxa ROCK 3A als Referenzboard. Die Übungen stammen von
einem eigenen RK3568-Board mit SPI-NOR-Boot; sie lassen sich am ROCK 3A
nachvollziehen, wo sie nicht an Hardware gebunden sind.

Kein linearer Einsteiger-Kurs — die Module sind um die eigenen, bereits
dokumentierten offenen Baustellen an diesem Board herum gebaut. Jedes Modul
endet mit einer Übung, die direkt ein echtes offenes Item abschließt, statt
mit synthetischen Beispielen zu arbeiten.

## Stufe 0 — Die Boot-Kette als mentales Modell festigen

**Ziel:** RK3568 BootROM → idbloader (TPL+SPL) → FIT (ATF+U-Boot[+OP-TEE]) →
U-Boot proper → Kernel als ein zusammenhängendes Bild, nicht als lose Fakten.

**Ressourcen:** U-Boot `doc/board/rockchip/rockchip.rst` und
`doc/README.rockchip`, eigene Notizen zu den Boot-Sel-Tastern des Boards.

**Übung:** Die Kette für das eigene Board durchzeichnen, inklusive der beiden
Taster und was bei `BOOT_SEL0#`/`BOOT_SEL1#` jeweils aus der Reihenfolge fällt.

## Stufe 1 — U-Boot Driver Model (DM) wirklich verstehen

Größter Hebel: ein Board-Setup, das fest die Sysinfo-Instanz mit Sequenz 0
liest, ist ein reines DM-Verständnisproblem, kein Rockchip-Problem.

**Ziel:** `uclass`, `udevice`, `ofnode`, Unterschiede zwischen
`uclass_get_device_by_seq`, `_by_phandle`, `uclass_first_device` — und warum
`sysinfo_get_and_detect()` sich für `seq=0` entscheidet.

**Ressourcen:** U-Boot `doc/develop/driver-model/design.rst` und
`livetree.rst`, `include/sysinfo.h`.

**Übung:** Prototyp für eine zweite Sysinfo-Instanz (Sequenz 1) in einem
lokalen Branch verifizieren, bevor der Vorschlag an die Maintainer geht.

## Stufe 2 — Devicetree-Bindings: alt vs. neu, am eigenen Fall

**Ziel:** Den nvmem-Binding-Stil (`nvmem-cells`/`nvmem-layout`, Linux-Standard)
gegen einen herstellereigenen `i2c-eeprom`-Stil durchdenken — welche Annahmen
ändern sich (Offset-Berechnung, CRC-Platzierung)? Relevant für jeden Wechsel
eines Board-spezifischen EEPROM-Layouts in Richtung Mainline.

**Ressourcen:** `Documentation/devicetree/bindings/nvmem/` und
`.../eeprom/at24.yaml` im Linux-Tree.

**Übung:** Skizzieren, wie die Daten des eigenen EEPROM-Layouts als
nvmem-Cells aussehen müssten.

## Stufe 3 — Rockchip GRF/PMU_GRF

Hier ist schon Register-Arbeit passiert (`PMU_GRF_IO_VSEL0`/`_VSEL2` live
verifiziert) — nicht neu lernen, den Rest fertigmachen.

**Übung (offenes Item):** VCCIO2, VCCIO3, VCCIO5 ermitteln. `pdftotext` zerlegt
die mehrzeiligen TRM-Tabellenzellen — LibreOffice-Calc-Extraktion oder
manuelles Nachzählen der Bitfelder probieren.

## Stufe 4 — USB/PHY/dwc3: vom Fund zum Upstream-Patch

Der `phy_sus`-Bitmask-Bug in `phy-rockchip-inno-usb2.c` (U-Boot deckt nur Bits
[1:0] ab, Linux [8:0]) ist schon gefunden — ideale erste Gelegenheit für den
kompletten Upstream-Workflow, nicht nur den Code.

**Ziel:** U-Boot-Patch-Workflow (`checkpatch.pl`, `b4`/Patman — `.b4-config`
liegt schon im Checkout).

**Übung:** Den PHY-Fix als eigenständigen, upstream-fähigen Patch extrahieren
(losgelöst von der Board-Serie) und einreichen.

## Stufe 5 — FSPI-Boot + UBI/MTD: Kapstone-Übung

Fasst Stufe 0 (Boot-Kette), 1 (DM) und 2 (DT-Bindings) zusammen.

**Ziel:** MTD/UBI-Grundlagen — PEB vs. LEB, warum `UBI_PEB_SIZE=65536` zur
Erase-Block-Größe passen muss, was `ubinize`/`mkubifs` tatsächlich tun.

**Übung:** NOR-Chip-Part aus dem Schaltplan holen (512 Mbit/133 MHz/1,8V/Quad
ist schon bekannt, Hersteller/Part-Nummer fehlt noch für die Treiberwahl wie
`SPI_FLASH_MACRONIX`/`_XTX`), dann `CONFIG_ROCKCHIP_SFC` + Konsorten nach
`rock-3a-rk3568_defconfig`-Vorbild eintragen und FSPI-Boot zum Laufen bringen
(auf dem eigenen Board ohne Hardware-Änderung erreichbar — eMMC-Taster drücken).

## Stufe 6 — Meta-Skill: Branch-Archäologie

Schon "the hard way" gelernt (falsche Branch-Familie empfohlen, weil weniger
Dateien). Als feste Checkliste vor jeder Branch-Entscheidung:

```
ls drivers/sysinfo/ board/<hersteller>/common/ 2>/dev/null
grep "^config" board/<hersteller>/common/Kconfig
```

Nicht nach Dateizahl oder Versionsnummer urteilen.

---

## Reihenfolge-Empfehlung

1 → 4 (kurzer, motivierender Abschluss mit echtem Upstream-Erfolg) → 5
(größtes, zusammenhängendes Stück) → 2/3/6 parallel nebenher, wann immer die
jeweilige Baustelle wieder ansteht. Kein festes Zeitraster — die Übungen sind
an echte Items gebunden, die sowieso erledigt werden müssen.
