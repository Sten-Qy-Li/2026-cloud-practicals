# Practice 13

Practical webpage: https://courses.cs.ut.ee/2026/cloud/spring/Main/Practice13

> Theme: IoT Hub on Azure — a Python device sends telemetry over AMQP, a consumer listens via Event Hub, data is routed to Blob Storage, then a self-provisioning device is built with the Device Provisioning Service (DPS).

> Reminder: Free tier IoT Hub allows **one endpoint** — after Exercise 13.5 you may lose the ability to redo 13.3. Take all deliverable screenshots for 13.3 **before** starting 13.5.

> Timing note: Exercise 13.4 sends one message every 10 s from the CSV. Do not leave it running — you will blow through the daily free quota. Archive the screenshots with their timestamps for evidence.

---

## Exercise 13.1 — Create the IoT Hub + Register a Device

### Step-by-step

1. In the Azure Portal search for **IoT Hub** → Create.
2. Resource group: `lab13`. Region: `Sweden Central`. Tier: **Free F1**.
3. Name: `iothub-li-<random>` (must include your surname and be globally unique).
4. After creation, in the IoT Hub blade → **Devices → Add device** → ID `device-li-01`. Auth type: **Symmetric key**. Auto-generate keys. Save.
5. Open the device → copy the **Primary key** (this becomes `ACCESS_KEY`).

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

   iot_hub_name = 'iothub-li-<random>'
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

1. From the IoT Hub → **Hub settings → Built-in endpoints**, copy:
   - **Event Hub-compatible name** → `eventhub_name`
   - A consumer group from the list (`$Default` is fine) → `consumer_group`
   - The **Event Hub-compatible endpoint**, then switch the shared-access policy from `iothubowner` to **service** and copy that connection string → `CONNECTION_STRING`
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

1. In the `lab13` resource group, create a **Storage Account** if you don't have one (Standard LRS is fine). Add a **container** called `iot-data`.
2. IoT Hub blade → **Hub settings → Message routing → Add** → name `telemetry-to-storage`.
3. Endpoint type: **Storage** → add endpoint:
   - Name: `blob-iot-data`
   - Pick the storage account and `iot-data` container
   - **Encoding: JSON**
   - Authentication: **Key-based**
4. Data source: **Device Telemetry Messages**. Routing query: `true`. Click **Create + skip enrichments**.
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

1. Create **Azure IoT Hub Device Provisioning Service** in `lab13` → same region.
2. DPS blade → **Linked IoT Hubs → Add/Link** → select your IoT Hub.
3. DPS blade → **Manage enrollments → Enrollment groups → Add**:
   - Name: `li-group`
   - Attestation: **Symmetric Key** (auto-generate)
   - Linked hub: select it
4. Open the enrollment → copy the **Primary key**. Derive per-device keys following the "Derive a device key" Microsoft tutorial.
5. Copy the DPS **ID scope** from the DPS overview.
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
