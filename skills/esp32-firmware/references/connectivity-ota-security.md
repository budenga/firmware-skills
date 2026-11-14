# Wi-Fi, BLE, NVS, OTA and security on ESP32

## Contents
1. Wi-Fi that survives the real world
2. BLE notes
3. Persistent settings: Preferences / NVS
4. OTA updates
5. Security features (irreversible — read before enabling)

---

## 1. Wi-Fi that survives the real world

Non-blocking connect with timeout and event-driven reconnection (Arduino):
```cpp
#include <WiFi.h>

static volatile bool s_wifiUp = false;

static void onWiFiEvent(WiFiEvent_t event) {
    switch (event) {
    case ARDUINO_EVENT_WIFI_STA_GOT_IP:       s_wifiUp = true;  break;
    case ARDUINO_EVENT_WIFI_STA_DISCONNECTED: s_wifiUp = false; break;   // auto-reconnect handles retries
    default: break;
    }
}

void wifiStart(const char *ssid, const char *pass) {
    WiFi.onEvent(onWiFiEvent);
    WiFi.mode(WIFI_STA);
    WiFi.setAutoReconnect(true);
    WiFi.begin(ssid, pass);
}

bool wifiWaitConnected(uint32_t timeoutMs) {
    const uint32_t t0 = millis();
    while (!s_wifiUp && millis() - t0 < timeoutMs) delay(50);
    return s_wifiUp;
}
```
- Never put an unbounded `while (WiFi.status() != WL_CONNECTED)` in `setup()` — a missing AP then means a dead device.
- Network code (HTTP, MQTT) must handle "not connected" and retry with **exponential back-off + jitter**, not hammer the AP/server.
- Credentials: not hard-coded in the repo. Provision via a captive portal (e.g. WiFiManager library), BLE, or a serial command, and store in NVS.
- MQTT: set keep-alive, last-will, reconnect in the loop/task, and buffer or drop data deliberately while offline.
- TLS: verify the server (CA certificate bundle or pinned CA) — `setInsecure()` is only for experiments. Keep ~50 KB heap free for TLS.
- ESP-IDF: use the event loop (`esp_event_handler_register(WIFI_EVENT/IP_EVENT ...)`) with a reconnect counter, and `esp_netif` APIs.

## 2. BLE notes

- Arduino core 2.x/3.x BLE library (Bluedroid) uses a lot of flash/RAM; **NimBLE-Arduino** (`h2zero/NimBLE-Arduino`) is much lighter and the usual choice.
- Wi-Fi + BLE coexist on one radio: expect lower throughput and higher latency; keep BLE intervals reasonable.
- Classic Bluetooth (SPP, A2DP) exists only on the original ESP32.

## 3. Persistent settings: Preferences / NVS

Arduino:
```cpp
#include <Preferences.h>
Preferences prefs;
prefs.begin("cfg", false);                 // namespace ≤ 15 chars, false = read/write
uint32_t interval = prefs.getUInt("interval", 60);  // default if absent
prefs.putUInt("interval", 120);            // only when the value changed
prefs.end();
```
ESP-IDF init (required before Wi-Fi):
```c
esp_err_t err = nvs_flash_init();
if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());     // partition layout changed or full
    err = nvs_flash_init();
}
ESP_ERROR_CHECK(err);
```
- Keys ≤ 15 characters. NVS is wear-levelled but not for high-rate logging — write on change, rate-limit.
- Add a schema version key; migrate settings on firmware updates.
- Large data (logs, files) → LittleFS partition, not NVS.

## 4. OTA updates

Partition table must have two app slots (`ota_0`, `ota_1`) + `otadata` (see `platformio-setup.md` §4). Each app slot must fit the firmware with headroom for growth.

| Method | Use |
|---|---|
| `ArduinoOTA` | Development on the LAN (upload from PlatformIO with `upload_protocol = espota`, `upload_port = <ip>`) — not for products |
| `HTTPUpdate` / `Update` (Arduino), `esp_https_ota` (IDF) | Device downloads a signed/authenticated image from your server over HTTPS |
| Managed services | If the product already uses a cloud platform with OTA |

Production OTA checklist:
- HTTPS with server verification; image authenticity (Secure Boot signing, or at least a signature/hash check of the image).
- Version check before download; don't downgrade unless intended (anti-rollback needs secure boot features).
- **Rollback**: enable `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE` (needs ESP-IDF or Arduino+IDF `sdkconfig`). New firmware boots in "pending verify" state; after self-test passes (Wi-Fi connects, server reachable), call `esp_ota_mark_app_valid_cancel_rollback()`; if it crashes/reboots first, the bootloader returns to the previous image.
- Power loss during download is safe (old image stays active until the switch); keep it that way — never erase the running slot.
- Log the running version (`esp_app_get_description()->version` on IDF 5 / Arduino core 3.x; `esp_ota_get_app_description()` on IDF 4.4 / core 2.x; or a build flag `-DFW_VERSION=\"1.2.3\"`) at boot and report it.

## 5. Security features (irreversible — read before enabling)

| Feature | What it does | Irreversible part |
|---|---|---|
| Secure Boot v2 | Bootloader and app must be signed with your key | Burns eFuses with the key digest; lose the private key = no more updates |
| Flash encryption | Flash content encrypted with a per-device key | Development mode can be disabled a limited number of times; **release mode is permanent** (no more plain serial flashing) |
| NVS encryption | Encrypts NVS (needs flash encryption or HMAC-based scheme) | Tied to keys in eFuse/flash |
| JTAG / UART download disable | Closes debug and download ports | eFuse, permanent |

Rules:
- These require ESP-IDF configuration (`sdkconfig`), not plain Arduino.
- Never run `espefuse.py burn_*` commands, enable release-mode flash encryption, or enable secure boot on a user's device without an explicit, informed confirmation and a written key-management plan (where the signing key lives, backups, who can sign).
- Test the whole flow (build → sign → flash → OTA → rollback) on sacrificial boards first.
- Development without these features: still never ship default credentials, debug consoles without authentication, or `setInsecure()` TLS.
