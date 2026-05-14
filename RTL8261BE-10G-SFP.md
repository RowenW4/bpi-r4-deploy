# RTL8261BE SFP-10G-T — 10G on BPI-R4 (OpenWrt)

## Result

RTL8261BE copper SFP+ module (OEM SFP-10G-T) works at full **9.34 Gbits/sec** on BPI-R4
with MT7988A SoC and OpenWrt. The fix is 4 lines of kernel code.

---

## Hardware

- **Module:** OEM SFP-10G-T (Realtek RTL8261BE-CG, 10G copper SFP+)
- **Router:** Banana Pi BPI-R4 (MT7988A), SFP2 port (sfp-lan)
- **Cable:** Cat5e/Cat6, short length (tested with 2m patch cable)
- **Link partner:** second BPI-R4 with the same module

---

## Problem

The Linux kernel identified OEM SFP-10G-T as a ROLLBALL module and attempted
to communicate via the I2C-to-MDIO bridge protocol:

```
sfp sfp2: module OEM SFP-10G-T rev A has been found in the quirk list
sfp sfp2: probing phy device through the [MDIO_I2C_ROLLBALL] protocol
sfp sfp2: no PHY detected, 24 tries left
sfp sfp2: no PHY detected, 23 tries left
...
```

Result: no link, module non-functional.

---

## Root Cause

RTL8261BE is a **pure Media Converter** — it has no I2C-to-MDIO PHY bridge and
does not support the ROLLBALL protocol. It auto-switches SerDes speed based on
the copper autoneg result:

- 1G copper → 1000BASE-X SerDes
- 2.5G copper → 2500BASE-X SerDes
- 10G copper → 10GBASE-R SerDes

The kernel must skip PHY probing entirely and let the MAC use the EEPROM-declared
link mode (10gbase-r) directly.

---

## Fix

**File:** `my_files/999-sfp-11-rtl8261be-mdio-none.patch`

```diff
--- a/drivers/net/phy/sfp.c
+++ b/drivers/net/phy/sfp.c
@@ -466,5 +466,10 @@ static void sfp_fixup_rollball_cc(struct sfp *sfp)
 	sfp->id.base.extended_cc = SFF8024_ECC_10GBASE_T_SFI;
 }
 
+static void sfp_fixup_rtl8261be(struct sfp *sfp)
+{
+	sfp->mdio_protocol = MDIO_I2C_NONE;
+}
+
 static void sfp_quirk_2500basex(const struct sfp_eeprom_id *id,
@@ -610,4 +615,4 @@ static const struct sfp_quirk sfp_quirks[] = {
 	SFP_QUIRK_F("ETU", "ESP-T5-R", sfp_fixup_rollball_cc),
-	SFP_QUIRK_F("OEM", "SFP-10G-T", sfp_fixup_rollball_cc),
+	SFP_QUIRK_F("OEM", "SFP-10G-T-I", sfp_fixup_rtl8261be),
+	SFP_QUIRK_F("OEM", "SFP-10G-T",   sfp_fixup_rtl8261be),
 	SFP_QUIRK_S("OEM", "SFP-2.5G-T", sfp_quirk_oem_2_5g),
```

`MDIO_I2C_NONE` tells the kernel: skip PHY probing, let the MAC configure the
SerDes directly from EEPROM (10gbase-r).

---

## Results

### dmesg after applying the patch

```
sfp sfp2: module OEM SFP-10G-T rev A has been found in the quirk list
sfp sfp2: probing phy device through the [MDIO_I2C_NONE] protocol
mtk_soc_eth sfp-lan: requesting link mode inband/10gbase-r with support 00,00000000,00000800,00006400
mtk_soc_eth sfp-lan: switched to inband/10gbase-r link mode
sfp sfp2: SM: exit present:up:link_up
mtk_soc_eth sfp-lan: Link is Up - 10Gbps/Full - flow control off
```

### ethtool sfp-lan

```
Settings for sfp-lan:
	Supported ports: [ FIBRE ]
	Supported link modes:   10000baseSR/Full
	Supported pause frame use: Symmetric Receive-only
	Supports auto-negotiation: No
	Supported FEC modes: Not reported
	Advertised link modes:  10000baseSR/Full
	Advertised pause frame use: Symmetric Receive-only
	Advertised auto-negotiation: No
	Advertised FEC modes: Not reported
	Speed: 10000Mb/s
	Duplex: Full
	Auto-negotiation: off
	Port: FIBRE
	PHYAD: 0
	Transceiver: internal
	Link detected: yes
```

### iperf3 (4 parallel streams, 10 seconds, BPI-R4 ↔ BPI-R4 via RTL8261BE + Cat5e)

```
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  2.50 GBytes  2.15 Gbits/sec  394            sender
[  7]   0.00-10.00  sec  3.25 GBytes  2.79 Gbits/sec  299            sender
[  9]   0.00-10.00  sec  2.33 GBytes  2.00 Gbits/sec   66            sender
[ 11]   0.00-10.00  sec  2.79 GBytes  2.40 Gbits/sec  178            sender
[SUM]   0.00-10.00  sec  10.9 GBytes  9.34 Gbits/sec  937             sender
[SUM]   0.00-10.01  sec  10.9 GBytes  9.32 Gbits/sec                  receiver
```

**9.34 Gbits/sec — 93% of 10GbE line rate.**

---

## Notes

- Patch also covers the industrial variant `SFP-10G-T-I` (same RTL8261BE chip)
- Without a copper link partner: SerDes stays at 10gbase-r (RTL8261BE power-on default)
- **Multi-speed limitation (2.5G/1G):** With this patch the MAC is locked to
  10gbase-r. If the copper link partner only supports 2.5G or 1G, the RTL8261BE
  copper side will autoneg correctly, but without MDIO access the SerDes cannot
  be switched to 2500BASE-X/1000BASE-X. The MAC will report "Link is Up - 10Gbps"
  but no traffic will pass — SerDes/MAC speed mismatch. This patch is therefore
  suitable for **10G-only setups**. Full multi-speed support would require a
  dedicated RTL8261BE MDIO driver.
- Patch is a candidate for upstream Linux kernel (`drivers/net/phy/sfp.c`)

---

## Date

Solved: 2026-05-14
