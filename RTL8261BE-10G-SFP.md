# RTL8261BE SFP-10G-T — 10G on BPI-R4 (OpenWrt)

## Result

RTL8261BE copper SFP+ module (OEM SFP-10G-T) works at full **9.34 Gbits/sec** on BPI-R4
with MT7988A SoC and OpenWrt. The fix is in the kernel sfp.c driver.

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

Result: no link, module non-functional. The retry loop runs for ~8 minutes.

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

**Why ROLLBALL doesn't work:**
The ROLLBALL protocol uses I2C address 0x51 (standard SFP A2h DOM), switches to
page 3, writes CMD_READ, and polls CMD_DONE. RTL8261BE has no MDIO bridge — it
never asserts CMD_DONE. The poll times out after 10 × 20ms = 200ms per attempt,
and the kernel retries for minutes.

---

## Fix

**File:** `my_files/999-sfp-11-rtl8261be-mdio-none.patch`

The fix extends `sfp_fixup_rollball_cc` with a probe that auto-detects whether
a module has a real ROLLBALL I2C-to-MDIO bridge:

1. Send ROLLBALL unlock password (0xffffffff) to A2h at SFP_VSL+3
2. Switch to ROLLBALL MDIO page 3
3. Issue CMD_READ
4. Poll CMD_DONE for up to 200ms (10 × 20ms)
5. **CMD_DONE received** → real ROLLBALL bridge → `MDIO_I2C_ROLLBALL` + extended_cc fix
6. **Timeout** → no bridge (RTL8261BE) → `MDIO_I2C_NONE`

This replaces the previous approach of hardcoding a dedicated `sfp_fixup_rtl8261be`
function, which would have broken other rollball modules sharing the same vendor/product
strings. The new approach is safe for all modules matched by `sfp_fixup_rollball_cc`.

The industrial grade variant `SFP-10G-T-I` (same RTL8261BE chip) is also covered
by adding a quirk entry.

### Patch summary

```diff
--- a/drivers/net/phy/sfp.c
+++ b/drivers/net/phy/sfp.c

 /* Added before sfp_fixup_rollball_cc: */
+#define SFP_ROLLBALL_PHY_ADDR   0x51
+#define SFP_ROLLBALL_MDIO_PAGE  3
+#define SFP_ROLLBALL_CMD_ADDR   0x80
+#define SFP_ROLLBALL_CMD_READ   0x02
+#define SFP_ROLLBALL_CMD_DONE   0x04
+
+static bool sfp_has_rollball_bridge(struct sfp *sfp)
+{
+    /* Send password, issue CMD_READ, poll CMD_DONE 10×20ms.
+     * RTL8261BE has no bridge → timeout → returns false. */
+    ...
+}

 static void sfp_fixup_rollball_cc(struct sfp *sfp)
 {
-    sfp_fixup_rollball(sfp);
-    sfp->id.base.extended_cc = SFF8024_ECC_10GBASE_T_SFI;
+    if (!sfp_has_rollball_bridge(sfp)) {
+        sfp->mdio_protocol = MDIO_I2C_NONE;
+        return;
+    }
+    sfp_fixup_rollball(sfp);
+    sfp->id.base.extended_cc = SFF8024_ECC_10GBASE_T_SFI;
 }

 /* Quirk table: */
+    SFP_QUIRK_F("OEM", "SFP-10G-T-I", sfp_fixup_rollball_cc),
     SFP_QUIRK_F("OEM", "SFP-10G-T",   sfp_fixup_rollball_cc),
```

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

## Test status

| Case | Status | Notes |
|------|--------|-------|
| RTL8261BE → MDIO_I2C_NONE (probe timeout) | ✓ tested | Both routers, BPI-R4 MT7988A |
| Real ROLLBALL → MDIO_I2C_ROLLBALL (CMD_DONE) | ✗ not tested | Need a rollball module |

**To test the positive case:** Turris RTROM01-RTSF-10G (available at Discomp.cz).
Temporarily add `SFP_QUIRK_F("Turris", "RTSFP-10G", sfp_fixup_rollball_cc)` to
trigger the probe on that module and verify CMD_DONE is received.

---

## Notes

- **Multi-speed limitation (2.5G/1G):** With this patch the MAC is locked to
  10gbase-r. If the copper link partner only supports 2.5G or 1G, the RTL8261BE
  copper side will autoneg correctly, but the SerDes cannot switch speed without
  MDIO access. The MAC will report "Link is Up - 10Gbps" but no traffic will pass.
  This patch is suitable for **10G-only setups**. Full multi-speed support would
  require MT7988A internal SerDes register access (confidential MTK documentation).

- Patch is a candidate for upstream Linux kernel (`drivers/net/phy/sfp.c`),
  pending test of the positive rollball case.

---

## Date

First fix (hardcoded): 2026-05-14
Probe-based rewrite: 2026-05-14
