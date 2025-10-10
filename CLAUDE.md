# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a LoRa (Long Range) communication system simulation project implementing the Frequency Shift Chirp Modulation (FSCM) as described in Vangelista's paper "Frequency Shift Chirp Modulation: The LoRa Modulation". The implementation is in Python using Jupyter notebooks and focuses on reproducing the theoretical BER/SER curves under different channel conditions.

## Development Environment

- **Primary file**: `Proyecto-LoRa.ipynb` - Main Jupyter notebook containing all implementation
- **Language**: Python with NumPy and Matplotlib
- **Resources**: `Recursos/` directory contains reference materials (e.g., `image.png` showing Vangelista's BER curves)

### Running the Code

```bash
# Launch Jupyter notebook
jupyter notebook Proyecto-LoRa.ipynb

# Or use JupyterLab
jupyter lab Proyecto-LoRa.ipynb
```

## System Architecture

The implementation follows a layered modular design mirroring the LoRa PHY specification:

### 1. Symbol Encoding/Decoding Layer

**Encoder** (`coder` function): Converts groups of SF bits into integer symbols (0 to 2^SF-1)
- Uses binary-to-decimal conversion: `s = Σ(w[h] * 2^h)` for h=0 to SF-1
- Validates SF ∈ {7, 8, 9, 10, 11, 12}
- Ensures bit array length is multiple of SF

**Decoder** (`decoder` function): Reverse operation using bit-level operations
- Extracts bits using right-shift (`>>`) and AND (`&`) operations
- Reconstructs original bit sequence from symbols

### 2. Waveform Generation Layer

**Waveform Former** (`waveform_former` function): Implements Equation (2) from Vangelista
- Generates chirp signals: `c(nTs + kT) = (1/√M) * exp(j2π * [(s + k mod M) * k / M])`
- Parameters: SF (Spreading Factor), T (sample period), Bw (bandwidth)
- Returns matrix: rows = symbols, columns = complex samples (M = 2^SF samples per chirp)
- Each symbol maps to a unique chirp with frequency offset

**Chirp Primitives**:
- `upchirp(SF, T, Bw)`: Base up-chirp (symbol s=0), frequency increases linearly
- `downchirp(SF, T, Bw)`: Conjugate of upchirp, used for demodulation

### 3. Demodulation Layer

**n-Tuple Former** (`n_tuple_former` function): Optimal receiver implementing Section III of paper
1. Multiplies received chirp by downchirp (dechirping): `d(k) = r(k) * exp(-j2π * k²/M)`
2. Applies FFT to convert to frequency domain
3. Detects symbol as index of maximum magnitude: `ŝ = argmax|FFT{d(k)}|`

**Dechirping** (`dechirp` function): Window-based processing for frame detection
- Supports zero-padding for frequency resolution enhancement
- Returns peak value and bin index for synchronization

### 4. Frame Structure (LoRa Packet)

**Frame Builder** (`build_tx_frame` function): Constructs standard LoRa packet
- **Preamble**: Np up-chirps (default 8) for synchronization and channel estimation
- **SFD (Start Frame Delimiter)**: 2.25 down-chirps marking payload start
- **Payload**: Encoded data symbols as continuous chirp sequence

**Frame Processor** (`process_frame` function): Receiver-side frame handling
1. Strips preamble (8M samples) and SFD (2.25M samples)
2. Segments payload into M-sample chirp windows
3. Applies n_tuple_former to recover symbols

**Preamble Detection** (`detect_preamble` function): Sliding window detector
- Searches for consecutive up-chirps with consistent FFT peak bins
- Returns start index of detected preamble or -1 if not found

### 5. Channel Models

**AWGN Channel** (`agregacion_AWNG` function):
- Normalizes chirp power to 1
- Adds complex Gaussian noise: σ² = 1/(2*SNR_linear)
- Models flat fading with white noise

**Frequency-Selective Channel** (`canal_selectivo_frecuencia` function):
- Two-path model: `h(nT) = √0.8·δ(nT) + √0.2·δ(nT-T)`
- Direct path (80% power) + delayed path (20% power)
- Implemented via discrete convolution

### 6. Performance Metrics

**BER** (`ber` function): Bit Error Rate = mean(bits_tx ≠ bits_rx)

**SER** (`ser` function): Symbol Error Rate = mean(symbols_tx ≠ symbols_rx)
- Note: SER ≥ BER always (one symbol error can cause multiple bit errors)

## Key Implementation Details

### Normalization and Power
- Chirps normalized by `1/√M` to maintain constant energy per symbol
- AWGN channel normalizes input power before adding noise

### FFT-Based Demodulation
- Efficient O(M log M) complexity vs O(M²) correlation
- Transforms chirp multiplication into frequency-domain peak detection
- Zero-padding can improve frequency resolution in detection

### Sample Counts
- Each chirp: M = 2^SF samples
- Oversampling factor: 1/(Bw*T), typically 1 for baseband simulation
- Frame lengths depend on payload size and preamble configuration

## Simulation Parameters

### Standard Configuration
```python
SF = 7              # Spreading Factor (7-12)
M = 2**SF          # 128 samples per chirp for SF=7
Bw = 1             # Bandwidth (normalized)
T = 1/Bw           # Sample period
```

### BER/SER Curve Generation
- **Flat FSCM**: SNR range -10 to -6 dB, ~140k bits per point for accurate BER at -7dB
- **Freq-Selective**: SNR range -8 to -2 dB, ~200k bits per point for accurate BER at -3dB
- Results should match Vangelista's curves (e.g., BER ≈ 0.02 at -10dB for flat channel)

## Validation Approach

The implementation validates correctness by:
1. Perfect reconstruction (BER=0, SER=0) without channel impairments
2. Matching published BER/SER curves under AWGN and frequency-selective channels
3. Visual inspection of chirp waveforms and spectrograms

## Common Workflow

1. **Generate random bits** → `bits_tx = np.random.randint(0, 2, size=N*SF)`
2. **Encode to symbols** → `simbolos_tx = coder(bits_tx, SF)`
3. **Generate chirps** → `chirps_tx = waveform_former(simbolos_tx, SF, T, Bw)`
4. **Apply channel** → `chirps_rx = agregacion_AWNG(chirps_tx, snr_db)`
5. **Demodulate** → `simbolos_rx = n_tuple_former(chirps_rx, SF, T, Bw)`
6. **Decode** → `bits_rx = decoder(simbolos_rx, SF)`
7. **Measure performance** → `ber(bits_tx, bits_rx)`, `ser(simbolos_tx, simbolos_rx)`

For frame-based transmission:
1. **Build frame** → `trama_tx = build_tx_frame(simbolos_tx, SF, preamble_len=8)`
2. **Transmit through channel** → `trama_rx = agregacion_AWNG(trama_tx, snr_db)`
3. **Detect and process** → `simbolos_rx = process_frame(trama_rx, SF, preamble_len=8)`

## Branch Information

- **Current branch**: `etapa_2` (frame detection and processing implementation)
- **Main branch**: `main`
- Recent focus: Frame detection ("detector de trama"), dechirping, frame implementation
