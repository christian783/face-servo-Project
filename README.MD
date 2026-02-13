# Distributed Face-Locked Servo System

A complete distributed vision-control system implementing face tracking with servo motor actuation using MQTT messaging and real-time web visualization.

## 🎯 Project Overview

This system implements a distributed architecture with four main components:

1. **PC Vision Node** - Face detection and tracking using OpenCV
2. **ESP8266 Edge Controller** - Servo motor control via MQTT
3. **Backend API Service** - MQTT to WebSocket bridge
4. **Web Dashboard** - Real-time visualization and monitoring

### Architecture

```
┌─────────────┐         ┌──────────────┐         ┌──────────────┐
│   PC with   │         │   VPS with   │         │   Browser    │
│   Camera    │ ──MQTT──│ MQTT Broker  │──WSS────│  Dashboard   │
│             │         │  + Backend   │         │              │
└─────────────┘         └──────────────┘         └──────────────┘
       │                        │
       │                        │
       │                        │ MQTT
       │                        │
       │                  ┌─────▼────┐
       └──────MQTT────────│ ESP8266  │
                          │  Servo   │
                          └──────────┘
```

## 📋 System Requirements

### PC Vision Node
- Python 3.8+
- Webcam
- OpenCV library
- MQTT client library

### ESP8266 Controller
- ESP8266 development board (NodeMCU recommended)
- MicroPython firmware
- Servo motor (SG90 or similar)
- Power supply (5V for servo)

### Backend Server (VPS)
- Node.js 16+
- MQTT broker (Mosquitto recommended)
- Public IP address or domain

### Web Dashboard
- Modern web browser (Chrome, Firefox, Edge)
- Internet connection

## 🚀 Installation & Setup

### 1. PC Vision Node Setup

```bash
cd pc-vision

# Create virtual environment (recommended)
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Configure the system
# Edit vision_node.py and change:
# - TEAM_ID to your unique team identifier
# - MQTT_BROKER to your VPS IP/domain
```

**Run the vision node:**
```bash
python vision_node.py
```

### 2. ESP8266 Controller Setup

**Prerequisites:**
- ESP8266 flashed with MicroPython firmware
- See Gabriel Baziramwabo's guide: [DOI:10.13140/RG.2.2.35581.88800]

**Hardware Wiring:**

```
ESP8266 (NodeMCU)     Servo Motor
─────────────────     ───────────
5V              ──► VCC (Red)
GND               ──► GND (Brown/Black)
D2 (GPIO4)        ──► Signal (Orange/Yellow)
```

**Firmware Upload:**

```bash
# Install ampy if needed
pip install adafruit-ampy

# Edit main.py configuration:
# - WIFI_SSID and WIFI_PASS
# - TEAM_ID (must match PC node)
# - MQTT_BROKER (your VPS IP)

# Upload to ESP8266
ampy --port /dev/ttyUSB0 put main.py

# Reboot ESP8266
```

**Verify:**
```bash
# Monitor serial output
screen /dev/ttyUSB0 115200
# or
minicom -D /dev/ttyUSB0 -b 115200
```

### 3. Backend Server Setup

**Install MQTT Broker (VPS):**

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install mosquitto mosquitto-clients

# Start Mosquitto
sudo systemctl start mosquitto
sudo systemctl enable mosquitto

# Test MQTT broker
mosquitto_sub -h localhost -t 'test' -v
```

**Run the Backend Service Locally:**

```bash
cd backend

# Install Node.js dependencies
npm install

# Configure server.js:
# - Change TEAM_ID to match your team
# - Verify MQTT_BROKER points to VPS IP

# Run backend
npm start

# Or with auto-restart (development)
npm run dev
```

### 4. Web Dashboard Setup

**Configure Dashboard:**

Edit `web-dashboard/dashboard.html`:

```javascript
const TEAM_ID = 'team01';  // Your team ID
```

**Deployment Options:**

**Option A: Serve Locally**
```bash
cd web-dashboard
python3 -m http.server 8000
# Access at: http://localhost:8000/dashboard.html
```

**Option B: Deploy to VPS**
```bash
# On VPS, install nginx
sudo apt install nginx

# Copy dashboard to web root
sudo cp dashboard.html /var/www/html/

