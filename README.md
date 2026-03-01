# RTL-Monitor

This project turns an **Ameba**-based board (e.g., BW20-12F, RTL8711) into a dual‑band Wi-Fi monitor. It captures 802.11 frames in promiscuous mode, adds radiotap headers, and streams them as a **pcapng** file over a high‑speed UART. The output can be read directly by Wireshark.

## Features

- **Dual‑band channel hopping** – scans both 2.4 GHz (channels 1–13) and 5 GHz (channels 36–165).
- **pcapng output** – emits a valid Section Header Block (SHB) and Interface Description Block (IDB) with linktype `127` (radiotap), followed by Enhanced Packet Blocks (EPB) containing radiotap + 802.11 frames.
- **Radiotap headers** – include channel, frequency, RSSI, and data rate.
- **High‑resolution timestamps** – nanosecond resolution using the Cortex‑M DWT cycle counter (optional).
- **ISR‑safe packet handling** – lock‑free ring buffer and packet pool for zero‑copy from the promiscuous callback.
- **Configurable** – channel list, dwell time, UART baud rate, pinout, buffer sizes, etc. (see `monitor.h`).

## Hardware Requirements

- An Ameba‑based board with Wi‑Fi (e.g., BW20-12F, AmebaPro2, RTL8735, RTL8721).
- A USB‑to‑UART adapter (3.3 V logic) connected to the board’s UART pins.
- A computer to receive and analyse the pcapng stream (e.g., using Wireshark).

## Pin Configuration

By default the UART uses:

| Signal | Pin   |
|--------|-------|
| TX     | `_PB_5` |
| RX     | `_PB_4` |

These can be changed in `monitor.h` by redefining `TX_PIN` and `RX_PIN`.

## Building and Flashing

1. **Set up the Ameba SDK** – you need the official SDK (e.g., `ameba-rtos`) with FreeRTOS and the Wi-Fi driver.
2. **Configure** – edit `monitor.h` to suit your needs (baud rate, channel list, timestamps, etc.).
3. **Build** – use the usual Ameba build process (e.g., `./build.py`).
4. **Flash** – upload the generated firmware to your board.

## Usage

1. Connect the UART adapter to your PC. Note the serial port name (e.g., `/dev/ttyUSB0` on Linux, `COM3` on Windows).
2. Start a serial capture tool that can save raw binary data, for example:
   ```bash
   cat /dev/ttyUSB0 > capture.pcapng
   ```
   or using `screen` with logging:
   ```bash
   screen -L -Logfile capture.pcapng /dev/ttyUSB0 2000000
   ```
   (adjust baud rate to match `BAUD_RATE`).
3. Power on or reset the Ameba board. After a short delay (3 seconds in `app_example.c`), the monitor starts and immediately sends the pcapng headers.
4. Stop the capture after some time, then open `capture.pcapng` in Wireshark. Wireshark will automatically interpret the radiotap headers and 802.11 frames.

## Configuration Options (`monitor.h`)

| Macro                | Description                                                                 | Default                                     |
|----------------------|-----------------------------------------------------------------------------|---------------------------------------------|
| `WLAN_IDX`           | Wi‑Fi interface index (0 = STA)                                             | `STA_WLAN_INDEX` (0)                        |
| `CHANNEL_LIST`       | Array of channels to hop                                                    | 2.4 GHz 1–13, 5 GHz 36–165                  |
| `HOP_INTERVAL_MS`    | Dwell time per channel (ms)                                                 | `100`                                       |
| `MONITOR_TASK_STACK` | Stack size for the monitor task (bytes)                                     | `4096`                                      |
| `HOP_TASK_STACK`     | Stack size for the channel hopper task (bytes)                              | `1024`                                      |
| `RING_SIZE`          | Number of packet descriptors in the ring buffer                             | `1024`                                      |
| `PACKET_POOL_SIZE`   | Number of fixed‑size buffers for frame data                                 | `64`                                        |
| `PACKET_BUFFER_SIZE` | Maximum frame length that can be stored (bytes)                             | `2346` (max 802.11 frame)                   |
| `TX_PIN`, `RX_PIN`   | UART pins                                                                   | `_PB_5`, `_PB_4`                            |
| `BAUD_RATE`          | UART baud rate                                                              | `2000000`                                   |
| `IDB_FCS_LEN`        | FCS length in Interface Description Block (`0` = no FCS, `4` = FCS present) | `4`                                         |
| `IDB_TSRESOL`        | Timestamp resolution (`9` = nanoseconds)                                    | `9`                                         |
| `USE_DWT_CYCCNT`     | Enable nanosecond timestamps using DWT cycle counter                        | defined (comment out to use fallback)       |

## Output Format

The UART stream is a concatenation of pcapng blocks:

1. **Section Header Block (SHB)** – identifies the file as pcapng.
2. **Interface Description Block (IDB)** – declares one interface with linktype `127` (radiotap) and options like timestamp resolution and FCS length.
3. **Enhanced Packet Blocks (EPB)** – one per captured frame, containing:
   - Radiotap header (channel, frequency, RSSI, data rate).
   - The raw 802.11 frame (possibly truncated to `PACKET_BUFFER_SIZE`).

All blocks are in little‑endian order and are written without any extra framing – the stream is a raw pcapng file.

## Troubleshooting

- **No data in Wireshark** – verify UART wiring and baud rate. Check that the monitor task actually started (the example `app_example.c` delays 3 s then calls `monitor_start()`).
- **Garbage output** – wrong baud rate or parity settings. Ensure `BAUD_RATE` matches your terminal/sniffer.
- **Missing frames** – increase `RING_SIZE` or `PACKET_POOL_SIZE` if the CPU cannot keep up with the UART.
- **Timestamp resolution** – if `USE_DWT_CYCCNT` is defined but DWT is not available (older Cortex‑M), timestamps will fall back to FreeRTOS tick count (millisecond resolution). Remove the define to use the fallback explicitly.
