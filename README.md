# EEG CMOS Front-End Development

This repository documents the step-by-step development of a simplified EEG signal model and an ideal system-level analog front end in LTspice.

The long-term goal is to move from:

EEG signal understanding  
→ behavioral EEG modeling  
→ ideal front-end architecture  
→ block-level verification  
→ CMOS implementation

The project is intentionally developed in small verified stages so that each block can be understood and tested before replacing it with a transistor-level CMOS circuit.

---

## Project Goal

The complete development flow is:

EEG understanding  
→ EEG behavioral signal generation  
→ differential signal generation  
→ common-mode interference modeling  
→ electrode-interface modeling  
→ ideal analog front end  
→ ideal ADC  
→ CMOS implementation

The current repository contains the work up to the ideal analog front-end stage.

---

# 1. EEG Signal Modeling

The first stage of the project was to understand EEG signal characteristics and create a controllable EEG test stimulus in LTspice.

The main EEG frequency bands considered were:

- Delta
- Theta
- Alpha
- Beta

The initial signal was modeled as the sum of sinusoidal components.

V_EEG = V_delta + V_theta + V_alpha + V_beta

Initial engineering test values:

| Band | Frequency | Amplitude |
|---|---:|---:|
| Delta | 2 Hz | 20 µV |
| Theta | 6 Hz | 10 µV |
| Alpha | 10 Hz | 30 µV |
| Beta | 20 Hz | 5 µV |

The individual components were simulated separately and then combined.

The reconstruction was verified using:

`V(EEG)-V(DELTA)-V(THETA)-V(ALPHA)-V(BETA)`

The residual was approximately zero, confirming correct signal reconstruction.

---

# 2. Differential EEG Signal

The EEG signal was then converted into a differential signal suitable for an analog front end.

The two input signals were modeled as:

VINP = VCM + VEEG/2

VINN = VCM - VEEG/2

with:

VCM = 0.9 V

Therefore:

VINP - VINN = VEEG

This was verified directly in LTspice.

---

# 3. 50 Hz Common-Mode Interference

Power-line interference was added as a common-mode disturbance.

The common-mode signal was modeled as:

VCM_NOISY = 0.9 V + 100 mV × sin(2π × 50 × t)

The differential inputs became:

VINP = VCM_NOISY + VEEG/2

VINN = VCM_NOISY - VEEG/2

For perfectly matched input paths, the 50 Hz common-mode component cancels in the differential signal.

---

# 4. Electrode Impedance Mismatch

To study common-mode-to-differential conversion, unequal electrode impedances were introduced.

Initial resistive model:

Positive electrode resistance = 50 kΩ

Negative electrode resistance = 70 kΩ

A finite input resistance of:

100 MΩ

was added from each input node to a 0.9 V bias reference.

Because the two electrode paths were different, part of the 50 Hz common-mode signal appeared as a differential error.

The error was isolated using:

`(V(INP)-V(INN))-V(EEG)`

A residual of approximately 20 µV at 50 Hz was observed.

This is important because the residual interference becomes comparable to the EEG signal itself.

---

# 5. Frequency-Dependent Electrode Model

The simple resistive electrode model was then replaced by a first-order electrode-skin interface model.

The model used was:

Rs + (Rct || Cdl)

Positive electrode example:

Rs = 10 kΩ  
Rct = 500 kΩ  
Cdl = 100 nF

Negative electrode example:

Rs = 15 kΩ  
Rct = 700 kΩ  
Cdl = 68 nF

The two electrode models were intentionally mismatched.

This makes the electrode impedance frequency-dependent and introduces both attenuation and phase differences between the two channels.

---

# 6. Literature-Informed Dynamic EEG Model

A second EEG source called:

`EEG_FINAL`

was developed to represent a simplified change in physiological state.

The objective was not to reproduce a clinical EEG recording, but to create a literature-informed behavioral test signal.

The modeled sequence was:

0–4 s: eyes closed  
4–7 s: eyes open  
7–10 s: eyes closed

