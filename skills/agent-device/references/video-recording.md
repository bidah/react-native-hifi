# Video Recording

Capture device automation sessions as video for debugging, documentation, or verification.

## iOS Simulator

```bash
agent-device record start ./recordings/ios.mov

# Perform actions
agent-device open App
agent-device snapshot -i
agent-device click @e3
agent-device close

agent-device record stop
```

## Android Emulator/Device

```bash
agent-device record start ./recordings/android.mp4

# Perform actions
agent-device open App
agent-device snapshot -i
agent-device click @e3
agent-device close

agent-device record stop
```
