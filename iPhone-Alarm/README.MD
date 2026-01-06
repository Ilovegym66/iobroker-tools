# iPhone Alarm -> MQTT -> ioBroker (Shortcuts + MQTTAnalyzer)

This guide documents a practical setup to trigger an **ioBroker state change after you stop an iPhone alarm** by sending an **MQTT message** from iOS (via **Shortcuts** and **MQTTAnalyzer**) and receiving it in ioBroker using the **mqtt adapter**.

> Goal: **When the alarm is stopped** → iPhone publishes an MQTT event → ioBroker reacts (set states, start automations, etc.).

---

## Architecture

1. **iPhone Clock / Alarm**
2. **Shortcuts Automation** (trigger: *Alarm is stopped*)
3. **MQTTAnalyzer** publishes an MQTT message
4. **MQTT Broker** (e.g., Mosquitto)
5. **ioBroker mqtt adapter** subscribes to the topic and creates/updates states
6. Optional: **ioBroker JavaScript** maps the incoming topic payload to your own `0_userdata.*` states

---

## Requirements

- iPhone with iOS that supports **Shortcuts Automations for Alarms**
- **MQTTAnalyzer** installed
- MQTT broker reachable by the iPhone (same Wi‑Fi or via VPN)
- ioBroker with:
  - **mqtt adapter** (client mode recommended)
  - optional: **JavaScript adapter** (for mapping/logic)

---

## Recommended Topic & Payload Design

Use a dedicated topic namespace and send **non-retained** event messages.

### Topic
```
iphone/alarm/event
```

### Payload (JSON)
```json
{
  "event": "alarm_stopped",
  "ts": "2026-01-06T07:12:03+01:00",
  "device": "Bernd-iPhone",
  "alarm": "any"
}
```

**Best practices**
- `retain`: **false** (avoid “stuck” events after reconnect/restart)
- `qos`: 0 or 1 (1 if you want better delivery reliability)

---

## Step 1 — MQTT Broker Setup (Mosquitto)

You can use any broker. If your iPhone is sometimes outside your home network, prefer **VPN** (WireGuard, etc.) rather than exposing MQTT to the public Internet.

**Security minimum**
- Create a dedicated MQTT user (e.g., `iphone_alarm`)
- Use ACLs so this user can only publish to `iphone/alarm/#`

Example ACL idea (conceptual):
```
user iphone_alarm
topic write iphone/alarm/#
```

---

## Step 2 — ioBroker mqtt Adapter Setup

1. Install the **mqtt** adapter in ioBroker
2. Configure it as **Client** (connect to your broker)
3. Subscribe to the topic namespace (or let it auto-create states on incoming topics)

After the first publish, ioBroker typically creates a state like:
```
mqtt.0.iphone.alarm.event
```
(The exact path depends on your mqtt adapter configuration.)

---

## Step 3 — iPhone Shortcuts Automation (Alarm Stopped)

Create a Shortcuts Automation:

1. Open **Shortcuts** → **Automation**
2. Create a **Personal Automation**
3. Trigger: **Alarm** → **Is Stopped** → **Any** (or choose a specific alarm)
4. Action: publish via **MQTTAnalyzer** (if available)
5. Disable **Ask Before Running** (if shown) so it can run automatically

### Important Note About MQTTAnalyzer Actions
Some users report that **MQTTAnalyzer actions do not appear** inside Shortcuts until the app has “donated” actions (i.e., you have configured a broker and done at least one publish in-app). See **Troubleshooting** below.

---

## Step 4 — Configure MQTTAnalyzer for Publishing

In MQTTAnalyzer:

1. Add your broker
2. Verify connection
3. Test **Publish** manually to your topic (e.g., `iphone/alarm/event`)
4. Confirm the message arrives (e.g., in ioBroker mqtt states or with another MQTT client)

---

## Step 5 — Map MQTT Event to Your Own ioBroker States (Optional but Recommended)

Instead of building automations directly on `mqtt.0.*`, map it to stable custom states under `0_userdata.*`.

### Example States
- `0_userdata.0.iPhoneAlarm.stopped` (boolean “event trigger”)
- `0_userdata.0.iPhoneAlarm.lastTs` (string timestamp)

### Example ioBroker JavaScript

> Adjust `IN_STATE` to match your mqtt adapter’s actual state path.

```js
'use strict';

const IN_STATE   = 'mqtt.0.iphone.alarm.event'; // change to your actual MQTT state
const OUT_STOP   = '0_userdata.0.iPhoneAlarm.stopped';
const OUT_LASTTS = '0_userdata.0.iPhoneAlarm.lastTs';

on({ id: IN_STATE, change: 'any' }, (obj) => {
  if (!obj?.state) return;

  const raw = String(obj.state.val ?? '');
  let msg = null;

  try { msg = JSON.parse(raw); } catch (e) {}

  const isStopped =
    (msg && msg.event === 'alarm_stopped') ||
    raw === '1' ||
    raw.toLowerCase() === 'true' ||
    raw.toLowerCase().includes('stopped');

  if (!isStopped) return;

  const ts = (msg && msg.ts) ? String(msg.ts) : new Date().toISOString();

  setState(OUT_LASTTS, ts, true);
  setState(OUT_STOP, true, true);

  // Reset after a short delay to keep it as an "event"
  setTimeout(() => setState(OUT_STOP, false, true), 1500);
});
```

Now you can trigger automations off:
- `0_userdata.0.iPhoneAlarm.stopped == true`

---

## Extending the Payload (Battery, Location, Steps, Floors)

Conceptually, you can extend the JSON payload with additional fields such as:
- `battery_pct`
- `location`: `{ "lat": ..., "lon": ... }`
- `steps_today`
- `floors_today`

However, these values are typically sourced via:
- **Battery**: Shortcuts action “Get Battery Level”
- **Location**: Shortcuts action “Get Current Location”
- **Health (Steps/Floors)**: Shortcuts Health actions (requires permissions)

### Practical caveat
In many iOS setups, **Health and Location** access can be more restrictive in automations (sometimes requiring permissions/unlock). If you need maximum reliability:
- Send only the **alarm event** via MQTT
- Send health/location telemetry on a separate schedule (e.g., every morning or hourly), or via a manual shortcut

---

## Troubleshooting

### MQTTAnalyzer actions do not show up in Shortcuts
Try the following:
1. Open MQTTAnalyzer, configure a broker, and do at least **one manual publish**
2. Restart the iPhone
3. Reinstall MQTTAnalyzer (last resort), then repeat step 1

Some apps only expose Shortcuts actions after initial usage.

### Message arrives but ioBroker state name is different
The created state path depends on mqtt adapter settings (prefix, “own instance”, JSON mode, etc.).  
Publish once, then look up the created object under `mqtt.0.*` and copy the exact state ID into the JS mapping script.

### Duplicate triggers after reconnect
Ensure publishes are **retain=false** and you treat the event as edge-triggered (use a short reset timer as shown).

### iPhone outside home Wi‑Fi
Use VPN to your home network so the iPhone can reach the broker securely.

---

## Optional Fallback (If MQTT from Shortcuts is not possible)

If MQTTAnalyzer cannot be used from Shortcuts on your device, a robust alternative is:
- Shortcuts → “Get Contents of URL” → call ioBroker **simple-api** endpoint
- ioBroker then publishes MQTT internally (or sets states directly)

This README focuses on the MQTTAnalyzer + mqtt adapter flow, but the fallback is often the fastest path if Shortcuts actions are missing.

---

## License

Choose a license appropriate for your repo (MIT is common for scripts/snippets).
