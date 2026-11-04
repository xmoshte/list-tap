# Power Source Assessment: CR2450 vs. LIR2450 for the ESP32-C3 / ESP-NOW

**Prepared for:** Hale  
**Subject:** Battery suitability and recommended power arrangement for the ESP32-C3 device

---

## 1. Executive Summary

I have reviewed the suitability of the CR2450 coin cell currently being considered for the ESP32-C3, particularly when the device uses ESP-NOW for periodic wireless transmissions.

My assessment is that the **CR2450 is not a reliable direct power source for this application**. Although the regulator in the design may be rated for 2 A, that rating does not mean the battery can deliver the current required by the ESP32-C3 during radio transmission. The short, high-current pulses can cause the CR2450's terminal voltage to drop significantly, resulting in regulator dropout and possible brownout resets.

I recommend considering the **LIR2450 rechargeable lithium-ion coin cell** instead. It has the same nominal physical form factor as the CR2450, but its lower internal resistance makes it better suited to handling the ESP32-C3's transmission current peaks. The main compromise is capacity: an LIR2450 typically stores substantially less energy than a CR2450 and will therefore need to be recharged more frequently.

There are two practical ways to implement the LIR2450:

- **Option A – On-board charging:** integrate a suitable lithium-ion charger into the device so the battery can be recharged without removing it.
- **Option B – External charging:** charge the cell separately and insert it into the device when required. This reduces the board's component count and complexity, but places more responsibility on the user.

The sections below explain the electrical limitations of the CR2450, the proposed LIR2450 arrangement, the two charging options, an illustrative runtime estimate and the relative cell costs. I have also included alternative approaches in case retaining a primary, non-rechargeable battery is a firm requirement.

| Parameter | CR2450 (current option) | **LIR2450 (recommended option)** |
|---|---:|---:|
| Chemistry | Li-MnO₂, primary | Li-ion, rechargeable |
| Nominal voltage | 3.0 V | 3.6 V (approximately 3.0–4.2 V operating range) |
| Typical capacity | ~540–620 mAh | ~100–120 mAh |
| Internal resistance | ~10–30 Ω when fresh; increases with discharge and low temperature | Much lower; typically a few ohms or less |
| Pulse capability | Approximately 15–50 mA | Approximately 100–200 mA continuous, subject to the specific cell datasheet |
| Footprint | 24.5 mm × 5.0 mm | Same form factor |
| Suitability for ESP32-C3 transmission bursts | Not recommended | More suitable, subject to power-stage validation |

## 2. Why I Do Not Recommend the CR2450

### 2.1 Transmission current demand

During an ESP-NOW transmission, the ESP32-C3 can draw short current pulses in the region of **200–300 mA**. These bursts are brief, often below 5–10 ms, but the battery still has to supply the current at the moment the radio transmits.

A CR2450 is designed for relatively low-current applications. Its typical continuous load is around 0.2–1 mA, with pulse capability generally in the region of 15–50 mA, depending on the cell and test conditions. A 200–300 mA transmission demand is therefore well beyond what I would consider a dependable operating condition for this battery.

### 2.2 Internal resistance and voltage drop

The CR2450's internal resistance is a major limitation. A fresh cell may have an internal resistance of approximately 10–30 Ω, and this value increases as the cell discharges or operates at lower temperatures.

For illustration, using a 300 mA current pulse and an assumed internal resistance of 20 Ω:

```text
V_drop = I × R_in
       = 0.3 A × 20 Ω
       = 6.0 V
```

This calculated drop is greater than the battery's nominal 3.0 V voltage. In practice, the cell cannot sustain that demand; its terminal voltage will collapse significantly during the pulse. The actual voltage response will depend on the cell, its state of charge, temperature and pulse duration, but the calculation demonstrates why the CR2450 is unsuitable for this load profile.

### 2.3 Why the 2 A regulator does not solve the problem

The regulator's 2 A rating describes the current it can deliver under its specified operating conditions. It does not increase the current capability of the battery connected to its input.

When the ESP32-C3 begins transmitting, the regulator draws the required energy from the cell. If the CR2450's voltage falls below the regulator's minimum input voltage, the regulator can no longer maintain its output. The 3.3 V supply then drops, potentially causing the ESP32-C3 to brown out or reset. Repeated transmission attempts can consequently lead to unstable operation or a boot loop.

For this reason, changing to a higher-current regulator alone will not address the underlying limitation. The battery and the complete power path need to be able to support the radio's peak demand.

