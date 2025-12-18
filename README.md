# Rx_pcm_TLM_TX – GNU Radio Loopback Transceiver

## Overview

This repository contains a **GNU Radio flowgraph implementing a PCM/FM (CPM-based) telemetry transceiver**, designed to operate both in:

* **Pure software loopback mode** (no external hardware required)
* **Real RF mode** using supported SDR hardware (e.g. ADALM-Pluto, FMCOMMS2)

In loopback mode, the system transmits and receives a **pre-generated CCSDS-like packet** entirely in software. This configuration is especially useful for:

* Understanding the overall functionality of the signal processing chain
* Debugging and validating TX/RX algorithms
* Verifying synchronization and packet detection
* Computing and monitoring the Bit Error Rate (BER)
* Aligning the received signal with the known transmitted bitstream

Once validated in loopback, the same flowgraph can be easily adapted to operate with real RF hardware.

---

## System Architecture

The flowgraph is logically divided into the following sections:

1. Bitstream source and packet definition
2. Transmitter (PCM/FM – CPM modulation)
3. Channel model / loopback path
4. Receiver front-end
5. Synchronization and packet detection
6. Demodulation and bit recovery
7. BER computation and monitoring
8. Optional hardware interfaces

---

## ASCII Block Diagram

```
+------------------+       +----------------------+       +------------------+
|  Bitstream File  | ----> |  PCM/FM (CPM) TX     | ----> |  Channel /       |
|  (CCSDS Packet)  |       |  Modulator           |       |  Loopback Model  |
+------------------+       +----------------------+       +------------------+
                                                                   |
                                                                   v
+------------------+       +----------------------+       +------------------+
|  Synchronization | <---- |  RX Front-End        | <---- |  Received Signal |
|  & Packet Detect |       |  (Filter, AGC, Sync) |       +------------------+
+------------------+       +----------------------+
        |
        v
+------------------+       +----------------------+       +------------------+
|  CPM / FM        | ----> |  Bit Decisions       | ----> |  BER Computation |
|  Demodulation    |       |  & Alignment         |       |  (TX vs RX)      |
+------------------+       +----------------------+       +------------------+
```

---

## 1. Bitstream Source and Packet Structure

### Bitstream File Source

* **Block**: `blocks_file_source`
* **Input file**: `bitstream_in`
* **Mode**: Repetition enabled

The input file contains a **pre-generated telemetry packet** composed of:

* A **CCSDS preamble**
* Payload data bits

The repetition mode ensures continuous transmission, which simplifies synchronization, correlation, and long BER measurements.

### Preamble Definitions

Several versions of the CCSDS preamble are defined inside the flowgraph:

* Binary representation
* Antipodal representation (±1 or ±255)
* CPM-modulated version used as a sync word

These are used by the receiver for **correlation-based packet detection**.

---

## 2. Transmitter Chain (TX)

### PCM/FM (CPM) Modulator

* **Block**: `digital_cpmmod_bc`
* **Modulation type**: Continuous Phase Modulation (CPM)
* **Pulse shape**: LRC

Key parameters include:

* Modulation index `h`
* Samples per symbol `sps`
* Pulse length `L`

This configuration emulates **PCM/FM telemetry modulation**, commonly used in space communication systems.

### TX Monitoring

QT GUI sinks are used to visualize:

* Time-domain waveform
* Frequency spectrum
* Constellation / phase evolution

These tools are helpful for debugging and understanding the modulation behavior.

---

## 3. Channel Model / Loopback Path

### Software Channel Model

* **Block**: `channels_channel_model`

In loopback mode, this block provides an ideal or near-ideal channel, with optional:

* Frequency offset
* Additive noise

When noise is disabled, the system behaves as a perfect internal loopback, transmitting and receiving the signal entirely in software.

---

## 4. Receiver Front-End (RX)

### Filtering and AGC

* **Low-pass filtering** removes out-of-band noise and artifacts
* **AGC (`analog_agc_xx`)** normalizes the signal amplitude

### Symbol Timing Recovery

* **Block**: `digital_symbol_sync_xx`
* **Timing Error Detector**: Gardner TED

This stage aligns the received samples with the symbol timing of the transmitter.

---

## 5. Synchronization and Packet Detection

### Correlation-Based Detection

* **Block**: `digital_corr_est_cc`

The receiver correlates the incoming signal with the CPM-modulated CCSDS sync word. When the correlation peak is detected, a tag indicating the **start of the packet** is generated.

### Stream Alignment

* **Blocks**: `blocks_tagged_stream_align`

These blocks ensure that the received bitstream is correctly aligned with the transmitted one before BER computation.

---

## 6. Demodulation and Bit Recovery

### CPM / FM Demodulation

The demodulation chain includes:

* Quadrature demodulation
* Phase differencing and delay blocks
* Frequency-to-symbol conversion

This process extracts the transmitted NRZ bitstream from the received CPM/FM signal.

### Bit Decisions

Thresholding blocks convert soft demodulated values into binary decisions suitable for BER analysis.

---

## 7. BER Computation

### BER Measurement

* **Block**: `fec_ber_bf`

The recovered bitstream is compared against the original transmitted bitstream read from the file source. The resulting **Bit Error Rate (BER)** is displayed in real time using QT GUI numeric sinks.

---

## 8. Hardware Support (Optional)

The flowgraph includes **disabled SDR source and sink blocks** that can be enabled to move from loopback mode to real RF transmission:

* **ADALM-Pluto** (`iio_pluto_source`, `iio_pluto_sink`)
* **FMCOMMS2** (`iio_fmcomms2_source`, `iio_fmcomms2_sink`)

By disabling the software channel model and enabling these blocks, the same signal processing chain can be tested on real hardware.

---

## Purpose of the Loopback Configuration

The loopback configuration:

* Runs without any external hardware
* Transmits and receives the signal internally
* Provides a controlled environment to:

  * Understand the signal chain
  * Debug synchronization and detection
  * Validate BER computation
  * Prepare the system for RF deployment

---

## Conclusion

This project implements a **complete end-to-end PCM/FM telemetry transceiver** in GNU Radio. The loopback mode enables safe and repeatable testing, while the modular design allows a seamless transition to real RF hardware for over-the-air experiments.
