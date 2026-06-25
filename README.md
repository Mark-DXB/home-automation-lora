# ESPHome BLE-to-LoRa Bridge

This project enables you to monitor off-grid Bluetooth Low Energy (BLE) sensors over extremely long distances by bridging them through a LoRa radio network into Home Assistant. 

By using LoRa, this architecture bypasses the strict range limitations of Wi-Fi and the latency/bandwidth limitations of forwarding raw Zigbee serial streams.

## Hardware Used
*   **Field Hub (Sender):** Heltec WiFi LoRa 32 (V3) - ESP32-S3 868MHz
*   **Home Gateway (Receiver):** Heltec WiFi LoRa 32 (V3) - ESP32-S3 868MHz
*   **Sensors:** Inkbird IBS-TH2 (Temperature-only variant) or any ESPHome-supported BLE sensor.

## How It Works

1. **The Sensor (Offline):** The Inkbird IBS-TH2 sits completely offline in the field (e.g., in a greenhouse, freezer, or garden). It wakes up periodically and broadcasts its temperature and battery level via standard Bluetooth Low Energy (BLE) advertising packets.
2. **The Field Hub (Sender):** The Heltec V3 "Sender" sits within Bluetooth range of the sensors. It uses ESPHome's `esp32_ble_tracker` to passively listen for those broadcasts without needing to formally pair. It decodes the temperature data locally and then injects that data into a LoRa radio payload using the `packet_transport` custom component.
3. **The Home Gateway (Receiver):** The Heltec V3 "Receiver" sits inside the house and is connected to the local Wi-Fi and Home Assistant. It listens on the 868MHz LoRa frequency. When it receives a packet from the Sender, it parses the payload and exposes the remote temperature, battery, and signal strength directly to Home Assistant as native entities.

## Configuration Files

*   **`lora-sender.yaml`:** Flashed to the field node. Includes the `esp32_ble_tracker`, the `inkbird_ibsth1_mini` sensor definition, and the `packet_transport` configuration to broadcast the data over the `sx126x` radio. It also features a custom OLED display routine to show uptime and link status.
*   **`lora-receiver.yaml`:** Flashed to the home node. Includes the `packet_transport` configuration to listen for incoming LoRa data and map it to Home Assistant sensor entities. It also drives the local OLED display to show the last received data and local Wi-Fi strength.

## Scaling
To add more sensors (like soil moisture meters or door contacts) to the field, you do not need additional LoRa hardware. Simply configure the new BLE sensor's MAC address in the `lora-sender.yaml`, add its ID to the `packet_transport` block, and create a matching receiving sensor in `lora-receiver.yaml`.

## UK Deployment & Commissioning Guide
When transferring this setup from the test bench to the live UK deployment (2BG Home Assistant & Allotment), follow these exact steps to ensure the off-grid node functions correctly without killing its battery.

### 1. Update ESPHome
Ensure the ESPHome Add-on on your UK Home Assistant server is fully up to date so that it includes native support for the `packet_transport` component.

### 2. Configure the Receiver (Home Node)
1. Open the ESPHome dashboard in your UK Home Assistant.
2. Create a new device named `lora-receiver` and paste the contents of `lora-receiver.yaml` into it.
3. Update the `wifi:` block to use your UK Wi-Fi credentials (or ensure your UK `secrets.yaml` matches).
4. Flash the device.

### 3. Configure the Sender (Off-Grid Node) - CRITICAL STEP
1. Create a new device named `lora-sender` in the UK ESPHome dashboard and paste the contents of `lora-sender.yaml`.
2. **WARNING:** Before you hit install, you **MUST delete the `wifi:`, `api:`, and `ota:` blocks completely.**
    *   *Why?* If the ESP32 is instructed to look for Wi-Fi and Home Assistant (`api:`) but cannot find them in the middle of a field, a safety watchdog will automatically reboot the chip every 15 minutes. This constant searching and rebooting will destroy your solar battery life.
3. Flash the device via USB. The Sender will now run fully off-grid, permanently listening for Bluetooth and transmitting over LoRa without relying on Wi-Fi.
