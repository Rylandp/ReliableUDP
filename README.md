# ReliableUDP

A Java implementation of reliable file transfer over UDP using a stop-and-wait protocol with sequence numbers, explicit acknowledgments, retransmissions, out-of-order buffering, duplicate detection, and end-to-end MD5 checksum verification. Includes an interactive/single-file sender with configurable network impairment simulation and a multi-threaded concurrent test harness.

## Architecture & Reliability Features

- **`receiver.java`**: Listens on a UDP port (`DatagramSocket`), tracks concurrent inbound transfers in a `ConcurrentHashMap<String, Transfer>`, buffers chunks by sequence number (`0` to `totalChunks - 1`), ignores and re-ACKs duplicates, reassembles completed transfers in order, verifies the MD5 hex digest, and runs a background daemon thread every 5 seconds to purge transfers idle for more than `30,000 ms`.
- **`sender.java`**: Splits an input file into `1000`-byte chunks, computes its MD5 hash, Base64-encodes each payload, and transmits packets using stop-and-wait (`2000 ms` `SO_TIMEOUT`, up to `20` retries per packet). Supports configurable packet loss, packet reordering, and duplicate packet injection.
- **`tsender.java`**: Concurrent test harness using a fixed thread pool (`ExecutorService`) to transfer multiple files simultaneously and report `PASS`/`FAIL` per transfer. Supports passing `auto` to generate and clean up synthetic small (`500 B`), medium (`4.5 KB`), and large (`19.5 KB`) test files automatically.

## Requirements

- **Java 11+** (supports running single-file source programs directly via `java <file>.java`, or compiling with `javac`).

## Running the Project

### 1. Start the Receiver

```bash
java receiver.java [-v] <port>
```

Example (port `9000` with verbose stderr logging):

```bash
java receiver.java -v 9000
```

### 2. Run the Single-File Sender

```bash
java sender.java [-v] <host> <port> <file> <loss%> <reorder%> <dup%>
```

Example (send `testfile.txt` to `localhost:9000` with 20% packet loss, 30% reorder probability, and 10% duplicate probability):

```bash
java sender.java -v localhost 9000 testfile.txt 20 30 10
```

### 3. Run the Concurrent Test Harness

```bash
java tsender.java <host> <port> <loss%> <reorder%> <dup%> <file1> [file2 ...]
```

Example (run auto-generated small/medium/large files plus `testfile.txt` concurrently under 15% loss, 25% reorder, and 15% duplication):

```bash
java tsender.java localhost 9000 15 25 15 auto testfile.txt
```

## Wire Protocol

All control and data messages are UTF-8 strings sent as individual UDP datagrams:

| Direction | Format | Description |
| :--- | :--- | :--- |
| Sender $\rightarrow$ Receiver | `BEGIN <transfer_id> <filename> <total_chunks> <checksum>` | Initiates a transfer with expected chunk count and MD5 hex checksum |
| Receiver $\rightarrow$ Sender | `ACK <transfer_id> BEGIN` | Acknowledges transfer initialization (or `ERR duplicate transfer` / `ERR malformed`) |
| Sender $\rightarrow$ Receiver | `DATA <transfer_id> <seqnum> <base64_payload>` | Sends a `1000`-byte (or final smaller) Base64-encoded chunk |
| Receiver $\rightarrow$ Sender | `ACK <transfer_id> <seqnum>` | Acknowledges receipt of chunk `<seqnum>` (also re-sent on duplicates or late packets for completed transfers) |
| Sender $\rightarrow$ Receiver | `END <transfer_id>` | Signals end of transmission |
| Receiver $\rightarrow$ Sender | `ACK <transfer_id> END` | Acknowledges `END` and triggers final reassembly/MD5 check if all chunks have arrived |

### Receiver Log Output

- `RECV <transfer_id> BEGIN chunks=<n>` — New transfer registered
- `RECV <transfer_id> DATA seq=<seqnum>` — New in-range chunk stored
- `DUP <transfer_id> seq=<seqnum>` — Duplicate chunk received and re-ACKed
- `COMPLETE <transfer_id> file=<filename> status=<OK|FAIL>` — All chunks received, reassembled, and verified against expected MD5
- `TIMEOUT <transfer_id>` — Stale transfer purged after 30 seconds of inactivity

## Network Impairment Simulation (`sender.java` & `tsender.java`)

- **Loss (`<loss%>`)**: Each transmission attempt has a `<loss%>` probability of being skipped without calling `socket.send()`, forcing a `2000 ms` timeout and retransmission.
- **Reorder (`<reorder%>`)**: Outgoing `DATA` packets have a `<reorder%>` probability of being held in a buffer up to capacity $\max(2, \min(6, \text{reorder\%} / 10))$, then flushed in shuffled order when full or when a non-buffered packet arrives.
- **Duplication (`<dup%>`)**: Transmitted packets have a `<dup%>` probability of being sent twice back-to-back on the socket.

## Project Structure

```text
.
├── receiver.java   # Multithreaded UDP receiver, state tracker, MD5 verifier, and timeout cleaner
├── sender.java     # Stop-and-wait UDP file sender with loss/reorder/dup simulation
├── tsender.java    # Concurrent multi-file test runner with 'auto' synthetic file generator
└── testfile.txt    # Sample text payload for baseline transfer verification
```
```
