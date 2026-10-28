# IKEA LILJEBAGGE with Home Assistant over Matter and Thread

**Field guide — 28 October 2026.**

A field guide based on an ESP32-S3/ESP32-H2 OpenThread Border Router, Linux Mint, chip-tool and Home Assistant Matter Server.

The key tested correction was to pass the Thread Operational Dataset as **`hex:<dataset>`**, rather than as an unprefixed string. Commissioning then completed, and an encrypted operational read returned `VendorName: IKEA of Sweden`.

This is a documented commissioning success, not proof of uninterrupted operation. Home Assistant also logged successful commissioning, followed by discovery and subscription failures. The exact Multi-Admin handoff was not recovered from the available history.

## Evidence labels

- **Observed:** user-reported result or a result recorded in the retrieved logs.
- **Reconstructed:** an executable command template consistent with upstream documentation; not a recovered verbatim command.
- **Hypothesis:** a possible explanation that has not been isolated experimentally.

The history available for this draft is partial. Missing commands and versions are explicitly identified rather than invented.

## 1. Components and prerequisites

| Component | Recorded environment / requirement |
| --- | --- |
| Thermostat | IKEA LILJEBAGGE, Matter over Thread |
| Border router | ESP32-S3 host with ESP32-H2 IEEE 802.15.4 radio/RCP |
| Linux commissioner | Linux Mint 22.3, Ubuntu 24.04/Noble base, x86_64 |
| Bluetooth | Working Lenovo BLE adapter; thermostat discovered at approximately −46 dBm |
| chip-tool | Built executable at `out/linux-x64-chip-tool-clang/chip-tool`; exact SDK commit unavailable |
| Home Assistant | Separate Matter Server in Docker, service port 5580; exact HA/server versions unavailable |
| Device firmware | HA reported current 1.0.28 and an available 1.1.0 update; successful installation not established |
| ESP firmware | ESP-IDF 5.5.4 reported in conversation; an earlier H2/RCP log reports `v5.4.2-dirty` and a 2025-08-19 build. These cannot be treated as one verified firmware version |

The S3 and H2 have different roles: the S3 handles the border-router application/backhaul, and the H2 supplies the Thread radio. A Matter bridge is not a substitute for a Thread Border Router.

Before starting, have:

1. A configured border router with an active Thread network.
2. USB/serial console access to the border router, used in our setup to retrieve that network's complete Active Operational Dataset in hexadecimal.
3. The thermostat's setup PIN and discriminator, obtained privately from its setup information.
4. BLE available to the Mint host for initial commissioning.
5. IPv6 connectivity between the Mint host, Matter Server and Thread device through the border router, with local discovery/multicast working on the infrastructure network.
6. Persistent storage for both controllers' Matter fabric credentials.

For initial diagnosis, keep the controller hosts and border router on the same infrastructure LAN/VLAN. This does not mean that Thread and Ethernet/Wi-Fi are the same radio network. Internet IPv6 service is not the same requirement as working local IPv6.

### Border-router setup scope

