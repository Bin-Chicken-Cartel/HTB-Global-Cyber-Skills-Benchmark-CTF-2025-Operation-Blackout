# HTB Operation Blackout - Uplink Artefact

## Challenge Overview

**Challenge Name:** Uplink Artefact  
**Category:** Machine Learning / Cryptography  
**Difficulty:** very easy  
**Flag:** `HTB{clu5t3r_k3y_l34k3d}`

During an analysis of a compromised satellite uplink, a suspicious dataset was recovered. Intelligence indicates it may encode physical access credentials hidden within the spatial structure of Volnaya's covert data infrastructure.

## Given Files

- `uplink_spatial_auth.csv` - A CSV file containing 1,822 rows of 3D coordinate data
- `Uplink_Artefact_CTF_QR_Code.svg` - Generated QR code for flag extraction

## Technical Analysis

### Dataset Structure
The CSV contains four columns:
- `x`: X-coordinate (float, 0-25 range)  
- `y`: Y-coordinate (float, 0-25 range)  
- `z`: Z-coordinate (float, 0-1 range)  
- `label`: Integer label (0, 1, 2, or 3)

### Vulnerability Assessment
The data showed 1,822 total points with balanced label distribution across a 25x25 grid range, suggesting a 2D pattern when plotted.

## Attack Chain

### 1. Data Exploration
```bash
# Count label distribution
cut -d',' -f4 uplink_spatial_auth.csv | tail -n +2 | sort | uniq -c
    525 0
    322 1  
    518 2
    457 3
```

### 2. Pattern Identification
Filtered for points with integer coordinates and plotted them by label. Label 1 revealed a distinctive QR code pattern with recognizable finder patterns in the corners.

### 3. QR Code Extraction
```python
def create_qr_matrix():
    qr_points = set()
    
    with open("uplink_spatial_auth.csv", 'r') as f:
        reader = csv.reader(f)
        next(reader)
        
        for row in reader:
            x, y, z, label = float(row[0]), float(row[1]), float(row[2]), int(row[3])
            if label == 1 and x == int(x) and y == int(y):
                qr_points.add((int(x), int(y)))
    
    return qr_points
```

### 4. QR Code Generation
Generated the QR code in SVG format for optimal scanning quality:

```python
# SVG format for best scanning quality
with open("qr_scannable.svg", "w") as f:
    module_size = 20
    quiet_zone = 4 * module_size
    total_size = (25 * module_size) + (2 * quiet_zone)
    
    # SVG header and white background
    f.write(f'<svg width="{total_size}" height="{total_size}" xmlns="http://www.w3.org/2000/svg">')
    f.write(f'<rect width="{total_size}" height="{total_size}" fill="white"/>')
    
    # Black modules for QR code
    for y in range(25):
        for x in range(25):
            if (x, 24-y) in qr_points:
                svg_x = quiet_zone + (x * module_size)
                svg_y = quiet_zone + (y * module_size)
                f.write(f'<rect x="{svg_x}" y="{svg_y}" width="{module_size}" height="{module_size}" fill="black"/>')
    
    f.write('</svg>')
```

**Note:** The generated QR code is available as `Uplink_Artefact_CTF_QR_Code.svg` for scanning.

## Key Technical Concepts

### Spatial Data Analysis
- 3D coordinate data can encode 2D patterns when properly filtered and plotted
- Label-based filtering reveals hidden structures within seemingly random datasets

### QR Code Structure
- 25x25 grid corresponds to QR Code Version 1 (up to 25 alphanumeric characters)
- Integer coordinates represent QR code modules (black squares)
- Distinctive finder patterns in corners enable automatic recognition

### Steganographic Techniques
- Data hidden in coordinate space rather than traditional image steganography
- Noise data (labels 0, 2, 3) obscures the actual QR pattern (label 1)

## Mitigation Recommendations

### Data Security
- Implement data validation to detect anomalous coordinate patterns
- Use entropy analysis to identify potential steganographic content
- Apply clustering algorithms to detect suspicious data groupings

### Access Control
- Restrict access to coordinate data that could contain embedded information
- Implement monitoring for unusual data export patterns
- Use data loss prevention tools to detect QR code patterns in datasets

## Tools Used

- Python for data analysis and QR code generation
- CSV processing libraries for data extraction
- QR code scanner applications for flag extraction

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)