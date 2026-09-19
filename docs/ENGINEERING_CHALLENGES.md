# Engineering Challenges and Lessons

Each entry traces to project files or commits (kept in private repositories). The supplier is described generically and never named. Nothing here is a claim about the physical boards working; bring-up is pending.

---

## 1. Module footprints that passed DRC but did not match the real modules

- **Challenge:** The Ra-02 LoRa module socket had its two pin rows 2.54 mm apart, which is the pin pitch within a row, not the row-to-row spacing. The real module needs 25.4 mm.
- **Why it mattered:** DRC only checks copper rules. A footprint can be electrically clean and still physically impossible to plug a real module into.
- **How found:** A pre-fabrication audit comparing the footprints against the modules' real specifications and supplier drawings.
- **Investigation:** Compared footprints against the supplier's dimensioned drawings and the module datasheet, measured actual pad-to-pad coordinates with KiCad's Python API (not footprint anchor points), and checked the interleaved pin numbering.
- **Solution:** Corrected the row spacing to 25.4 mm on all three boards. The same review then found and fixed the W5500 module (two 6-pin rows side by side, not one stacked column, 20.32 mm apart) and the RS485 module (37.0 mm to 40.64 mm).
- **Verification:** Fresh DRC, ERC and netlist match after each change; pin-for-pin comparison of the module nets between schematic and PCB; gerbers regenerated and re-checked.
- **Lesson:** Verify footprints against the physical part, not only against DRC.

## 2. A "fixed" measurement that was actually measuring the wrong thing

- **Challenge:** A check on the RS485 module spacing reported 37.0 mm after the fix, which looked like a regression.
- **How found:** Cross-checking the result before reporting it.
- **Investigation:** The script compared footprint anchor points; this footprint's origin is offset from pin 1.
- **Solution:** Re-measured pin 1 to pin 1, giving 40.64 mm.
- **Verification:** Independent web reference (headers 1.6 inches apart) and the supplier drawing.
- **Lesson:** A verification script needs verifying too. Measure the thing that physically matters.

## 3. ESP32-S3 header spacing on the Camera Node (30 mm vs 25.4 mm)

- **Challenge:** The two ESP32-S3 header rows were 30 mm apart; both the official DevKitC-1 and the clone the supplier used have 25.4 mm.
- **Why it mattered:** The board cannot seat the module in both rows. On the physical boards the headers exist and are populated but sit at the old spacing, so rework is needed.
- **Investigation:** Checked both boards' dimension drawings. Moving one header put its busiest signal pins in the path of the camera data bus traces, so the fix was a reroute, not a nudge.
- **Solution:** Moved the header and re-routed 14 nets jointly with Freerouting, then re-applied it on the production board files (an earlier attempt had been made on an outdated copy of the board, which I caught and redid).
- **Verification:** KiCad DRC on the corrected board is identical to the original, item by item: no new violations, no new unconnected items. This is a design-file fix for future orders; the existing boards still need rework.
- **Lesson:** Check the mechanical interface of every module against two independent sources, and expect a footprint change to ripple into routing.

## 4. Fuse holder footprint smaller than the real part, and a short caused by my own fix

- **Challenge:** The fuse holder footprint (20 mm span, 1.05 mm drill) did not fit the real holder (23.33 mm span, about 1.32 mm leads).
- **Solution:** Widened to 23.5 mm span and 1.6 mm drill for future orders.
- **Mistake caught:** First attempt also enlarged the pad copper; DRC reported a short between a +5V pad and a +12V trace whose edges touched at exactly the same coordinate. Reverted the pad size and enlarged only the drill.
- **Verification:** Violation-by-violation diff against the pre-fix baseline showed zero new violations. The design change is for future orders; the current boards use a wire-in rework.
- **Lesson:** Compare DRC results item by item, not by count. A count can look fine while a new short hides in it.

## 5. Schematic and PCB out of sync (reversed diodes)

- **Challenge:** A forensic re-audit found the D2 flyback diode and the D1 backup-power diode wired backwards on the Sensor Node, and schematic-vs-PCB desyncs on D1, D2 and J4.
- **Why it mattered:** A reversed flyback diode fails to protect the relay driver, and a reversed backup diode defeats the power path. Both compile and route without error.
- **Investigation and fix:** Traced each net at pad level and corrected both files.
- **Lesson:** Keep the schematic and PCB in lockstep and audit both, not just the layout.

