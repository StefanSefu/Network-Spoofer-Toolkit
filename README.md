# Network Spoofer Toolkit

## Project Overview

Network Spoofer Toolkit is a Python-based network interception toolkit for demonstrating Man-in-the-Middle (MITM) techniques in a controlled laboratory environment. The project now includes ARP poisoning, DNS spoofing, and SSL stripping, with a launcher script that can run supported attack combinations from one command.

The toolkit is built around:

- `scapy` for ARP and DNS packet inspection/modification.
- `netfilterqueue` for routing forwarded packets from the Linux kernel into Python.
- `iptables` for packet forwarding, trapping, and HTTP redirection rules.
- A custom transparent HTTP proxy for SSL stripping demonstrations.

---

## Legal & Ethical Disclaimer

**DO NOT RUN THIS TOOLKIT ON PUBLIC NETWORKS OR AGAINST SYSTEMS YOU DO NOT OWN OR HAVE EXPLICIT PERMISSION TO TEST.**

This project is for educational and laboratory use only. It is intended to help understand network-layer attacks, DNS manipulation, and HTTPS downgrade risks in a private, controlled environment such as virtual machines connected through a NAT or host-only test network. Unauthorized use may be illegal and harmful.

---

## Features

### 1. ARP Poisoning

`arp_poison.py` places the attacker machine between a victim and the gateway by continuously sending spoofed ARP replies.

It performs two spoofing actions:

- Tells the victim that the attacker is the gateway.
- Tells the gateway that the attacker is the victim.

The script enables IP forwarding so the victim can continue browsing while traffic passes through the attacker. When stopped, it attempts to restore the correct ARP mappings.

### 2. DNS Spoofing

`dns_spoof.py` intercepts forwarded DNS packets using `NetfilterQueue`. When a DNS response contains the configured target domain, the script replaces the DNS answer with the configured attacker IP address.

This can be used in the lab to redirect a victim from a chosen domain to a controlled host.

### 3. SSL Stripping

`ssl_strip.py` implements a transparent SSL stripping proxy for lab demonstrations. It redirects HTTP traffic to a local proxy running on port `8080`, forwards the victim's HTTP request to the real destination over HTTPS, receives the HTTPS response, rewrites HTTPS links back to HTTP, removes security-related response headers such as HSTS, and sends the modified HTTP response back to the victim.

The SSL stripping component includes support for:

- Transparent HTTP redirection with `iptables` NAT rules.
- A local proxy listener on `0.0.0.0:8080`.
- Upstream TLS connections to the real website.
- Header rewriting for requests and responses.
- Removal of headers that interfere with downgrade behavior, including `Strict-Transport-Security`.
- Response body rewriting from `https://` to `http://` for text-based and JSON responses.
- Gzip, deflate, and Brotli decompression/recompression.
- HTTP response parsing for `Content-Length`, `Transfer-Encoding: chunked`, and read-until-close responses.

### 4. Integrated Launcher

`launch.py` can start supported components together using a single command.

Supported modes:

- `arp`
- `arp+dns`
- `arp+ssl`

---

## Project Structure

```text
Network-Spoofer-Toolkit-main/
├── README.md
├── arp_poison.py       # ARP poisoning module
├── dns_spoof.py        # DNS spoofing module
├── ssl_strip.py        # SSL stripping transparent proxy
├── launch.py           # Integrated launcher for supported modes
├── rules.py            # IP forwarding and iptables helper functions
├── full_http.py        # HTTP request/response parsing helpers
└── body_response.py    # Body decoding, encoding, compression, and decompression helpers
```

---

## Prerequisites

### Operating System

Linux is required. Kali Linux is recommended.

This toolkit depends on Linux networking features and requires root privileges. It uses `/proc/sys/net/ipv4/ip_forward`, `iptables`, and Netfilter Queue. It does not run natively on Windows or macOS.

### System Dependencies

Install the required Linux packages:

```bash
sudo apt-get update
sudo apt-get install -y build-essential python3-dev libnetfilter-queue-dev iptables
```

### Python Dependencies

Install the required Python libraries:

```bash
pip3 install scapy netfilterqueue brotli
```

---

## Configuration

Most values can now be configured through command-line arguments instead of editing the Python files directly.

### ARP Poisoning Arguments

Used by `arp_poison.py` and by `launch.py` when the selected mode contains `arp`.

| Argument | Default | Description |
|---|---:|---|
| `--target-ip` | `192.168.2.12` | Victim IP address |
| `--gateway-ip` | `192.168.2.254` | Router/gateway IP address |
| `--interval` | `0.1` | Delay between ARP poisoning loops, in seconds |

### DNS Spoofing Arguments

Used by `dns_spoof.py` and by `launch.py` when the selected mode contains `dns`.

| Argument | Default | Description |
|---|---:|---|
| `--target-url` | `bing.com` | Domain substring to spoof |
| `--attacker-ip` | `192.168.2.13` | IP address returned in the forged DNS answer |
| `--queue-num` | `0` | Netfilter Queue number |

### SSL Stripping Configuration

`ssl_strip.py` currently uses the following fixed proxy settings:

| Setting | Value | Description |
|---|---:|---|
| Proxy listen host | `0.0.0.0` | Listen on all interfaces |
| Proxy listen port | `8080` | Local transparent proxy port |
| Redirected traffic | TCP port `80` | HTTP traffic redirected through iptables NAT |
| Upstream default port | `443` | Proxy connects to the real server over HTTPS |

---

## Usage Guide

Run all scripts with root privileges.

### Option A: Use the Integrated Launcher

#### ARP Poisoning Only

```bash
sudo python3 launch.py --mode arp \
  --target-ip 192.168.2.12 \
  --gateway-ip 192.168.2.254
```