During the eyes-closed interval:

- Alpha activity is dominant
- Beta activity is lower

During the eyes-open interval:

- Alpha activity is strongly reduced
- Beta activity is increased

The dynamic components are:

`DELTA_DYNAMIC`

`THETA_DYNAMIC`

`ALPHA_DYNAMIC`

`BETA_DYNAMIC`

The final EEG signal is:

EEG_FINAL = DELTA_DYNAMIC + THETA_DYNAMIC + ALPHA_DYNAMIC + BETA_DYNAMIC

The final reconstruction was verified using:

`V(EEG_FINAL)-V(DELTA_DYNAMIC)-V(THETA_DYNAMIC)-V(ALPHA_DYNAMIC)-V(BETA_DYNAMIC)`

The residual remained only in the picovolt range, confirming correct reconstruction.

---

# 7. Reusable EEG Input Model

The completed EEG generator and electrode-interface model were converted into a reusable hierarchical LTspice block:

`EEG_INPUT_MODEL`

The internal model contains:

- Dynamic EEG generation
- 50 Hz common-mode interference
- Differential input generation
- Frequency-dependent electrode impedance
- Electrode mismatch
- Input resistance
- Bias reference

Only two signals are exposed to the top-level analog front end:

`INP`

`INN`

This keeps the AFE schematic clean and modular.

---

# 8. Ideal Analog Front End

A separate LTspice schematic was created for the ideal system-level analog front end.

The current signal chain is:

EEG_INPUT_MODEL  
→ Ideal Differential Amplifier  
→ High-Pass Filter  
→ Low-Pass Filter  
→ Post-Gain Stage  
→ AFE_OUT

---

## 8.1 Ideal Differential Amplifier

The first ideal amplifier provides differential gain.

The behavioral equation is:

`V=20*(V(INP)-V(INN))`

The first-stage gain is:

G1 = 20 V/V

Before connecting the EEG model, the amplifier was independently verified using a simple differential test signal.

Test input:

50 µV peak at 10 Hz

Expected output:

50 µV × 20 = 1 mV peak

The gain was cross-verified using:

`V(AFE_AMP_OUT)-20*(V(INP)-V(INN))`

The residual was approximately zero.

After this standalone verification, the reusable `EEG_INPUT_MODEL` block was connected to the amplifier.

---

# 8.2 High-Pass Filter

A first-order high-pass filter was added after the differential amplifier.

The purpose of this stage is to remove DC offset and very-low-frequency drift.

Component values:

RHP = 1 MΩ

CHP = 330 nF

The approximate cutoff frequency is:

fc ≈ 0.48 Hz

The filter was cross-verified by temporarily adding a 50 mV DC offset to the amplifier output.

Before the filter:

EEG signal + 50 mV DC offset

After the filter:

EEG signal centered approximately around 0 V

This confirmed that the high-pass filter successfully removes the DC component while preserving the EEG waveform.

---

# 8.3 Low-Pass Filter

A first-order low-pass filter was added to limit the upper signal bandwidth.

Component values:

RLP = 10 kΩ

CLP = 390 nF

The approximate cutoff frequency is:

fc ≈ 40.8 Hz

To verify the filter independently, a temporary 200 Hz test signal with 1 mV peak amplitude was applied.

Observed result:

HP_OUT ≈ 1 mV peak

LP_OUT ≈ 0.2 mV peak

This matches the expected attenuation of a first-order low-pass filter at 200 Hz.

After verification, the temporary 200 Hz signal was removed and the normal EEG signal path was restored.

---

# 8.4 Post-Gain Stage

A second ideal gain stage was added after filtering.

The behavioral equation is:

`V=10*V(LP_OUT)`

The post-gain is:

G2 = 10 V/V

The total nominal analog gain is therefore:

GTOTAL = G1 × G2

GTOTAL = 20 × 10

GTOTAL = 200 V/V

The gain stage was cross-verified using:

`V(AFE_OUT)-10*V(LP_OUT)`

