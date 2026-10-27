# Network Traffic Analysis Lab

A beginner-friendly networking and cybersecurity practical covering Cisco Packet Tracer, Windows network commands, DNS lookups, and packet inspection with Wireshark.

> **Portfolio note:** This README documents the lab and observations from a personal practice session. Before publishing screenshots or PCAP files, review them for personal information, local network details, public IP addresses, hostnames, usernames, tokens, and other sensitive data. Do not upload an unrestricted capture of everyday browsing.

## Objectives

- Build a small LAN in Cisco Packet Tracer.
- Configure IPv4 addresses and a router interface.
- Test connectivity with `ping`.
- Observe routes with `tracert`.
- inspect local network settings with `ipconfig`.
- Resolve domain names with `nslookup`.
- Capture and identify DNS, HTTP, and HTTPS/TLS traffic in Wireshark.
- Inspect active connections with `netstat`.

## Tools

- Cisco Packet Tracer
- Wireshark on Windows
- Windows Command Prompt

## 1. Packet Tracer topology

Devices used:

- PC0
- PC1
- One 2960 switch
- One router
- One server

Connect the devices using Ethernet connections. In this practice topology, the two PCs and server connect to the switch, and the router connects to the same switch.

### IPv4 addressing plan

| Device | Interface | IPv4 address | Subnet mask | Default gateway |
|---|---|---:|---:|---:|
| Router0 | FastEthernet0/0 | `192.168.1.1` | `255.255.255.0` | Not applicable |
| PC0 | FastEthernet0 | `192.168.1.10` | `255.255.255.0` | `192.168.1.1` |
| PC1 | FastEthernet0 | `192.168.1.11` | `255.255.255.0` | `192.168.1.1` |
| Server0 | FastEthernet0 | `192.168.1.20` | `255.255.255.0` | `192.168.1.1` |

Use this table as the intended addressing plan; verify each device's settings in your saved Packet Tracer file. The DNS server field should point to the server only after DNS service and records have been configured on that server.

### Router configuration

The Packet Tracer router used `FastEthernet0/0` (not `GigabitEthernet0/0`).

```text
enable
configure terminal
interface FastEthernet0/0
ip address 192.168.1.1 255.255.255.0
no shutdown
end
show ip interface brief
```

The interface status was verified as `up` and the line protocol as `up`.

**CLI lesson:** `Router>` is user EXEC mode. Enter `enable` to reach `Router#`, then `configure terminal` to reach global configuration mode. Interface configuration commands must be entered from the appropriate configuration mode.

## 2. Connectivity testing

From a PC command prompt:

```bat
ping 192.168.1.1
ping 192.168.1.11
ping 192.168.1.20
```

Observed during the exercise: successful replies were received from the router (`192.168.1.1`), PC1 (`192.168.1.11`), and server (`192.168.1.20`), with 4 packets received and 0% loss in the shown tests.

A successful ping confirms IP reachability for that test. It does not, by itself, prove that every service or application is working.

## 3. Trace the path

```bat
tracert 192.168.1.20
```

Observed: the destination `192.168.1.20` appeared at hop 1 and the trace completed. This is expected when the destination is directly reachable on the same local subnet; traffic does not need to pass through the router as a routed hop.

## 4. View IP configuration

```bat
ipconfig
```

The PC output showed:

- IPv4 address: `192.168.1.10`
- Subnet mask: `255.255.255.0`
- Default gateway: `192.168.1.1`

The output also included IPv6 and other adapter details. `ipconfig /all` can show additional information such as the configured DNS servers and physical address.

## 5. DNS lookups

### Windows laptop

```bat
nslookup example.com
nslookup www.google.com
```

Observed: both names resolved to IPv4 and IPv6 addresses. The resolver was shown as an IPv6 address, while its display name was `Unknown`; that label alone does not mean DNS resolution failed.

### Packet Tracer server

```text
nslookup www.lab.com
```

Observed in Packet Tracer: `www.lab.com` resolved to `192.168.1.20` using the configured DNS service in the simulation.

### DNS in Wireshark

Useful display filters:

```text
dns
dns.qry.name == "www.google.com"
dns.flags.response == 1
```

DNS commonly uses UDP port 53; TCP port 53 is also used in some situations. In the captured traffic, Wireshark decoded DNS queries and responses, including a query for `docs.google.com` with query type HTTPS (type 65). A DNS HTTPS/SVCB record can provide service-connection information; it is not itself the encrypted HTTPS webpage.

