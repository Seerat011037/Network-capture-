# Network Capturing by Wireshark

Capture live network traffic on **Kali Linux**, filter it by protocol, identify at least three protocols, and save the result as a `.pcap` file.

| | |
|---|---|
| **Objective** | Capture live packets and identify basic protocols and traffic types |
| **Tools** | Wireshark 4.6.9, tshark, Kali Linux 2026.2 (VirtualBox VM) |
| **Capture interface** | `any` (includes `eth0`) |
| **Capture length** | About 1 minute |
| **Protocols identified** | DNS, ICMP, TCP, ARP, TLSv1.3 |
| **Deliverables** | `capture/network_capture.pcap` and this report |



---

## Step 1: Update Kali and install Wireshark

```bash
sudo apt update
sudo apt install wireshark tshark -y
```

When asked "Should non-superusers be able to capture packets?", choose **Yes**.

![Install Wireshark](screenshots/01-install-wireshark.jpg)

## Step 2: Allow your user to capture, then list interfaces

```bash
sudo usermod -aG wireshark $USER
tshark -D
```

`tshark -D` lists the interfaces Wireshark can see (`eth0`, `any`, `lo`, and others). Log out and back in if capturing without `sudo` fails.

![Add user and list interfaces](screenshots/02-add-user-and-list-interfaces.jpg)

## Step 3: Open Wireshark and start the capture

```bash
wireshark &
```

Select the interface with the moving graph (`any` or `eth0`), then use **Capture → Start** (`Ctrl+E`).

![Start capture](screenshots/03-start-capture.jpg)

## Step 4: Generate traffic

Open a second terminal while the capture is running:

```bash
ping -c 10 google.com
nslookup example.com
curl http://neverssl.com
curl -I https://www.wikipedia.org
```

| Command | Traffic it creates |
|---|---|
| `ping` | ICMP echo request and reply |
| `nslookup` | DNS query and response |
| `curl http://...` | TCP, plus HTTP when the server answers |
| `curl https://...` | TCP and TLS |

Result: `10 packets transmitted, 10 received, 0% packet loss` (average round trip about 58 ms).

![Generate traffic](screenshots/04-generate-traffic.jpg)

## Step 5: Stop after about one minute

Click the red **Stop** button, or press `Ctrl+E`.

![Live capture](screenshots/05-live-capture.jpg)

## Step 6: Filter by protocol

Type a filter in the display filter bar and press Enter.

```text
dns
icmp
tcp
arp
http
tls
```

### DNS

Filter: `dns`

- Source `10.0.2.15` (the Kali VM), destination `192.168.100.1` (the DNS server)
- UDP, destination port `53`
- Standard queries followed by standard query responses

![DNS filter](screenshots/06-filter-dns.jpg)

### ICMP

Filter: `icmp`

- Echo (ping) request from `10.0.2.15` to `142.250.202.238`
- Echo (ping) reply from `142.250.202.238` back to `10.0.2.15`
- 100 bytes per packet in the capture (84 bytes of ICMP and IP, plus the capture header)

![ICMP filter](screenshots/09-filter-icmp.jpg)

### TCP

Filter: `tcp`

- Connection attempts to port `80` (HTTP) on `34.223.124.45` (the `neverssl.com` request)
- Many `[TCP Retransmission]` packets: the SYN was sent repeatedly with no SYN-ACK back, including attempts over IPv6
- Port `443` traffic with `[FIN, ACK]` and `[RST]` flags where connections closed

![TCP filter](screenshots/08-filter-tcp.jpg)

### ARP

Filter: `arp`

- Request: "Who has 10.0.2.2? Tell 10.0.2.15"
- Reply: "10.0.2.2 is at 52:54:00:12:35:00" (the VirtualBox NAT gateway)

![ARP filter](screenshots/07-filter-arp.jpg)

### TLS

Filter: `tls`

- `TLSv1.3 Application Data` packets between `10.0.2.15` and `103.102.166.224` on port `443`. The data is encrypted, so only the metadata is visible.

(See the live capture screenshot in Step 5.)

## Step 7: Save the capture as `.pcap`

1. Clear the display filter so the whole capture is saved.
2. Go to **File → Save As** (`Ctrl+Shift+S`).
3. Name it `network_capture`.
4. In **Save as type**, pick **Wireshark/tcpdump/... - pcap**.

Wireshark's default format is `.pcapng`. If you saved a `.pcapng` file, convert it:

```bash
editcap -F pcap network_capture.pcapng network_capture.pcap
```

Verify it:

```bash
file network_capture.pcap
capinfos network_capture.pcap
```

`file` should report `tcpdump capture file`.

![Saved capture file](screenshots/10-saved-pcap-file.jpg)

## Step 8: Analyze from the command line (optional)

```bash
# Protocol hierarchy summary
tshark -r network_capture.pcap -q -z io,phs

# Show only DNS, ICMP, ARP packets
tshark -r network_capture.pcap -Y "dns"
tshark -r network_capture.pcap -Y "icmp"
tshark -r network_capture.pcap -Y "arp"

# TCP handshake attempts (SYN packets)
tshark -r network_capture.pcap -Y "tcp.flags.syn==1"

# Conversations between IP addresses
tshark -r network_capture.pcap -q -z conv,ip

# Packet count per protocol
tshark -r network_capture.pcap -T fields -e _ws.col.Protocol | sort | uniq -c | sort -rn
```

## Findings

| Protocol | Purpose | Example from this capture |
|---|---|---|
| DNS | Turns domain names into IP addresses | `10.0.2.15` queried `192.168.100.1` on UDP port 53 |
| ICMP | Network diagnostics (ping) | Echo request and reply with `142.250.202.238` |
| TCP | Reliable connections | SYN to `34.223.124.45:80`, retransmitted with no reply |
| ARP | Maps an IP address to a MAC address | `10.0.2.2` is at `52:54:00:12:35:00` |
| TLSv1.3 | Encrypted HTTPS traffic | Application Data on port 443 |

Observations:

- DNS lookups happened before the connections they served.
- Ping showed 0% packet loss, so the VM had working internet access.
- The repeated TCP SYN retransmissions to port 80 show a connection that was never completed, which is a useful pattern to recognise when troubleshooting.
- No HTTP packets appeared in the screenshots above, because the plain-HTTP connection did not complete. Re-run `curl http://neverssl.com` if you want to capture HTTP.