The residual remained approximately zero.

---

# Current Ideal AFE Specification

| Parameter | Current Value |
|---|---:|
| First-stage differential gain | 20 V/V |
| High-pass cutoff | ~0.5 Hz |
| Low-pass cutoff | ~40 Hz |
| Post gain | 10 V/V |
| Total nominal gain | 200 V/V |
| Input signal amplitude | Tens to ~100 µV |
| Output signal amplitude | Tens of mV |
| Common-mode interference | 50 Hz |
| Electrode impedance model | Frequency-dependent |

---

# Reference Architecture

Commercial EEG front ends such as the Texas Instruments ADS1299 are used as reference architectures and performance benchmarks.

The ADS1299 is not directly implemented in this project.

Instead, it is used to understand and compare:

- Low-noise EEG amplification
- Programmable gain
- Common-mode rejection
- Input dynamic range
- ADC resolution
- Sampling rate
- Biasing
- Electrode connection
- Integrated EEG acquisition architecture

The final goal is to design our own CMOS implementation of the required blocks.

---

# Current Development Status

## Completed

- [x] EEG frequency-band modeling
- [x] Composite EEG signal generation
- [x] Differential EEG representation
- [x] 50 Hz common-mode interference modeling
- [x] Electrode impedance mismatch
- [x] Frequency-dependent electrode model
- [x] Dynamic alpha/beta EEG behavior
- [x] Reusable hierarchical EEG input block
- [x] Ideal differential amplifier
- [x] High-pass filter
- [x] Low-pass filter
- [x] Post-gain stage
- [x] System-level cross-verification

## In Progress

- [ ] ADC assumption selection
- [ ] Ideal sample-and-hold
- [ ] ADC quantization model
- [ ] Complete ideal acquisition-chain verification

## Future Work

- [ ] Define transistor-level front-end requirements
- [ ] CMOS low-noise amplifier / PGA
- [ ] CMOS bias circuitry
- [ ] CMOS filter implementation
- [ ] Input-referred noise analysis
- [ ] CMRR analysis
- [ ] PSRR analysis
- [ ] PVT analysis
- [ ] Monte Carlo analysis
- [ ] Layout
- [ ] Post-layout verification
- [ ] ADC architecture evaluation

---

# Documentation

Detailed reports are included in this repository.

## EEG Behavioral Signal Modeling Report

Documents:

- EEG signal generation
- Individual EEG bands
- Differential signal generation
- Common-mode interference
- Electrode mismatch
- Frequency-dependent electrode model
- Dynamic alpha/beta behavior
- Final EEG signal verification

## Ideal AFE System-Level Implementation Report

Documents:

- Ideal differential amplifier
- Standalone amplifier verification
- Hierarchical EEG block integration
- High-pass filter design
- DC-removal verification
- Low-pass filter design
- 200 Hz attenuation verification
- Post-gain stage
- Cross-verification of the complete analog chain

---

# Tools

- LTspice
- Behavioral voltage sources
- Hierarchical LTspice schematics
- Literature-based EEG modeling

Future transistor-level implementation will use a suitable CMOS PDK.

---

# Important Note

The EEG waveform used in this project is a simplified behavioral engineering model.

It is intended for:

- Circuit-development learning
- Analog front-end verification
- System architecture exploration
- CMOS design development

It should not be interpreted as a clinical EEG recording or diagnostic model.

---

# Long-Term Objective

The long-term goal is to progressively replace the ideal system-level blocks with CMOS implementations.

Ideal differential amplifier  
→ CMOS low-noise amplifier / PGA

Ideal high-pass and low-pass filters  
→ CMOS analog filtering / DC-servo architecture

Ideal post-gain stage  
→ CMOS programmable gain stage

Ideal ADC  
→ ADC architecture suitable for EEG acquisition

The complete project flow is:

Physiological EEG behavior  
→ Behavioral signal model  
→ Ideal system-level front end  
→ Transistor-level CMOS blocks  
→ Complete CMOS EEG front end