## 6. Firmware pin map disagreed with the PCB copper

- **Challenge:** The Camera Node firmware assigned the camera master clock to GPIO14, but the routed PCB and the reference CSV both put it on GPIO4.
- **Why it mattered:** The camera cannot run without its clock, and the code would have compiled and appeared to work.
- **Investigation:** Queried the real pads with pcbnew and cross-checked them against the project's pin CSV. Two independent sources agreed with each other and disagreed with the firmware.
- **Solution:** Corrected the firmware pins and rebuilt.
- **Lesson:** The PCB is the source of truth for pin assignments. Generate the firmware pin map from it.

## 7. Duplicate and orphaned copper left by scripted routing

- **Challenge:** Scripted re-routing left duplicate tracks, orphan vias and dead-end stubs (for example 84 duplicate tracks on one board, and a 104 mm dead-end stub on one net).
- **Investigation:** Traced each DRC hit individually to decide whether it was a real orphan or a benign stub off a connected pad.
- **Verification:** Connectivity showed zero unconnected items before and after removal.
- **Lesson:** Clean up after automation, and read each warning instead of counting them.

## 8. A placeholder that must not become a claim

- **Challenge:** The soil probe Modbus register map in the firmware is a placeholder from common conventions; the probe's datasheet was never obtained.
- **Decision:** Documented it in the code and in this project's public wording: soil sensing is implemented but not validated.
- **Next step:** Obtain the datasheet and correct the map during bring-up.
- **Lesson:** Record what is unverified as clearly as what is verified.

## 9. Component change late in procurement

- **Challenge:** The originally specified rain gauge became unavailable and a substitute (0.3 mm per pulse instead of 0.2794) was chosen.
- **Investigation:** Confirmed from a photo of the connector that only two wires are populated, and traced the PCB net to confirm polarity does not matter (a pull-up and a GPIO only).
- **Solution:** Changed the firmware constant and rebuilt.
- **Lesson:** Late component changes should propagate through firmware constants and documentation at once.

## 10. Two real vulnerabilities in my own deployed system

- **Challenge:** A security self-review found that every Gateway-facing endpoint on the live backend accepted anonymous requests (including one that returned every account's pending pump commands), and that the live WebSocket sent all accounts' readings to any client.
- **How found:** Reading the code, then confirming against the live deployment with harmless requests that create no data.
- **Solution:** Per-account device keys (hash-only storage, shown once, revocable) required on every device endpoint and scoped to the owner's nodes; an authenticated WebSocket with per-owner event delivery; a production startup guard; security headers; input bounds. Gateway firmware updated to send the key over HTTPS.
- **Verification:** 34 new tests, with two of the protections deliberately broken to prove the tests fail; the migration run up, down and up; the new Settings page used end to end; the live system re-probed (401 everywhere, docs off).
- **A surprise:** Render's proxy did not relay server-initiated WebSocket close frames, so rejected clients looked "still open". A temporary diagnostic route (deployed, then removed) showed it was the proxy, not the code, and that unauthenticated sockets still receive nothing.
- **Still open:** LoRa payload authentication; firmware signing.
- **Lesson:** Test the deployed system, not just the code, and separate "what the client observes" from "what the server did".

---

## 11. Fixes made on the wrong copy of the board files

- **Challenge:** Two design fixes (Camera header spacing, fuse footprint) had been made and documented on a copy of the board files that was not the one sent for fabrication. The copy differed in real ways, for example the LoRa socket rows.
- **How found:** Comparing the file lineages with KiCad's Python API before reusing the fixes.
- **Solution:** Kept the production files as the base, re-applied both fixes on them, and confirmed by DRC diff and drill-file diff that only the intended features changed.
- **Lesson:** Know which file is the source of truth before you fix anything, and compare against it.

---

## Cross-cutting lessons (for interviews)

1. Physical-world verification beats tool output: DRC cannot see a module that does not fit.
2. Measure with the right reference. Use pad coordinates, not anchor points.
3. Diff before and after, item by item.
4. Keep a single source of truth for pin assignments, and check both files when two must agree.
5. State plainly what has not been tested.