## 3. Recommended Approach: LIR2450

### 3.1 Why the LIR2450 is a better fit

The LIR2450 is a more suitable option for this application for the following reasons:

- **Lower internal resistance:** it can handle higher current demands with less voltage sag than a CR2450.
- **Rechargeable chemistry:** it is suitable for a device that wakes, transmits and returns to sleep repeatedly, provided the charging arrangement is designed correctly.
- **Same physical format:** it uses the same 24.5 mm × 5.0 mm coin-cell form factor, which may allow the existing holder and enclosure to be retained.
- **Higher nominal voltage:** its nominal voltage is around 3.6 V, with a full-charge voltage of approximately 4.2 V. This provides more input-voltage headroom, although the regulator must be selected to accommodate the full cell-voltage range.

The LIR2450 is not a drop-in solution without design checks. The charging circuit, cell protection and regulator must all be compatible with a rechargeable lithium-ion cell.

### 3.2 Trade-offs to consider

There are a few compromises worth considering before making the change:

- **Lower capacity:** an LIR2450 typically provides around 100–120 mAh, compared with approximately 540–620 mAh for a CR2450. The time between charges will therefore be shorter, all else being equal.
- **Charging requirement:** the cell needs a proper constant-current/constant-voltage (CC/CV) lithium-ion charging circuit. It must not be charged using a circuit intended for a primary CR2450 cell.
- **Cell protection:** over-charge, over-discharge and short-circuit protection need to be provided either by a protected cell or by the device circuitry. Coin-format lithium-ion cells also have more limited cycle life than many larger rechargeable cells.
- **Temperature limits:** lithium-ion cells should not be charged below approximately 0 °C. The selected cell and charger datasheets should be checked for their exact temperature limits.

### 3.3 Charging arrangements

I see two workable ways to implement the LIR2450. The choice depends mainly on whether the priority is a simpler, lower-cost board or a more convenient charging experience for the end user.

#### Option A: Integrate the charger on the board

```text
USB / DC input → Li-ion CC/CV charger → LIR2450 → protection
               → 3.3 V regulator → ESP32-C3
```

With this arrangement, the battery remains installed in the device and is recharged through a USB or DC input. From the user's perspective, this is the more convenient option because there is no need to remove or handle the cell during normal charging.

| Design block | My recommendation |
|---|---|
| **Charger** | Use a linear lithium-ion charger with programmable charge current. For a 120 mAh cell, a starting design target of 0.5C or less is approximately 50–60 mA. Confirm the permitted charge current against the selected cell datasheet. Consider a charger with an NTC input if temperature monitoring is required. |
| **Cell protection** | Use a protected cell or add a suitable protection IC. Provide an undervoltage cutoff around 3.0 V, subject to the cell manufacturer's limits. |
| **Regulator** | The cell voltage varies from approximately 4.2 V when fully charged to around 3.0 V near discharge. Since the load requires 3.3 V, a buck-boost regulator is the more robust choice. An LDO may be acceptable if reduced regulation near the end of discharge is understood and tested. Select a low-quiescent-current part; the existing 2 A regulator may be oversized, and its quiescent current could have a significant effect on sleep life. |
| **Bulk capacitance** | Place approximately 100–220 µF of low-ESR capacitance close to the ESP32-C3 supply pins to help support radio current bursts. Validate the value and placement through measurement. |
| **Brownout detection** | Keep the ESP32-C3 brownout detector enabled. Test operation at low battery voltage and at the expected temperature extremes. |
| **Firmware** | Use deep sleep between events, keep boot and transmission times short, and avoid unnecessary Wi-Fi scanning. Reducing radio-on time will help extend battery life. |

**Advantages**

- The user can recharge the device without removing the cell.
- There is less handling of the battery and fewer opportunities for incorrect insertion.
- Charging and battery-temperature monitoring can be managed by the device.

**Disadvantages**

- The charger IC, input connector and supporting components increase the bill of materials (BoM).
- The charger takes up additional PCB area.
- The charging circuit adds design, testing and validation work.

#### Option B: Charge the cell externally and insert it when needed

```text
External coin-cell charger → LIR2450 → board battery holder
                           → protection → 3.3 V regulator → ESP32-C3
```

Under this arrangement, the charger and USB input are kept off the device PCB. The user charges the LIR2450 in a separate, compatible charger and inserts the charged cell into the board's holder when the device is to be used.

This approach reduces the board's component count, PCB area and design complexity. It is likely to be the more cost-conscious option, but it is less convenient for the user.

