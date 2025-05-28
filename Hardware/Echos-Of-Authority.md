# Echos Of Authority - CTF Write-Up

## Challenge Overview

**Challenge Name:** Echos Of Authority  
**Category:** Hardware / Digital Forensics / VoIP Analysis  
**Difficulty:** easy  
**Flag:** `HTB{13513377}`

**Challenge Description:**
> Our Intel-Team has intercepted a VoIP call made by a high-ranking government official. During the call, the official accessed a secure government IVR (Interactive Voice Response) system, which requires a password to proceed.
> 
> Your mission is to analyze the captured call and determine the password he entered.

## Initial Reconnaissance

### Challenge Artifact
- **File Provided:** `.wav` audio file (extracted from Wireshark)
- **Content:** VoIP call recording containing IVR system interaction
- **Target:** Extract the password entered via telephone keypad

### Technical Background
The challenge involves analyzing **DTMF (Dual-Tone Multi-Frequency)** tones generated when pressing numbers on a telephone keypad. Each keypad button produces two simultaneous frequencies that can be decoded to determine the pressed digits.

## Vulnerability Analysis

### DTMF System Analysis
DTMF tones consist of:
- **Low Frequency Component:** One of 697, 770, 852, or 941 Hz
- **High Frequency Component:** One of 1209, 1336, or 1477 Hz
- **Duration:** Typically 250ms per tone
- **Mapping:** Each frequency pair corresponds to a specific keypad digit

### DTMF Frequency Table
```
        1209 Hz  1336 Hz  1477 Hz
697 Hz     1        2        3
770 Hz     4        5        6
852 Hz     7        8        9
941 Hz     *        0        #
```

## Exploitation Strategy

### Step 1: Audio Loading and Visualization

Load the audio file and visualize the waveform:

```python
from pydub import AudioSegment
import matplotlib.pyplot as plt

# Load the audio file
audio = AudioSegment.from_wav("wireshark password.wav")

# Visualize waveform to identify tone bursts
samples = audio.set_channels(1).get_array_of_samples()
plt.plot(samples)
plt.title("DTMF Audio Waveform")
plt.show()
```

This confirms the presence of distinct tone bursts characteristic of DTMF signals.

### Step 2: DTMF Frequency Analysis

Implement frequency analysis to detect DTMF tones:

```python
from scipy.fft import fft, fftfreq
from scipy.signal import find_peaks
import numpy as np

# Define DTMF frequency mapping
dtmf_freqs = {
    (697, 1209): '1', (697, 1336): '2', (697, 1477): '3',
    (770, 1209): '4', (770, 1336): '5', (770, 1477): '6',
    (852, 1209): '7', (852, 1336): '8', (852, 1477): '9',
    (941, 1209): '*', (941, 1336): '0', (941, 1477): '#'
}

def analyze_dtmf_chunk(audio_chunk):
    # Convert to mono and get samples
    samples = audio_chunk.set_channels(1).get_array_of_samples()
    
    # Apply FFT
    fft_result = fft(samples)
    freqs = fftfreq(len(samples), 1/audio_chunk.frame_rate)
    
    # Find peaks in frequency domain
    magnitude = np.abs(fft_result)
    peaks, _ = find_peaks(magnitude, height=np.max(magnitude) * 0.3)
    
    # Extract dominant frequencies
    peak_freqs = freqs[peaks]
    
    # Match to DTMF frequencies
    # ... (frequency matching logic)
    
    return detected_digit
```

### Step 3: Audio Segmentation and Processing

Divide the audio into chunks and analyze each segment:

```python
# Process audio in 250ms chunks (typical DTMF duration)
chunk_duration = 250  # milliseconds
detected_sequence = []

for i in range(0, len(audio), chunk_duration):
    chunk = audio[i:i+chunk_duration]
    digit = analyze_dtmf_chunk(chunk)
    if digit:
        detected_sequence.append(digit)
```

### Step 4: Raw Sequence Analysis

