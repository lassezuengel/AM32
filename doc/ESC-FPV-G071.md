# ESC-FPV STM32G071 target

`ESC_FPV_G071` supports the four identical motor-driver channels on the
ESC-FPV board. Each channel has its own STM32G071GBU6 and MP6531A, so the same
firmware image is installed on all four MCUs.

This port is based on the official AM32 v2.21 release. The pre-migration v2.01
port remains available on the `legacy-esc-fpv-v2.01` branch.

## Pin mapping

| Function | STM32 pin | Peripheral |
| --- | --- | --- |
| MP6531 HA | PA8 | TIM1_CH1 |
| MP6531 LA | PA7 | TIM1_CH1N |
| MP6531 HB | PA9 (physical PA11 remapped) | TIM1_CH2 |
| MP6531 LB | PB0 | TIM1_CH2N |
| MP6531 HC | PA10 (physical PA12 remapped) | TIM1_CH3 |
| MP6531 LC | PB1 | TIM1_CH3N |
| Phase A BEMF | PB7 | COMP2 negative input IO2 |
| Phase B BEMF | PB3 | COMP2 negative input IO1 |
| Phase C BEMF | PA2 | COMP2 negative input IO3 |
| BEMF reference | PA3 | COMP2 positive input IO3 |
| DShot / bidirectional DShot | PB4 | TIM3_CH1 |
| MP6531 nSLEEP | PA6 | GPIO output, active high |
| MP6531 nFAULT | PA0 | GPIO input, active low |
| SWDIO | PA13 | SWD |
| SWCLK | PA14 | SWD |
| Reset | PF2 | NRST |

AM32 phase A is TIM1 channel 1 and BEMF_A, phase B is channel 2 and BEMF_B,
and phase C is channel 3 and BEMF_C. Do not substitute the generic G0-A phase
mapping: its logical A/C assignments do not match this schematic.

The G071 port enables the PA11-to-PA9 and PA12-to-PA10 remaps before GPIO
initialization. Consequently, physical PA11 supplies TIM1_CH2/HB and physical
PA12 supplies TIM1_CH3/HC.

PA6 is held low while PWM, phase GPIO, ADC and comparator peripherals are
initialized. It is raised only at the end of peripheral enable, followed by a
2 ms MP6531A wake delay. PA0 is configured as an input; AM32 does not currently
consume nFAULT, while the MP6531A continues to enforce its internal protection.

## 1S/high-KV behavior

The schematic has no voltage or current ADC connections. This target samples
only the STM32 internal temperature sensor; voltage, current and consumption
telemetry report zero. Low-voltage cutoff and current limiting are forced off
in RAM even if stale EEPROM settings request them. Temperature limiting remains
active.

The target also disables AM32's separate KV-derived low-RPM duty ceiling. The
normal startup duty limit, acceleration ramp, stuck-rotor handling, BEMF checks
and temperature limit remain active. This avoids treating the configurator's
10,220-KV storage limit as a throttle ceiling for 1S motors around 22,000 KV.
The Motor KV field therefore does not need a fake `/16` value on this target.

## Build and installation

Build with:

```sh
make ESC_FPV_G071
```

The application starts at `0x08001000` and requires a compatible G071/PB4 AM32
bootloader. Install the same resulting image on all four channels.

Before installing propellers, validate one channel at a time from a
current-limited supply: verify PA6 stays low through reset, rises after phase
initialization, the six MP6531 inputs are inactive during reset, and the driven
phase matches the selected BEMF input.