#### ARP Poisoning + DNS Spoofing

```bash
sudo python3 launch.py --mode arp+dns \
  --target-ip 192.168.2.12 \
  --gateway-ip 192.168.2.254 \
  --target-url bing.com \
  --attacker-ip 192.168.2.13 \
  --queue-num 0
```

#### ARP Poisoning + SSL Stripping

```bash
sudo python3 launch.py --mode arp+ssl \
  --target-ip 192.168.2.12 \
  --gateway-ip 192.168.2.254
```

When `CTRL+C` is pressed, the launcher flushes iptables rules and restores ARP tables when ARP poisoning was enabled.

### Option B: Run Components Manually

#### Terminal 1: Start ARP Poisoning

```bash
sudo python3 arp_poison.py \
  --target-ip 192.168.2.12 \
  --gateway-ip 192.168.2.254 \
  --interval 0.1
```

#### Terminal 2: Start DNS Spoofing

```bash
sudo python3 dns_spoof.py \
  --target-url bing.com \
  --attacker-ip 192.168.2.13 \
  --queue-num 0
```

#### Or Terminal 2: Start SSL Stripping

```bash
sudo python3 ssl_strip.py
```

---

## Technical Architecture

### `arp_poison.py`

- Parses target, gateway, and interval values from command-line arguments.
- Enables IP forwarding through `rules.enable_ip_forwarding()`.
- Resolves MAC addresses using ARP requests.
- Sends spoofed ARP replies with Scapy.
- Restores ARP mappings on interruption.

### `dns_spoof.py`

- Enables IP forwarding.
- Inserts an iptables FORWARD-chain rule that sends packets to a Netfilter Queue.
- Reads packets from the queue with `NetfilterQueue`.
- Uses Scapy to inspect DNS response packets.
- Replaces matching DNS answers with the configured attacker IP.
- Deletes IP and UDP length/checksum fields so Scapy recalculates them.
- Flushes iptables rules during cleanup.

### `ssl_strip.py`

- Enables IP forwarding.
- Redirects HTTP traffic to local port `8080` using an iptables NAT PREROUTING rule.
- Starts a threaded TCP proxy on `0.0.0.0:8080`.
- Reads full HTTP requests from the victim.
- Removes hop-by-hop request headers such as `Connection`, `Transfer-Encoding`, and `Upgrade`.
- Converts request header references from `http://` to `https://` before forwarding upstream.
- Creates an upstream TLS connection to the requested host on port `443`.
- Reads the full HTTPS response from the upstream server.
- Removes response headers such as `Strict-Transport-Security`, `Transfer-Encoding`, `Content-Length`, and `Connection`.
- Rewrites response headers and eligible response bodies from `https://` to `http://`.
- Recalculates `Content-Length` before returning the response to the victim.
- Flushes iptables rules during cleanup.

### `full_http.py`

- Provides helper functions for reading complete HTTP requests and responses.
- Supports fixed-length bodies through `Content-Length`.
- Supports chunked responses.
- Supports responses that end when the upstream connection closes.
- Enforces a maximum HTTP size of 10 MB.

### `body_response.py`

- Handles response body decompression and recompression for gzip, deflate, and Brotli.
- Detects response charset from the `Content-Type` header.
- Decodes and re-encodes text bodies so URL rewriting can be applied safely.

### `rules.py`

- Enables Linux IP forwarding.
- Adds an iptables NAT rule to redirect HTTP traffic to the SSL stripping proxy.
- Adds an iptables FORWARD-chain rule for Netfilter Queue packet trapping.
- Flushes NAT and filter table rules during cleanup.

---

## Cleanup

The scripts attempt to clean up automatically when stopped with `CTRL+C`:

- ARP poisoning restores the victim and gateway ARP tables.
- DNS spoofing flushes iptables rules.
- SSL stripping flushes iptables NAT and filter rules.
- The integrated launcher flushes iptables rules and restores ARP tables when ARP mode is active.

If traffic does not return to normal, manually flush iptables rules:

```bash
sudo iptables -t nat -F
sudo iptables -F
```

You can also manually re-enable or disable IP forwarding as needed:

```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
# or
echo 0 | sudo tee /proc/sys/net/ipv4/ip_forward
```

---

## Limitations

- This project is intended for lab environments only.
- SSL stripping works only when the victim can be kept on HTTP or can be downgraded to HTTP. Sites that enforce HTTPS through browser preload lists, HSTS already cached by the browser, certificate pinning, or HTTPS-only modes may not be vulnerable.
- The SSL stripping proxy currently listens on a fixed port, `8080`.
- The launcher supports `arp`, `arp+dns`, and `arp+ssl`; it does not currently support running DNS spoofing and SSL stripping together in one launcher mode.
- The DNS spoofer matches the configured target URL as a substring of the queried name.
- Cleanup uses broad iptables flush commands, which is acceptable for isolated labs but may remove unrelated local firewall rules.

---

## Example Lab Flow

1. Place attacker, victim, and gateway in the same private lab network.
2. Identify the victim IP and gateway IP.
3. Start the toolkit with one of the launcher modes.
4. Generate browsing or DNS traffic from the victim.
5. Observe spoofed ARP behavior, DNS redirection, or SSL stripping proxy output in the attacker terminal.
6. Stop the tool with `CTRL+C` and verify that network settings are restored.

---

## Future Improvements

- Add command-line arguments for SSL stripping proxy host and port.
- Add a launcher mode that combines ARP poisoning, DNS spoofing, and SSL stripping when rule conflicts are handled safely.
- Add more precise iptables cleanup that removes only rules created by the toolkit.
- Add interface selection support.
- Add structured logging instead of raw terminal previews.
- Add a `requirements.txt` file for easier dependency installation.