The frequency analysis reveals the following raw DTMF sequence:

```python
raw_sequence = ['1', '1', '1', '3', '3', '3', '5', '5', '5', '1', '1', '1', '3', '3', '3', '3', '3', '3', '7', '7', '7', '7', '7', '7', '#', '#']
```

### Step 5: Pattern Recognition and Interpretation

Analyze the sequence for grouping patterns:

**Grouped Analysis:**
- '1' × 3 repetitions
- '3' × 3 repetitions  
- '5' × 3 repetitions
- '1' × 3 repetitions
- '3' × 6 repetitions (extended)
- '7' × 6 repetitions (extended)
- '#' × 2 repetitions

**Interpretation Logic:**
- Each group of repetitions represents a single digit input
- Extended repetitions (6×) likely indicate deliberate emphasis or double digits
- Shorter repetitions (3×) represent single digits

### Step 6: Password Reconstruction

Based on pattern analysis:

```
Grouped Pattern: [1], [3], [5], [1], [3,3], [7,7]
Final Password: 1, 3, 5, 1, 3, 3, 7, 7
Result: 13513377
```

The 8-digit password is: **13513377**

## Attack Chain Summary

```
1. Audio Analysis → 2. DTMF Detection → 3. Frequency Mapping → 4. Sequence Extraction → 5. Pattern Recognition → 6. Password Reconstruction
```

## Key Technical Concepts

1. **DTMF Analysis:** Understanding dual-tone multi-frequency signaling
2. **FFT (Fast Fourier Transform):** Converting time-domain signals to frequency domain
3. **Signal Processing:** Identifying meaningful patterns in audio data
4. **VoIP Forensics:** Extracting intelligence from voice communications
5. **Pattern Recognition:** Interpreting repeated tone sequences

## Tools and Techniques Used

- **Python Audio Processing:** `pydub` for audio file manipulation
- **Signal Analysis:** `scipy.fft` for frequency domain analysis
- **Peak Detection:** `scipy.signal.find_peaks` for identifying dominant frequencies
- **Data Visualization:** `matplotlib` for waveform analysis
- **Pattern Analysis:** Manual interpretation of tone groupings

## Forensic Methodology

### Audio Forensics Process
1. **Waveform Analysis:** Visual identification of tone bursts
2. **Frequency Domain Conversion:** FFT analysis for precise frequency detection
3. **Peak Detection:** Identifying dominant frequency components
4. **DTMF Mapping:** Converting frequency pairs to keypad digits
5. **Temporal Analysis:** Understanding timing and repetition patterns
6. **Contextual Interpretation:** Applying domain knowledge about IVR systems

### IVR System Behavior
- Users often press and hold buttons, creating repeated tones
- Important digits may be emphasized with longer presses
- System responses can be distinguished from user input
- Termination characters (# or *) typically indicate end of input

## Alternative Analysis Approaches

Other methods that could be applied:
1. **Automated DTMF Decoders:** Tools like `multimon-ng` for direct DTMF decoding
2. **Spectrogram Analysis:** Visual frequency analysis over time
3. **Machine Learning:** Trained models for audio pattern recognition
4. **Commercial VoIP Analysis Tools:** Professional forensics software

## Mitigation Recommendations

### For Secure Communications
1. **Audio Encryption:** Implement end-to-end encryption for sensitive calls
2. **Secure Authentication:** Use multi-factor authentication beyond DTMF
3. **Call Recording Policies:** Strict controls on call recording and storage
4. **Access Monitoring:** Log and monitor all IVR system interactions

### For VoIP Security
1. **Protocol Security:** Secure VoIP protocols (SRTP, TLS)
2. **Network Isolation:** Separate voice and data networks
3. **Intrusion Detection:** Monitor for unauthorized call recording
4. **Key Management:** Secure handling of authentication credentials

## Write-Up Credit: [SR20DET](https://ctf.hackthebox.com/user/profile/605510)