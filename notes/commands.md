# Command Cheat Sheet

## Windows
```bat
ping 192.168.1.1
tracert 192.168.1.20
ipconfig
ipconfig /all
nslookup example.com
netstat -ano
netstat -an
```

## Cisco Packet Tracer router
```text
enable
configure terminal
interface FastEthernet0/0
ip address 192.168.1.1 255.255.255.0
no shutdown
end
show ip interface brief
```

## Wireshark display filters
```text
dns
dns.qry.name == "www.google.com"
dns.flags.response == 1
http
tcp.port == 443
tls
tls.handshake
tls.handshake.type == 1
```

Only capture traffic you are authorised to inspect. Review screenshots and captures for sensitive information before publishing.
