# Practice 13

Practical webpage: https://courses.cs.ut.ee/2026/cloud/spring/Main/Practice13

> Theme: IoT Hub on Azure — a Python device sends telemetry over AMQP, a consumer listens via Event Hub, data is routed to Blob Storage, then a self-provisioning device is built with the Device Provisioning Service (DPS).

> Reminder: Free tier IoT Hub allows **one endpoint** — after Exercise 13.5 you may lose the ability to redo 13.3. Take all deliverable screenshots for 13.3 **before** starting 13.5.

> Timing note: Exercise 13.4 sends one message every 10 s from the CSV. Do not leave it running — you will blow through the daily free quota. Archive the screenshots with their timestamps for evidence.

> Resource-name suffix: the instructions below use the suggested 4-character suffix **`m2q8`** for every globally-unique resource (IoT Hub, Storage Account, DPS). If Azure reports "name already taken", pick another 4–5 char lowercase alphanumeric suffix (e.g. `m2q9`, `h7v3`) and use the **same** suffix everywhere — the resource names and every script constant (`iot_hub_name`, `lilab13iot…`) must stay in sync.

---

## Exercise 13.1 — Create the IoT Hub + Register a Device

### Step-by-step

1. Open [https://portal.azure.com](https://portal.azure.com) and sign in with `qun.yan.li@ut.ee`. Confirm directory **Tartu Ülikool** and the UT student subscription is selected.

2. Create the resource group `lab13` first (re-usable for every resource below):
   - Portal top search → `Resource groups` → **+ Create** → Resource group: `lab13` · Region: `(Europe) Sweden Central` → **Review + create → Create**.

3. Create the IoT Hub:
   - Portal top search → `IoT Hub` → **+ Create**.
   - **Basics tab:** Subscription: UT student · Resource group: `lab13` · IoT hub name: `iothub-li-m2q8` (globally unique; must contain `li`) · Region: `Sweden Central`.
   - **Management tab:** Tier: **Free** · Daily message limit: `8000`.
   - **Networking / Add-ons / Tags:** leave defaults.
   - **Review + create → Create**. Deployment takes ~2–5 minutes.

4. Register the device:
   - Open the new IoT Hub → left blade **Device management → Devices → + Add device**.
   - Device ID: `device-li-01` · Authentication type: **Symmetric key** · Auto-generate keys: **ticked** · Connect this device to an IoT hub: **Enable** · **Save**.

5. Copy the primary key:
   - After save, click the `device-li-01` row in the Devices list.
   - Copy **Primary key** (click the copy icon next to the masked value). This is your `ACCESS_KEY` for Exercise 13.2 — store it somewhere you can retrieve later (e.g. a local text file that is **not** committed to the repo).

### Exercise Deliverables

- None — setup only.

### Checklist (fill in before proceeding)

- [ ] IoT Hub shows status `Active`.
- [ ] Device `device-li-01` exists, primary key copied safely.

---

## Exercise 13.2 — Python IoT Device (AMQP via `uamqp`)

### Step-by-step

1. SSH into the lab VM:
   ```powershell
   ssh -i "C:\Users\qunyan\Desktop\Qun Yan Li.pem" ubuntu@172.17.65.54
   ```
2. Install Python deps and prepare a working folder:
   ```bash
   sudo apt-get update && sudo apt-get install -y python3-pip python3-venv && mkdir -p ~/lab13 && cd ~/lab13 && python3 -m venv .venv && source .venv/bin/activate && pip install uamqp
   ```
3. Export secrets as environment variables (do **not** hardcode):
   ```bash
   export ACCESS_KEY='<PRIMARY_KEY_FROM_13_1>'
   ```
4. Create `pyIoTDevice.py`:
   ```python
   import uamqp, uuid, urllib.parse, json, os, time, hmac, hashlib, base64

   def generate_sas_token(uri, key, policy_name, expiry=3600):
       ttl = int(time.time()) + expiry
       sign_key = "%s\n%d" % (urllib.parse.quote_plus(uri), ttl)
       signature = base64.b64encode(hmac.HMAC(base64.b64decode(key), sign_key.encode('utf-8'), hashlib.sha256).digest())
       rawtoken = {'sr': uri, 'sig': signature, 'se': str(ttl)}
       if policy_name:
           rawtoken['skn'] = policy_name
       return 'SharedAccessSignature ' + urllib.parse.urlencode(rawtoken)

   iot_hub_name = 'iothub-li-m2q8'
   hostname = f'{iot_hub_name}.azure-devices.net'
   device_id = 'device-li-01'
   username = f'{device_id}@sas.{iot_hub_name}'
   access_key = os.getenv('ACCESS_KEY')

   sas_token = generate_sas_token(f'{hostname}/devices/{device_id}', access_key, None)
   operation = f'/devices/{device_id}/messages/events'
   uri = f'amqps://{urllib.parse.quote_plus(username)}:{urllib.parse.quote_plus(sas_token)}@{hostname}{operation}'

   send_client = uamqp.SendClient(uri, debug=True)
   msg_props = uamqp.message.MessageProperties()
   msg_props.message_id = str(uuid.uuid4())
   msg_data = json.dumps({'temperature': 22})
   send_client.send_message(uamqp.Message(msg_data, properties=msg_props))
   print('sent')
   ```
5. Run it, wait ~1 min, then check the IoT Hub **Overview → Device-to-cloud messages** counter:
   ```bash
   python pyIoTDevice.py
   ```
6. Take a screenshot of the IoT Hub Overview chart showing the incoming message spike with your hub name in the breadcrumbs.

### Exercise Deliverables

- `13_2_arrival.png` — Overview chart with hub name visible.
- `pyIoTDevice.py`.

### Checklist (fill in before proceeding)

- [ ] Script exits cleanly with no AMQP error.
- [ ] Overview counter ticked up.

---

## Exercise 13.3 — Event-Hub Consumer

### Step-by-step

1. In the IoT Hub blade → left blade **Hub settings → Built-in endpoints**:
   - **Event Hub-compatible name** field (top) → copy into `eventhub_name` (a short alphanumeric string).
   - **Consumer Groups** list → keep `$Default` or click **+ Add consumer group** to make a new one — value goes into `consumer_group`.
   - **Shared access policy** dropdown (just above the connection string) → change from `iothubowner` to **service**. The **Event Hub-compatible endpoint** field updates; click the copy icon → paste into `CONNECTION_STRING`. This is an `Endpoint=sb://...;SharedAccessKeyName=service;SharedAccessKey=...;EntityPath=...` string.
2. Install the SDK and set the env var:
   ```bash
   pip install azure-eventhub && export CONNECTION_STRING='<EVENT_HUB_COMPATIBLE_ENDPOINT_SERVICE>'
   ```
3. Create `consumeIoTData.py`:
   ```python
   import logging, asyncio, os
   from azure.eventhub.aio import EventHubConsumerClient

   consumer_group = '$Default'
   eventhub_name = '<EVENT_HUB_COMPATIBLE_NAME>'
   CONNECTION_STR = os.getenv('CONNECTION_STRING')

   logger = logging.getLogger('azure.eventhub')
   logging.basicConfig(level=logging.INFO)

   async def on_event(partition_context, event):
       print('Telemetry received:', event.body_as_str())
       print('Properties:', event.properties)
       print('System properties:', event.system_properties)
       await partition_context.update_checkpoint(event)

   async def receive():
       client = EventHubConsumerClient.from_connection_string(
           conn_str=CONNECTION_STR,
           consumer_group=consumer_group,
           eventhub_name=eventhub_name)
       async with client:
           await client.receive(on_event=on_event)

   if __name__ == '__main__':
       asyncio.get_event_loop().run_until_complete(receive())
   ```
4. In **two PowerShell windows** side-by-side (same VM, two SSH sessions):
   - Window A: `python consumeIoTData.py`
   - Window B: `python pyIoTDevice.py`
5. Screenshot both windows together so the sent message matches the received one. Device ID must be readable.

> If you hit `AttributeError: __aexit__`, comment out the `async with client:` line as the practical suggests.

### Exercise Deliverables

- `13_3_consumer.png` — side-by-side windows.
- `consumeIoTData.py`.

### Checklist (fill in before proceeding)

- [ ] Listener prints the exact JSON sent.
- [ ] Screenshot saved **before** starting 13.5.

---

## Exercise 13.4 — Send Real Sensor Data from CSV

### Step-by-step

1. Download `puhatu.csv` (linked from the practical page) into `~/lab13`.
2. Install pandas:
   ```bash
   pip install pandas
   ```
3. Modify `pyIoTDevice.py` — add imports at the top:
   ```python
   import time as t
   import pandas as pd
   ```
4. Replace the single-message send block with:
   ```python
   df = pd.read_csv('./puhatu.csv')
   messages = json.loads(df.to_json(orient='records'))
   try:
       for m in messages:
           msg_props.message_id = str(uuid.uuid4())
           msg = uamqp.Message(json.dumps(m), properties=msg_props)
           send_client.send_message(msg)
           print(msg)
           t.sleep(10)
   except KeyboardInterrupt:
       pass
   ```
5. Run for ~2–3 minutes, hit `Ctrl+C`, confirm the consumer printed several rows.

### Exercise Deliverables

- Updated `pyIoTDevice.py`.

### Checklist (fill in before proceeding)

- [ ] CSV rows appear in the consumer window.
- [ ] You **stopped** the sender before leaving — free tier quota preserved.

---

## Exercise 13.5 — Route IoT Data to Blob Storage

> **Warning:** creating this route may block the Event Hub listener (one endpoint max on Free tier). Make sure you have already archived the 13.3 screenshot.

### Step-by-step

1. Create the storage target (skip if already present from Practice 10):
   - Portal top search → `Storage accounts` → **+ Create** → Resource group: `lab13` · Name: `lilab13iotm2q8` (3–24 lowercase letters/digits) · Region: `Sweden Central` · Redundancy: **LRS** → **Review + create → Create**.
   - Open the storage account → **Data storage → Containers → + Container** → Name: `iot-data` · Anonymous access level: **Private** → **Create**.

2. Configure the IoT Hub route:
   - IoT Hub blade → left blade **Hub settings → Message routing → + Add**.
   - Name: `telemetry-to-storage`.
   - Endpoint: click **+ Add endpoint → Storage**.
     - Endpoint name: `blob-iot-data`.
     - **Pick a container** → Subscription: UT student · Storage account: `lilab13iotm2q8` · Container: `iot-data`.
     - Encoding: **JSON** · Authentication type: **Key-based**.
     - Leave batch frequency / chunk size / file name format at defaults.
     - **Create**.
   - Back on the Add route form: Data source: **Device Telemetry Messages** · Routing query: `true`.
   - **Save + skip enrichments** (or **Save** on older UI).
5. From the VM run the sender again for 60–90 seconds:
   ```bash
   python pyIoTDevice.py
   ```
6. After a few minutes, in the Storage Account → Containers → `iot-data`, open the nested folders to confirm JSON blobs appeared.

### Exercise Deliverables

- `13_5_blobs.png` — Azure Portal view of `iot-data` with the generated files visible.

### Checklist (fill in before proceeding)

- [ ] Blobs listed under the dated folder structure.
- [ ] Screenshot saved.

---

## Exercise 13.6 — Self-Provisioning Devices via DPS

### Step-by-step

1. Create the DPS resource:
   - Portal top search → `Device Provisioning Services` → **+ Create**.
   - Subscription: UT student · Resource group: `lab13` · Name: `dps-li-m2q8` · Region: `Sweden Central` · Pricing tier: **S1 (free units available)**.
   - **Review + create → Create**.

2. Link the existing IoT Hub:
   - Open the DPS resource → left blade **Settings → Linked IoT hubs → + Add**.
   - Subscription: UT student · IoT hub: `iothub-li-m2q8` · Access Policy: `iothubowner` → **Save**.

3. Create the enrollment group:
   - DPS blade → left blade **Settings → Manage enrollments → Enrollment groups tab → + Add enrollment group**.
   - Group name: `li-group` · Attestation type: **Symmetric Key** · Auto-generate keys: **Enable**.
   - IoT hubs: tick the linked hub. Leave Initial device twin state / allocation policy at defaults. **Save**.

4. Copy the group primary key:
   - Open `li-group` from the list → copy **Primary key**.
   - To get a **per-device** symmetric key, run the helper (Microsoft's "Derive a device key" tutorial). Quick command on the lab VM:
     ```bash
     printf '%s' '<DEVICE_ID>' | openssl sha256 -mac HMAC -macopt hexkey:$(printf '%s' '<GROUP_PRIMARY_KEY>' | base64 -d | xxd -p -c 256) -binary | base64
     ```
     Run this once per `DEVICE_ID` (`li-device-01`, `li-device-02`, …). The output is the `REQUEST_KEY` for that device.

5. Copy the DPS **ID scope**:
   - DPS blade → **Overview** → copy the **ID Scope** value (format: `0ne…`).
6. Install and set env vars:
   ```bash
   pip install azure-iot-device && export REQUEST_KEY='<DERIVED_DEVICE_KEY>' && export DEVICE_ID='li-device-01'
   ```
7. Create `selfRegisterIoTDevice.py`:
   ```python
   import asyncio, uuid, json, os
   from azure.iot.device.aio import ProvisioningDeviceClient, IoTHubDeviceClient
   from azure.iot.device import Message

   provisioning_host = 'global.azure-devices-provisioning.net'
   id_scope = '<ID_SCOPE>'
   request_key = os.getenv('REQUEST_KEY')
   device_id = os.getenv('DEVICE_ID')

   async def main():
       prov = ProvisioningDeviceClient.create_from_symmetric_key(
           provisioning_host=provisioning_host,
           registration_id=device_id,
           id_scope=id_scope,
           symmetric_key=request_key)
       result = await prov.register()
       if result.status == 'assigned':
           print('Registration succeeded:', device_id)
           device = IoTHubDeviceClient.create_from_symmetric_key(
               symmetric_key=request_key,
               hostname=result.registration_state.assigned_hub,
               device_id=result.registration_state.device_id)
           await device.connect()
           msg = Message(json.dumps({'temp': 132}))
           msg.message_id = str(uuid.uuid4())
           await device.send_message(msg)
           print('Message sent from', device_id)

   if __name__ == '__main__':
       asyncio.run(main())
   ```
8. For the individual task, open two SSH windows, give each a different `DEVICE_ID` / `REQUEST_KEY`, and run both at once:
   - Window A: `export DEVICE_ID='li-device-01' REQUEST_KEY='...' && python selfRegisterIoTDevice.py`
   - Window B: `export DEVICE_ID='li-device-02' REQUEST_KEY='...' && python selfRegisterIoTDevice.py`
9. Screenshot both windows so each device ID appears in the "Registration succeeded" / "Message sent" lines.

### Exercise Deliverables

- `13_6_multi_devices.png` — dual-window screenshot.
- `selfRegisterIoTDevice.py`.

### Checklist (fill in before proceeding)

- [ ] Two registrations succeed.
- [ ] IoT Hub → Devices lists both IDs.

---

## Tear-down

On the VM:
```bash
rm -rf ~/lab13/.venv
```
In Azure Portal: delete resource group `lab13`.

### Final Practical Checklist

**Screenshots archived:**

- [ ] `13_2_arrival.png` — IoT Hub Overview spike (Ex 13.2)
- [ ] `13_3_consumer.png` — device + consumer side-by-side (Ex 13.3)
- [ ] `13_5_blobs.png` — `iot-data` container with routed JSON (Ex 13.5)
- [ ] `13_6_multi_devices.png` — dual-window DPS registrations (Ex 13.6)

**Source committed:**

- [ ] `pyIoTDevice.py` (updated through 13.4).
- [ ] `consumeIoTData.py`.
- [ ] `selfRegisterIoTDevice.py`.
- [ ] No `.env`, no `__pycache__`, no hardcoded keys — every secret read from env vars.

**Cleanup & submission:**

- [ ] Sender script stopped (free-tier quota preserved).
- [ ] Resource group `lab13` deleted.
- [ ] All deliverables uploaded to the course submission system.