## 6. HTTP packet analysis

Useful display filter:

```text
http
```

A captured request included these fields:

- **Method:** `GET`
- **Host:** `ctldl.windowsupdate.com`
- **Resource:** a path ending in `authrootstl.cab`
- **User-Agent:** `Microsoft-CryptoAPI/10.0`
- **Connection:** `Keep-Alive`
- **Cache validators:** `If-Modified-Since` and `If-None-Match`

The hostname and requested path are consistent with a Windows certificate-trust update resource. The packet alone cannot prove the full activity is legitimate, so treat this as a reasonable interpretation rather than a security verdict.

Another observed response was `HTTP/1.1 304 Not Modified`. This means the client can reuse a cached representation when the relevant cache validator indicates that it has not changed; the response does not normally include the resource body.

**Security observation:** Ordinary HTTP does not encrypt HTTP headers or content in transit. Avoid sending credentials or sensitive data over unencrypted HTTP.

## 7. HTTPS and TLS packet analysis

Useful display filters:

```text
tcp.port == 443
tls
tls.handshake
tls.handshake.type == 1
```

In the capture, Wireshark decoded TLS traffic on port 443 and displayed a TLS Client Hello with SNI `www.bing.com`. The capture also showed TLS application-data records.

- A **Client Hello** begins the TLS handshake and advertises supported protocol parameters.
- **SNI** can reveal the requested hostname in a typical unencrypted Client Hello; Encrypted Client Hello (ECH), where supported and used, changes this visibility.
- **TLS Application Data** is encrypted application traffic; Wireshark generally cannot show the plaintext webpage without appropriate decryption keys.
- The capture also showed QUIC over UDP port 443. QUIC is used by HTTP/3 and is not the same as TCP-based TLS traffic, even though both may use port 443.

Port 443 alone does not prove a packet is HTTPS. Use protocol decoding and packet details to support the interpretation.

## 8. Inspect active connections with netstat

```bat
netstat -ano
netstat -an
```

- `-a`: displays listening ports and active connections.
- `-n`: shows numerical addresses and ports.
- `-o`: includes the process ID (PID) for each connection.

The output showed listening TCP ports, established connections to remote port 443, local connections, and UDP endpoints. A `LISTENING` entry means an application is waiting for connections on that local endpoint; it is not, by itself, evidence of malicious activity. Use Task Manager or `tasklist /fi "PID eq <PID>"` to investigate a PID you do not recognise, and validate before taking action.

## 9. Troubleshooting notes

- **Cisco CLI says “Invalid input”:** check the prompt. Use `enable` first, then `configure terminal`, then enter interface configuration mode.
- **Router interface is down:** check the cable, selected interface, and `no shutdown`; verify with `show ip interface brief`.
- **`nslookup` times out:** verify the DNS server address, DNS service, and records. A successful ping to a host does not guarantee DNS is configured.
- **No HTTP packets appear:** modern websites normally use HTTPS. Generate only intentional test traffic, and remember that cached browser content may not create a new request.
- **Wireshark shows TLS Application Data:** encrypted payload is expected; packet capture does not automatically reveal HTTPS page contents.
- **Large captures are noisy:** use display filters to focus on the protocol you are studying.

## 10. What I learned

- How IPv4 addresses, subnet masks, gateways, and router interfaces work together in a simple LAN.
- How to test connectivity and interpret ping/traceroute output.
- How DNS queries and responses appear in packet captures.
- How to inspect HTTP methods, hosts, paths, headers, and cache-related responses.
- How TLS Client Hello, SNI, and encrypted application data appear in Wireshark.
- How `netstat` exposes listening endpoints and active connections, and why context is essential when triaging them.

## Suggested repository structure

```text
network-traffic-analysis-lab/
├── README.md
├── screenshots/
│   ├── packet-tracer-topology.png
│   ├── router-interface-status.png
│   ├── ping-and-tracert.png
│   ├── dns-query-response.png
│   ├── http-request.png
│   └── tls-client-hello.png
└── notes/
    └── commands.md
```


## Ethical and safety note

This is a learning lab using a local simulated network and traffic observed on my own device. Capture only traffic you are authorised to inspect. Do not publish credentials, session cookies, private messages, device identifiers, or unredacted network details.