# Access at: http://YOUR_VPS_IP/dashboard.html
```

**Option C: Deploy to GitHub Pages**
1. Push dashboard.html to GitHub repository
2. Enable GitHub Pages in repository settings
3. Access at: https://username.github.io/repo-name/dashboard.html

## 🔧 Configuration

### MQTT Topic Structure

All teams share the same broker, so topic isolation is **CRITICAL**:

```
vision/<TEAM_ID>/movement     - Face movement messages (PC → Broker → ESP8266)
vision/<TEAM_ID>/heartbeat    - System health messages
iot/status/<CLIENT_ID>        - Device online/offline status
```

**Example for team01:**
```
vision/team01/movement
vision/team01/heartbeat
iot/status/esp8266_a1b2c3d4
```

### Message Formats

**Movement Message:**
```json
{
  "status": "MOVE_LEFT",
  "confidence": 0.87,
  "timestamp": 1730000000,
  "face_position": {
    "x": 320,
    "y": 240
  }
}
```

**Status Values:**
- `MOVE_LEFT` - Face is right of center, servo moves left
- `MOVE_RIGHT` - Face is left of center, servo moves right
- `CENTERED` - Face is centered
- `NO_FACE` - No face detected

## 🧪 Testing & Validation

### 1. Test Individual Components

**Test PC Vision:**
```bash
cd pc-vision
python vision_node.py
# You should see: "✓ Connected to MQTT broker"
# Move your face and observe status changes
```

**Test ESP8266:**
```bash
# Monitor serial output
# You should see:
# "✓ Wi-Fi connected"
# "✓ MQTT connected"
# "← MOVE_LEFT (conf: 0.87)"
```

**Test Backend:**
```bash
cd backend
npm start
# You should see:
# "✓ Connected to MQTT broker"
# "✓ WebSocket server listening on port 9002"
```

**Test Dashboard:**
Open dashboard.html in browser
- Status should show "Connected"
- Movement indicators should update in real-time
- Message log should show incoming data

### 2. Integration Testing

**Full System Test:**

1. Start MQTT broker on VPS
2. Start backend service
3. Power on ESP8266 (servo should center at 90°)
4. Run PC vision node
5. Open dashboard in browser
6. Move your face left/right:
   - Dashboard should show movement
   - Servo should physically rotate
   - Message log should update

### 3. Common Issues

| Issue | Solution |
|-------|----------|
| PC can't connect to MQTT | Check firewall, verify broker IP |
| ESP8266 won't connect | Verify WiFi credentials, check serial output |
| Dashboard shows "Disconnected" | Verify WebSocket URL and port 9002 is open |
| Servo doesn't move | Check wiring, verify MQTT messages are received |
| Wrong team data appears | Verify TEAM_ID matches across all components |

## 📊 Performance Optimization

### Reduce Latency
- Publish interval: 100ms (configurable in vision_node.py)
- Servo smoothing: Enabled (reduces jitter)
- MQTT QoS: 1 (guaranteed delivery)

### Prevent Jitter
```python
# In ESP8266 main.py
SMOOTHING_ENABLED = True  # Gradual movement
MOVE_DELAY_MS = 50       # Delay between steps
```

### Network Optimization
- Use retained messages for state (LED status, device online/offline)
- Avoid flooding MQTT with identical messages
- Use WebSocket compression if available

## 📝 Development Phases

### Phase 1: Open-Loop (Current Implementation)
- ✅ Camera fixed on PC
- ✅ Servo rotates based on face movement
- ✅ Camera frame does NOT move with servo
- ✅ Validates MQTT communication and servo control

### Phase 2: Closed-Loop (Future Enhancement)
- ⬜ Mount camera on servo
- ⬜ Camera rotates with servo
- ⬜ True feedback loop
- ⬜ Advanced PID control for stability

## 🎓 Educational Outcomes

This project demonstrates:

1. **Distributed Systems** - Decoupled components communicating via messages
2. **Publish-Subscribe Pattern** - MQTT broker as central message router
3. **Real-Time Control** - Low-latency actuation based on sensor input
4. **Edge Computing** - Processing split between PC (vision) and ESP8266 (actuation)
5. **Web Technologies** - WebSocket for real-time browser updates
6. **Embedded Systems** - MicroPython on resource-constrained hardware

## 📚 References

1. Gabriel Baziramwabo (2025). "ESP8266-Based Environmental Monitoring & Actuator Control Using MQTT"
2. Gabriel Baziramwabo (2025). "Mastering ESP8266 with MicroPython" DOI:10.13140/RG.2.2.35581.88800
3. MicroPython Documentation: https://docs.micropython.org
4. MQTT Protocol Specification: https://mqtt.org/
5. OpenCV Face Detection: https://docs.opencv.org/

## 👥 Team Members

- TUYISHIME Christian - Vision & Integration
- GANZA Chael - ESP8266 & Hardware
- NTARE KAYITARE Prince - Backend & Deployment
- BAGABO Bonny - Dashboard & Documentation

## 📄 License

MIT License - Feel free to use for educational purposes

## 🤝 Acknowledgments

Special thanks to:
- Gabriel Baziramwabo for ESP8266 MQTT implementation guidance
- Rwanda Coding Academy for project framework
- OpenCV community for computer vision tools

---

## 📧 Support

For issues or questions:
1. Check the troubleshooting section above
2. Review Gabriel's documentation on ResearchGate
3. Watch BenaxMedia YouTube Channel tutorials
4. Contact instructor: Gabriel Baziramwabo

---

**Last Updated:** February 2026
**Version:** 1.0.0