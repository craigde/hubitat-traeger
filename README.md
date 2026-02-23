# Traeger WiFire Integration for Hubitat Elevation

Real-time Traeger grill integration for Hubitat using the WiFire API. Gets live temperature updates, pellet level, probe state, and full grill control — no polling, no Raspberry Pi, no local broker required.

## Features

- **Real-time updates** via MQTT over WebSocket (AWS IoT)
- **Full grill control** — set temperature, probe target, SuperSmoke, KeepWarm, timers, shutdown
- **5 automation buttons** for Rule Machine — probe at temp, preheat done, pellets low, etc.
- **Dashboard-ready** attributes for all key grill state
- **Multi-grill support** — each grill gets its own child device
- **Auto-reconnect** if the WebSocket drops

---

## How It Works

Traeger's app uses AWS Cognito authentication and AWS IoT MQTT over WebSocket for real-time grill state. Hubitat's built-in `interfaces.mqtt` can't connect to AWS IoT pre-signed WSS URLs, so this driver uses `interfaces.webSocket` with the MQTT protocol implemented manually as binary frames — the same technique used in the [Mysa thermostat driver](https://github.com/craigdewar/hubitat-mysa).

Commands (set temp, shutdown, etc.) are sent via Traeger's REST API. State updates arrive in real-time via MQTT.

---

## Installation

### Prerequisites
- Hubitat Elevation hub
- Traeger WiFire-enabled grill
- Your Traeger app login credentials

### Steps

1. In Hubitat, go to **Drivers Code** → **New Driver** → paste the contents of `TraegerGrillDriver.groovy` → **Save**
2. Go to **Apps Code** → **New App** → paste the contents of `TraegerApp.groovy` → **Save**
3. Go to **Apps** → **Add User App** → select **Traeger Integration**
4. Enter your Traeger email and password
5. Click **Discover Grills** — your grills will appear
6. Click **Create Devices** — child devices are created automatically
7. Click **Done**

The driver will connect to MQTT automatically within a few seconds. Check the device page — `Mqtt Status` should show `connected`.

---

## Device Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `grillTemperature` | number | Current grill temperature |
| `grillSetTemperature` | number | Target grill temperature |
| `probeTemperature` | number | Meat probe temperature |
| `probeSetTemperature` | number | Probe target temperature |
| `ambientTemperature` | number | Controller electronics temperature (not outdoor air) |
| `pelletLevel` | number | Pellet hopper level (0–100%) |
| `grillState` | string | idle, igniting, preheating, manual\_cook, custom\_cook, cool\_down, shutdown, offline |
| `heatingState` | string | idle, igniting, preheating, heating, at\_temp, over\_temp, cool\_down |
| `probeState` | string | idle, set, close, at\_temp, fell\_out |
| `superSmoke` | string | on / off |
| `keepWarm` | string | on / off |
| `timerActive` | string | true / false |
| `timerRemaining` | string | e.g. "45m 12s" |
| `probeAlarmFired` | string | true when probe has reached target |
| `cookTimerComplete` | string | true when cook timer has finished |
| `errors` | number | Error count from grill |
| `temperatureUnit` | string | F or C |
| `connected` | string | true / false |
| `mqttStatus` | string | connecting, connected, disconnected, error |
| `lastUpdate` | string | Timestamp of last MQTT message |

---

## Commands

| Command | Description |
|---------|-------------|
| `setGrillTemperature` | Set grill target temperature |
| `setProbeTemperature` | Set probe alarm temperature |
| `shutdownGrill` | Initiate grill shutdown sequence |
| `setSuperSmoke` | Enable/disable SuperSmoke (on/off) |
| `setKeepWarm` | Enable/disable KeepWarm mode (on/off) |
| `setTimerMinutes` | Start a cook timer |
| `cancelTimer` | Cancel the active timer |
| `requestStateUpdate` | Request immediate state refresh from grill |
| `connectMqtt` | Manually reconnect MQTT |
| `disconnectMqtt` | Disconnect MQTT |

---

## Automation Buttons (PushableButton)

Use these in Rule Machine with the trigger **"Button pushed"** on your grill device.

| Button | Event | Notes |
|--------|-------|-------|
| 1 | Preheat complete | Fires when grill transitions from preheating to cooking |
| 2 | Probe at target temperature | Fires when `probe_alarm_fired` goes true |
| 3 | Pellets low | Fires when pellet level drops below 20% |
| 4 | Grill offline or error | Fires when grill goes offline or an error appears |
| 5 | Cook timer complete | Fires when cook timer finishes |

### Example Rule Machine automations

- *"When Traeger button 2 pushed → Send notification: Your brisket has reached 203°F"*
- *"When Traeger button 3 pushed → Send push: Pellets are running low"*
- *"When Traeger button 1 pushed → Turn on patio lights"*
- *"When Traeger grillState changes to cool\_down → Send notification: Grill is cooling down"*

---

## Dashboard Tiles

Suggested tiles for a Hubitat dashboard:

- **Temperature** attribute → shows current grill temp
- **Heating Setpoint** attribute → shows target temp  
- **pelletLevel** attribute → shows pellet % (use a gauge template if available)
- **grillState** attribute → shows current state in plain English
- **probeTemperature** attribute → shows probe reading
- **switch** capability → on/off tile to show if grill is active

---

## Troubleshooting

**MQTT Status shows `error: 403 Forbidden`**
The pre-signed URL expired before connection. This shouldn't happen with the current code since a fresh URL is fetched on every connect attempt. If it persists, check that your Traeger credentials are still valid.

**MQTT Status shows `refused: 5`**
Authentication rejected by AWS IoT. Your Cognito tokens may have expired — go to the app settings page and re-save your credentials to force a fresh login.

**Attributes not updating**
Enable debug logging in the driver preferences and check the Hubitat logs. You should see `RAW status:` entries with every MQTT message. If those appear but attributes aren't updating, open an issue with the raw status payload.

**Multiple grills**
Each grill gets its own child device with its own MQTT connection. This should work fine but has only been tested with a single grill — feedback welcome.

---

## Technical Notes

- Auth uses standard Cognito `USER_PASSWORD_AUTH` flow (username/password, not SRP)
- MQTT connection uses a pre-signed WSS URL fetched from Traeger's `/mqtt-connections` endpoint
- A fresh pre-signed URL is fetched on every MQTT connect attempt (they expire quickly)
- The driver implements MQTT 3.1.1 framing manually: CONNECT, SUBSCRIBE, PUBLISH, PUBACK, PINGREQ
- Commands go via REST POST to Traeger's API at `1ywgyc65d1.execute-api.us-west-2.amazonaws.com/prod`
- MQTT topic for state: `prod/thing/update/{thingName}`

---

## Credits

- Traeger WiFire API reverse engineered from the official iOS app
- WebSocket MQTT approach adapted from my [Mysa thermostat driver](https://github.com/craigdewar/hubitat-mysa)
- Thanks to the Hubitat community for documentation on `interfaces.webSocket`

---

## License

MIT
