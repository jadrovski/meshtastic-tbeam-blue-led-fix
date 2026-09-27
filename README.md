# Meshtastic T-Beam blue LED fix

Patch against **meshtastic/firmware v2.7.26.54e0d8d** for the classic
LILYGO T-Beam (V1.x, AXP192). Fixes the blue LED that keeps glowing
after the board is powered off.

## Background

On the T-Beam V1.x the blue LED behind the display is driven by the
**CHGLED output of the AXP192 PMIC**. That output lives in the
always-powered domain of the PMIC, so whatever mode firmware last wrote
stays active even after the board is switched off — the LED keeps
glowing forever (see also meshtastic/firmware#10170, gereic/GXAirCom#129).

Stock 2.7.x firmware sets CHGLED modes manually (heartbeat / power
indication), which is what makes it stick ON after shutdown.

## What this patch does

1. **Hands CHGLED to the PMIC's autonomous charge-indication mode**
   (`XPOWERS_CHG_LED_CTRL_CHG`) at PMU init, and stops the firmware from
   writing manual LED modes afterwards:
   - LED is ON while charging
   - LED turns OFF by itself when charging completes (~100%)
   - LED is dark when running on battery
   - LED is dark after power-off — no more stuck glow, because the PMIC
     itself updates the LED state even with the main CPU off
2. **Holds the GPIO4 "power LED" pin at its OFF level through deep
   sleep** (`gpio_hold_en`), same technique the firmware already uses
   for the button and LoRA CS pins — prevents the pin from floating
   and dimly glowing in deep sleep.

## Applying

```bash
git clone https://github.com/meshtastic/firmware
cd firmware
git checkout v2.7.26.54e0d8d
git apply /path/to/led.patch
pio run -e tbeam            # build
pio run -e tbeam -t upload --upload-port /dev/ttyACM0
```

## Verified behavior (T-Beam V1.1)

| State                | Blue LED            |
| -------------------- | ------------------- |
| Charging via USB     | on                  |
| Charge complete      | off                 |
| Running on battery   | off                 |
| Powered off (PWR)    | off (fully dark)    |
| Deep sleep / shutdown| off (fully dark)    |

Tested on two T-Beam V1.1 units, firmware v2.7.26.54e0d8d + this patch.