The exact board wiring, ESP SDK commits, build configuration and flashing commands were not recovered. This guide starts with an operational border router and does not claim a reproduced firmware installation procedure. Use the [Espressif Thread Border Router SDK](https://github.com/espressif/esp-thread-br) for the board-specific setup. Do not interchange S3 application and H2 RCP firmware.

### Retrieve the Thread dataset through the console

**Observed and confirmed by the author:** we used USB/serial console access to the border router and entered this command at its ESP OpenThread console prompt:

```text
ot dataset active -x
```

This command runs on the border router, not in the Linux shell. It prints the existing Thread network's Active Operational Dataset as a long hexadecimal string. The exact serial device, terminal program and baud rate were not retained; use the console settings for your board and firmware.

Copy only the hexadecimal payload, excluding the prompt, any `Done` response and other console output. On the Linux host, paste that payload at the `THREAD_DATASET` prompt in section 3. The chip-tool argument is then `"hex:${THREAD_DATASET}"`: the variable contains only the hexadecimal payload, and `hex:` is added by the command.

The dataset includes network credentials. Keep it private; do not publish its output in screenshots, logs, issues or this repository.

### Should both ESP chips be updated?

The author reports rebuilding and flashing firmware for one ESP in the bridge. The available record does not establish which chip was flashed or whether the other chip was subsequently updated automatically.

- **ESP32-S3:** runs the border-router host application and Wi-Fi backhaul.
- **ESP32-H2:** runs the Thread radio co-processor (RCP) firmware.

**Upstream capability, not verified on our board:** Espressif supports updating the H2/RCP from the host. With `AUTO_UPDATE_RCP` enabled, an appropriate bundled RCP image and the required boot/reset wiring, the host can update the RCP automatically. See the [RCP update mechanism](https://docs.espressif.com/projects/esp-thread-br/en/latest/dev-guide/rcp_update.html) and [hardware requirements](https://docs.espressif.com/projects/esp-thread-br/en/latest/hardware_platforms.html).

Consequently, flashing the S3 can potentially also lead to an H2 update; this must not be assumed to have happened in our setup. Before manually flashing the other chip, inspect the build configuration, bundled RCP image and startup logs to establish which RCP firmware is running and whether an automatic update occurred. An older H2 version string alone does not establish incompatibility. Use the RCP firmware intended for the selected host build rather than independently selecting the newest image for each chip.

## 2. Prepare chip-tool on Linux Mint

**Observed executable checks:**

```bash
ls -lh out/linux-x64-chip-tool-clang/chip-tool
out/linux-x64-chip-tool-clang/chip-tool --help | head -25
```

The executable existed (approximately 141 MB). In that build, `--help` was interpreted as an unknown command set, and usage was printed. That message alone did not mean that the executable was broken.

**Reconstructed build outline, not the recovered installation transcript:**

```bash
git clone https://github.com/project-chip/connectedhomeip.git
cd connectedhomeip
git submodule update --init --recursive
source scripts/activate.sh
./scripts/build/build_examples.py --target linux-x64-chip-tool-clang build
git rev-parse HEAD
```

First install the dependencies and bootstrap requirements for the selected SDK revision using the upstream [Building Matter guide](https://project-chip.github.io/connectedhomeip-doc/guides/BUILDING.html). The historical dependency-install commands and commit are missing, so this outline is not a pinned, independently tested Mint installation recipe. Record the commit for any new test.

From the source directory:

```bash
CHIP_TOOL="$PWD/out/linux-x64-chip-tool-clang/chip-tool"
"${CHIP_TOOL}" pairing
"${CHIP_TOOL}" pairing ble-thread
"${CHIP_TOOL}" pairing open-commissioning-window
```

These usage probes are reconstructed diagnostics; incomplete arguments may return a nonzero status. Use the syntax printed by your binary if it differs.

**Reconstructed BLE diagnostics:**

```bash
systemctl status bluetooth --no-pager
rfkill list bluetooth
bluetoothctl list
bluetoothctl show
```

Controller index `0` was used in the commissioning command. Verify the appropriate index if the machine has multiple adapters. BLE discovery worked in our test; a generic Bluetooth failure does not explain every later commissioning error.

## 3. Commission over BLE into the Thread network

Use the manufacturer's documented procedure to put an uncommissioned thermostat into commissioning mode. The exact LILJEBAGGE button/reset sequence was not recovered; do not substitute a guessed sequence.

The shell examples below use Bash. Enter secrets interactively to avoid writing their literal values in shell history:

```bash
read -r -s -p 'Thread Active Operational Dataset (hex only): ' THREAD_DATASET
printf '\n'
read -r -s -p 'Thermostat setup PIN: ' PIN
printf '\n'
read -r -p 'Thermostat discriminator: ' DISCRIMINATOR
NODE_ID=10
BLE_CONTROLLER=0
```

Supply only the dataset's hexadecimal payload at the prompt, without a pre-existing `hex:` prefix, spaces or console decorations. These variables still become process arguments when chip-tool runs; use a trusted local host and do not record the session.

**Reconstructed normal attempt, with attestation verification enabled:**

```bash
"${CHIP_TOOL}" pairing ble-thread "$NODE_ID" \
  "hex:${THREAD_DATASET}" "$PIN" "$DISCRIMINATOR" \
  --ble-controller "$BLE_CONTROLLER"
```

**Recovered successful workaround, with device-specific values replaced by variables:**

```bash
"${CHIP_TOOL}" pairing ble-thread "$NODE_ID" \
  "hex:${THREAD_DATASET}" "$PIN" "$DISCRIMINATOR" \
  --ble-controller "$BLE_CONTROLLER" \
  --bypass-attestation-verifier true
```

The historical attempt used node 10 and BLE controller 0. A device-specific discriminator has deliberately been replaced. Node IDs are controller-local identifiers, not pairing codes.

The bypass disables device attestation verification. It was used in the experimental path after an attestation failure; it is not a general requirement for IKEA devices. Prefer resolving the PAA/trust-store issue and retaining verification. Do not present bypass commissioning as attestation success.

**Observed:**

- An attempt failed at attestation with `0x00000020: Failed Device Attestation`; the log reported a missing PAA certificate.
- Unprefixed dataset attempts reached `ThreadNetworkSetup`, then failed with `networkingStatus=1` and `0x000000AC: Internal error`.
- The original dataset represented 107 bytes. A filtered 94-byte variant also failed. Dataset trimming was not a verified fix.
- The corrected `hex:` command was followed by the user reporting `Device commissioning completed with success`.
- Another captured successful run used node `0xB` (decimal 11), with `CommissioningComplete` error code 0. Do not confuse that separate run with node 10.

After the command finishes:

```bash
unset THREAD_DATASET PIN
```

## 4. Verify operational communication

BLE/PASE progress alone is insufficient. The controller must reach the device over Thread/IP and establish an operational CASE session.

**Reconstructed read command** (the exact historical invocation was not recovered):

```bash
"${CHIP_TOOL}" basicinformation read vendor-name "$NODE_ID" 0
```

**Observed result:** a successful CASE/encrypted read returned:

```text
VendorName: IKEA of Sweden
```

This verifies an operational read on the chip-tool fabric. It does not prove Home Assistant control, successful OTA, or long-term availability.

Keep chip-tool's controller storage private and persistent. Reuse the same controller identity/storage for the following step. Switching Unix users, losing temporary storage or recreating the controller can leave it without administrative access to the existing fabric. Do not delete controller storage as routine troubleshooting.

## 5. Add Home Assistant through Multi-Admin

**Upstream-supported procedure; exact end-to-end handoff not recovered from our test record.**

Once chip-tool has commissioned the device, it is already on a Matter fabric. Have that existing administrator open a new commissioning window. Multi-Admin adds a second fabric; it does not require creating another Thread network.

**Reconstructed example:**

```bash
"${CHIP_TOOL}" pairing open-commissioning-window \
  "$NODE_ID" 1 300 1000 "$DISCRIMINATOR"
```

The arguments are node ID, enhanced-window option `1`, timeout `300` seconds, SPAKE2+ iteration count `1000`, and discriminator. The window discriminator is supplied by this command; using the earlier variable is an example choice, not a recovered historical value. Check your build's usage first.

Use the newly generated manual pairing code/QR payload privately in Home Assistant's flow for adding an already commissioned/shared Matter device. Complete it while the window is open. Use the new sharing code, not an assumption that the original printed code will reopen commissioning. Do not factory-reset between these steps, because a reset removes the existing commissioning state.

Refer to the [chip-tool Multi-Admin instructions](https://project-chip.github.io/connectedhomeip-doc/development_controllers/chip-tool/chip_tool_guide.html) and [Home Assistant Matter documentation](https://www.home-assistant.io/integrations/matter/).

The available Home Assistant logs show successful commissioning of its node 1, but do not establish that this specific command was the route used. A later user request referred to a LILJEBAGGE climate entity in Home Assistant. That supports entity creation; it does not establish reliable physical valve movement or setpoint synchronization.

## 6. Home Assistant Matter Server

**Observed deployment:** separate Docker Matter Server using service port 5580.

Configure Home Assistant's Matter integration to reach your server's WebSocket endpoint, typically `ws://<matter-server-host>:5580/ws`. Replace the placeholder locally. No personal host address is included here.

For a new container installation, follow the current [Matter Server Docker guide](https://github.com/matter-js/python-matter-server/blob/main/docs/docker.md) and [OS/network requirements](https://github.com/matter-js/python-matter-server/blob/main/docs/os_requirements.md). The historical Compose file/image tag is unavailable, so no template is labeled as our verified deployment.

Important checks for that deployment:

- Use the upstream host-network configuration, rather than assuming publishing TCP port 5580 also supplies Matter discovery and Thread connectivity.
- Persist the server's `/data` directory, which holds fabric/controller state.
- Provide Bluetooth/D-Bus access if the server itself will perform local BLE commissioning; Mint-side BLE commissioning and on-network Multi-Admin are distinct stages.
- Confirm host IPv6, routes and multicast discovery. A working HA-to-server WebSocket only checks that connection.

Home Assistant recommends HA OS with its Matter app; self-managed Docker requires handling the host network requirements yourself.

## 7. Troubleshooting: observations versus explanations

| Symptom | What we observed | Interpretation / next check |
| --- | --- | --- |
| `Unknown cluster or command set: --help` | Binary printed usage | Use command/subcommand usage; not evidence of a failed build |
| Missing PAA / `0x20` attestation failure | Commissioning stopped during attestation | Check the relevant build's PAA trust store. Bypass was an experimental workaround, not a verified trust-store repair |
| `ThreadNetworkSetup`, status 1, `0xAC` | Unprefixed dataset attempts failed | Adding `hex:` was the tested correction; do not generalize every internal error to this cause |
| Timeout `0x32` during cleanup | Followed an earlier failure | Preserve the first failure; cleanup timeout alone does not identify the cause |
| BLE connected but commissioning failed later | BLE/PASE had progressed | Check attestation, dataset encoding and operational IP reachability by stage |
| Retransmissions, mDNS/operational-discovery timeouts | Seen after HA commissioning succeeded | IPv6 routing, multicast filtering, border-router state and device availability are candidates; no single root cause was proven |
| Subscription liveness timeout | Seen in HA logs | Not proof that pairing failed; investigate operational reachability and device wake/availability behavior |
| Missing `/data/chip.json`, reset configuration on startup | Seen in server startup log | Can be first startup or lost/unmounted storage. Unexpected recurrence warrants checking mounts; the message alone does not prove data loss |
| OTA version 1.1.0 offered | Current version reported as 1.0.28 | Offer/discovery is not evidence of successful download or installation |

**Reconstructed, read-only Linux network checks** (use on the affected controller host):

```bash
ip -6 address show
ip -6 route show
ip -6 neigh show
docker logs --tail 200 matter-server
```

Adapt the container name. Review outputs privately: addresses and credentials can occur in logs. Do not publish raw logs. Do not disable firewalls, regenerate Thread credentials, erase controller state, or reset thermostats merely because a timeout occurred.

## 8. Verified outcome and remaining gaps

| Claim | Evidence status |
| --- | --- |
| Thread dataset retrieved with `ot dataset active -x` through USB/serial console | Confirmed by the author |
| Linux BLE discovery worked | Observed |
| chip-tool executable ran on Mint | Observed |
| Corrected BLE/Thread commissioning completed | User-reported success; successful commissioning log also available for a separate node |
| Operational encrypted read returned IKEA vendor name | Observed |
| HA Matter Server commissioned node 1 | Observed in logs |
| HA had a LILJEBAGGE climate entity | Supported by the user's later entity reference |
| Exact chip-tool-to-HA Multi-Admin sequence | Not recovered |
| Both thermostats were reliably controllable in HA | Not established by recovered user evidence; an earlier assistant claim is insufficient |
| Setpoint synchronization / physical valve response / stable overnight operation | Not verified |
| Successful firmware update to 1.1.0 | Not verified |

Before calling this a fully reproducible end-to-end recipe, record the missing SDK commit, firmware/build versions, board configuration and flashing steps, exact Multi-Admin handoff, and repeated HA read/setpoint results. These gaps should remain visible if the guide is published as a field report.

## 9. Privacy and contributions

Never commit Thread datasets/network keys, Matter fabric/private keys, controller storage, QR payloads, manual pairing codes, personal network addresses or unreviewed log captures. All examples intentionally use variables/placeholders.

When reporting a problem, provide software versions, the failing commissioning stage, sanitized error lines and whether an operational read succeeds. Separate executed steps from proposed explanations.

## Sources and provenance

Field observations were recovered from the author's 27–28 October 2026 conversation and pasted logs. The original private transcripts are not distributed. Reconstruction has not been retested on the hardware during preparation of this draft.

Upstream references checked on 28 October 2026:

- [CHIP Tool guide: BLE/Thread and Multi-Admin](https://project-chip.github.io/connectedhomeip-doc/development_controllers/chip-tool/chip_tool_guide.html)
- [Matter build prerequisites](https://project-chip.github.io/connectedhomeip-doc/guides/BUILDING.html)
- [Unified build script](https://github.com/project-chip/connectedhomeip/blob/master/scripts/build/README.md)
- [Espressif Thread Border Router SDK](https://github.com/espressif/esp-thread-br)
- [Home Assistant Matter](https://www.home-assistant.io/integrations/matter/)
- [Home Assistant Thread](https://www.home-assistant.io/integrations/thread/)
- [Matter Server Docker](https://github.com/matter-js/python-matter-server/blob/main/docs/docker.md)
- [Matter Server OS requirements](https://github.com/matter-js/python-matter-server/blob/main/docs/os_requirements.md)

This is an independent community guide, not an IKEA, Espressif or Home Assistant publication.