| Design block | My recommendation |
|---|---|
| **External charger** | Use a charger specifically designed for single-cell lithium-ion coin cells, or a suitable CC/CV charger with a coin-cell holder. Set the charge voltage to 4.2 V and use a conservative charge current, initially around 0.5C or less (approximately 50–60 mA for a 120 mAh cell), provided this is allowed by the cell datasheet. Avoid generic chargers that default to 500 mA or higher. |
| **Undervoltage protection on the board** | Many LIR2450 cells are sold without protection. Since the charger is external, the board must still protect the cell against excessive discharge. Add a hardware cutoff around 3.0 V using a regulator with programmable UVLO, a supervisor IC or a dedicated protection IC. Firmware shutdown at low voltage is useful as an additional measure, but should not replace hardware protection. |
| **Reverse-polarity protection** | The cell may be inserted incorrectly. I recommend a P-FET ideal-diode arrangement rather than a series Schottky diode, which can introduce approximately 0.2–0.4 V of voltage drop. |
| **Cell holder** | Holder contact resistance becomes part of the battery-to-load path. Use a good-quality spring-contact holder and measure the supply voltage during an ESP32-C3 transmission burst. |
| **Regulator, capacitance, brownout detection and firmware** | Apply the same recommendations outlined under Option A. |
| **Insertion behaviour** | Power will be interrupted when the cell is removed. Confirm that the board starts reliably after each insertion and check the inrush current associated with charging the bulk capacitor. |

**Advantages**

- Lower BoM cost and reduced PCB area.
- Fewer components and a simpler board design.
- A spare charged cell can be kept available for replacement.

**Disadvantages**

- The user must manage a separate charger and charged cells.
- An uncharged cell may make the device appear faulty or dead.
- Repeated insertion and removal can wear the holder contacts.
- Without a properly implemented undervoltage cutoff, the cell could remain installed and become excessively discharged.

#### Comparison of the two charging options

| Consideration | Option A: On-board charger | Option B: External charging |
|---|---|---|
| BoM cost | Higher | Lower |
| PCB area and complexity | Higher | Lower |
| User convenience | Higher; charge in place | Lower; remove and charge the cell separately |
| Protection on the board | Charger and protection circuitry | Undervoltage and reverse-polarity protection still required |
| Contact wear | Minimal | Repeated insertion can cause wear |
| Best suited to | Consumer-facing products where ease of use is important | Cost-sensitive products where users can manage cell replacement and charging |

**My recommendation:** If the device is intended for regular use by a client or end user, I would favour Option A because it provides a more straightforward charging experience. If reducing PCB cost and area is the main priority, Option B is viable, provided the external charger requirements and on-board protection are clearly specified.

### 3.4 Illustrative runtime estimate

To give an indication of the possible runtime, I have used the following simplified relationship:

```text
Average current = sleep current + (charge consumed per event ÷ event interval)
```

For this example, I have assumed:

- Usable battery capacity: 100 mAh (approximately 80% of a nominal 120 mAh cell)
- Sleep current: 10 µA
- Event interval: one event every 10 minutes
- Energy consumed per event: approximately 20 mC, equivalent to about 5.6 µAh, including boot and transmission

The calculation is:

```text
Event average = 5.6 µAh ÷ 0.167 h
              ≈ 33.6 µA

Total average = 10 µA + 33.6 µA
              ≈ 43.6 µA

Estimated runtime = 100 mAh ÷ 0.0436 mA
                  ≈ 2,300 hours
                  ≈ 95 days
```

On these assumptions, the estimated runtime is approximately **95 days**. This is an illustrative estimate rather than a guaranteed operating life. Actual performance will depend on boot time, transmit power, event frequency, sleep current, regulator efficiency, battery characteristics and operating temperature.

I recommend measuring the complete current profile on the actual hardware before using this figure as a product runtime specification.

## 4. Cell Cost Comparison: CR2450 vs. LIR2450

I have also compared indicative market prices for the two cell types to understand the likely battery cost impact of the proposed change.

The price points below are based on distributor listings checked in November 2026. They are indicative only and may change with order quantity, supplier, shipping costs and VAT treatment. Hale should obtain current supplier quotations before finalising the production BoM.

### 4.1 Indicative cell pricing

| Cell | Rated capacity | Indicative prices observed | Approximate cost per rated mAh |
|---|---:|---|---:|
| **CR2450** (Panasonic, 3 V) | 620 mAh | Approximately US$1.04–1.35 each at quantities of 20–3,600 pcs (Future Electronics / Digi-Key marketplace); around US$1.17 for a single unit (ichome); approximately €2–4 at single-unit retail | ~US$0.002 per mAh |
| **LIR2450** (3.6 V Li-ion) | 120 mAh | Approximately £3.40 for a single unit and £2.14 each at 500 pcs ex VAT (Farnell / Multicomp Pro); approximately €0.85–1.50 each at quantities of 10–1,000 pcs (nkon, depending on the VAT treatment shown) | ~US$0.01–0.02 per mAh |

The LIR2450 has a higher cost per rated mAh and a lower overall capacity. However, the decision should not be based on capacity or cell price alone. The CR2450's current-delivery limitation makes it a poor fit for the ESP32-C3 transmission profile, regardless of its higher nominal capacity.

The total cost comparison should also account for the selected charging approach. Option A adds charging components and an input connector to the PCB, while Option B requires a separate charger and a process for managing charged cells.

## 5. Other Options Considered

If the LIR2450 does not meet the product's runtime, size or cost requirements, I would consider the following alternatives.

### 5.1 Small LiPo pouch cell (150–300 mAh)

A small LiPo pouch cell is a good alternative if the enclosure can accommodate a flat, rectangular battery instead of the coin-cell format. It generally offers more capacity than an LIR2450 and has low internal resistance, making it better suited to the ESP32-C3's transmission bursts.

The specific cell datasheet should still be checked for its maximum continuous and pulse discharge ratings. Small pouch cells can have modest continuous-current ratings, even where they can tolerate higher short-duration peaks.

### 5.2 LiFePO₄ cell (for example, 14500 format)

A single LiFePO₄ cell has a nominal voltage of approximately 3.2 V and a maximum charged voltage of around 3.6 V. It offers good pulse-current capability, safety characteristics and cycle life.

One point to consider is that the cell voltage falls below 3.0 V towards the end of discharge. If the regulator is removed and the cell is connected directly to the load, a suitable low-voltage cutoff is still required. The 14500 format is approximately AA-sized, so it is considerably larger than a CR2450 and would require a change to the enclosure and battery holder.

### 5.3 Retain the CR2450 with an energy buffer

If a primary, non-rechargeable coin cell is a strict requirement, it may be possible to retain the CR2450 by separating the battery from the ESP32-C3's short current bursts.

One possible arrangement is to charge a supercapacitor or hybrid-layer capacitor slowly from the CR2450 through a series resistor (in the range of a few hundred ohms), then use a very-low-quiescent-current buck-boost regulator to power the device from the capacitor during transmission.

This approach is only suitable for infrequent events, potentially spaced several minutes apart. The capacitor's leakage current and equivalent series resistance (ESR) would need to be checked carefully, as would the recharge time between events. I would treat this as a workaround rather than the preferred solution for a general-purpose design.

## 6. Conclusion and Recommended Next Steps

Based on the ESP32-C3's expected transmission current and the limitations of the CR2450, I do not recommend retaining the CR2450 as the device's direct power source. The battery's internal resistance and limited pulse-current capability make supply-voltage collapse during ESP-NOW transmissions a significant risk. A higher-current regulator by itself will not resolve that limitation.

My preferred direction is to move to an **LIR2450**, subject to confirming that its discharge capability, capacity and charging requirements meet the product's operating needs. It retains the existing coin-cell footprint while providing a more suitable current source for the ESP32-C3.

For the implementation, I recommend the following:

1. **Confirm the exact LIR2450 cell.** Obtain its datasheet and verify the permitted charge current, discharge current, voltage limits, temperature limits and protection requirements.
2. **Select the charging arrangement.** Use Option A if ease of charging is the priority; use Option B if minimising PCB cost and area is more important.
3. **Add appropriate protection and local capacitance.** Include undervoltage protection, reverse-polarity protection where the cell is removable, and suitable low-ESR bulk capacitance near the ESP32-C3.
4. **Validate the design on hardware.** Measure the battery voltage and 3.3 V rail with an oscilloscope during ESP-NOW transmission, including at low state of charge. Confirm that the device does not brown out or reset.
5. **Measure actual runtime.** Record sleep current, boot current, transmission current and event frequency on the completed design, then update the runtime estimate using measured values.

In summary, **the LIR2450 is the more practical coin-cell option for this application**, while the final choice between integrated and external charging should be based on the intended user experience and the project's BoM and PCB constraints.
