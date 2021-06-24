# IkaGo

**IkaGo** is a proxy which helps bypassing UDP blocking, UDP QoS and NAT firewall written in Go.

*IkaGo is designed to accelerate games in game consoles.*

*If you have a SOCKS5 proxy, use [pcap2socks](https://github.com/zhxie/pcap2socks) for better performance.*

<p align="center">
  <img src="/assets/squid.jpg" alt="an Inkling going through a grate">
</p>
<p align="center">
  Pass the firewall like a squid : )
</p>

## Features

<p align="center">
  <img src="/assets/diagram.jpg" alt="diagram">
</p>

- **FakeTCP**: All TCP, UDP and ICMPv4 packets will be sent with a TCP header to bypass UDP blocking and UDP QoS. Inspired by [Udp2raw-tunnel](https://github.com/wangyu-/udp2raw-tunnel). The handshaking of TCP is also simulated.
- **Proxy ARP**: Reply ARP request as it owns the specified address which is not on the network.
- **Multiplexing and Multiple**: One client can handle multiple connections from different devices. And one server can serve multiple clients.
- **Cross Platform**: Works well with Windows, macOS, Linux and others in theory.
- **Monitor**: Observe traffic on [IkaGo-web](https://zhxie.github.io/ikago-web)
- **Full Cone NAT**
- **Encryption**
- **KCP Support**

## Dependencies

1. [Npcap](http://www.npcap.org/) or WinPcap in Windows, libpcap in macOS, Linux and others.

2. (Optional, recommended) pf in macOS, iptables and ethtool in Linux for automatic firewall rule addition.

## Usage

```
# Client
go run ./cmd/ikago-client -r [sources] -s [ip:port]

# Server
go run ./cmd/ikago-server -p [port]
```

Examples of configuration file are [here](/configs).

### Common options

`-list-devices`: (Optional, exclusive) List all valid devices in current computer.

`-c path`: (Optional, exclusive) Configuration file. Examples of configuration file are [here](/configs). If IkaGo does not receive any arguments except `-v`, it will automatically read the configuration file `config.json` in the working directory if it exists.

`-listen-devices devices`: (Optional) Devices for listening, use comma to separate multiple devices. If this value is not set, all valid devices excluding loopback devices will be used. For example, `-listen-devices eth0,wifi0,lo`.

`-upstream-device device`: (Optional) Device for routing upstream to. If this value is not set, the first valid device with the same domain of gateway will be used.

`-gateway address`: (Optional) Gateway address. If this value is not set, the first gateway address in the routing table will be used.

`-mode mode`: (Optional) Mode, can be `faketcp`, `tcp`. Default as `tcp`. This option needs to be set consistently between the client and the server. You may have to configure your firewall by using `-rule` or follow the [troubleshoot](https://github.com/zhxie/ikago#troubleshoot) below in some modes.

`-method method`: (Optional) Method of encryption, can be `plain`, `aes-128-gcm`, `aes-192-gcm`, `aes-256-gcm`, `chacha20-poly1305` or `xchacha20-poly1305`. Default as `plain`. This option needs to be set consistently between the client and the server. For more about encryption, please refer to the [development documentation](/dev.md).

`-password password`: (Optional) Password of encryption, must be set only when method is not `plain`. This option needs to be set consistently between the client and the server.

`-rule`: (Optional, recommended) Add firewall rule. In some OS, firewall rules need to be added to ensure the operation of IkaGo. Rules are described in [troubleshoot](https://github.com/zhxie/ikago#troubleshoot) below.

`-monitor port`: (Optional) Port for monitoring. If this value is set, IkaGo will host HTTP server on `localhost:port` and print JSON statistics on it. You can observe observe traffic on [IkaGo-web](http://ikago.ikas.ink).

`-v`: (Optional) Print verbose messages. Either `-v` or `verbose` in configuration file is set `true`, IkaGo will print verbose messages.

`-log path`: (Optional) Log.

#### FakeTCP options

`-mtu size`: (Optional) MTU. MTU is set in traffic between the client and the server.

`-kcp`: (Optional) Enable KCP. This option needs to be set consistently between the client and the server.

`-kcp-mtu size`, `-kcp-sndwnd size`, `-kcp-rcvwnd size`, `-kcp-datashard size`, `-kcp-parityshard size`, `-kcp-acknodelay`: (Optional) KCP tuning options. These options need to be set consistently between the client and the server. Please refer to the [kcp-go](https://godoc.org/github.com/xtaci/kcp-go).

`-kcp-nodelay`, `-kcp-interval size`, `kcp-resend size`, `kcp-nc size`: (Optional) KCP tuning options. These options need to be set consistently between the client and the server. Please refer to the [kcp](https://github.com/skywind3000/kcp/blob/master/README.en.md#protocol-configuration).

### Client options

`-publish addresses`: (Optional, recommended) ARP publishing address. If this value is set, IkaGo will reply ARP request as it owns the specified address which is not on the network, also called proxy ARP.

`-fragment size`: (Optional) Fragmentation size for listening. If this value is set, packets sending from the client to sources will be fragmented by the given size.

`-p port`: (Optional) Port for routing upstream. If this value is not set or set as `0`, a random port from 49152 to 65535 will be used.

`-r addresses`: Sources, use comma to separate multiple addresses. Packets with the same source's address will be proxied.

`-s address`: Server.

### Server options

`-fragment size`: (Optional) Fragmentation size for routing upstream. If this value is set, packets sending from the server to destinations will be fragmented by the given size.

`-p port`: Port for listening.

## Troubleshoot

1. Because IkaGo use pcap to handle packets, it will not notify the OS if IkaGo is listening to any ports, all the connections are built manually. Some OS may operate with the packet in advance, while they have no information of the packet in there TCP stacks, and respond with a RST packet or even drop the packet. **You may configure iptables in Linux, pf in macOS and FreeBSD**, or Windows Firewall in Windows (You may not need to) with the following rules to solve the problem. **If you are using mode `tcp`, you may not need to configure the firewall, but you still have to disable IP forward.**
   ```
   // Linux
   // IkaGo-server
   sysctl -w net.ipv4.ip_forward=0
   iptables -A OUTPUT -p tcp --tcp-flags RST RST -j DROP
   // IkaGo-client with proxy ARP and FakeTCP
   sysctl -w net.ipv4.ip_forward=0
   iptables -A OUTPUT -s server_ip/32 -p tcp --dport server_port -j DROP

   // macOS, FreeBSD
   // IkaGo-client with proxy ARP and FakeTCP
   sysctl -w net.inet.ip.forwarding=0
   echo "block drop proto tcp from any to server_ip port server_port" >> ./pf.conf
   pfctl -f ./pf.conf
   pfctl -e

   // Windows (You may not need to)
   // IkaGo-client with proxy ARP
   netsh advfirewall firewall add rule name=IkaGo-client protocol=TCP dir=in remoteip=server_ip/32 remoteport=server_port action=block
   netsh advfirewall firewall add rule name=IkaGo-client protocol=TCP dir=out remoteip=server_ip/32 remoteport=server_port action=block
   ```

2. IkaGo prepend packets with TCP header, so an extra IPv4 and TCP header will be added to the packet. As a consequence, an extra 40 Bytes will be added to the total packet size. For encryption, extra bytes according to the method, up to 40 Bytes, and for KCP support, another 32 Bytes. IkaGo will fragment packets which are oversize, but excessive use in the packet header will cause a significant decrease in performance.

3. IkaGo requires root permission in some OS by default. But you can run IkaGo with non-root running this command
   ```
   // Linux
   setcap cap_net_raw+ep path_to_ikago
   ```
   before opening IkaGo. If you run IkaGo with non-root, `-rule` will not work, please add firewall rules described in [troubleshoot](https://github.com/zhxie/ikago#troubleshoot) manually.

4. IkaGo acts as a router and handles segments and fragments. Generic receive offload (GRO) enabled by default in some OS may increase network performance, but affect the normal operation of IkaGo because it breaks the end-to-end principle. You can disable GRO manually.
   ```
   // Linux
   sudo ethtool --offload network_interface_like_eth0 gro off
   ```

## Limitations

1. IPv6 is not supported because the dependency package [gopacket](https://github.com/google/gopacket) does not fully implement the serialization of the IPv6 extension header.

## Known Issues

1. When using mode TCP, sticky packets problems may occur in TCP connections. If encryption is enabled at the same time, IkaGo may not be able to destick these packets.

2. Applications like VMWare Workstation on Windows may implement their own IP forwarding and forward packets that should be handled by IkaGo, resulting in abnormal operations in IkaGo.

## Todo

- [ ] Change sending packets to destinations procedures in IkaGo-server from pcap to standard connection
- [ ] Build own application layer protocol to realize functions like delay detection
- [ ] Discover the way handling packets concurrently to optimize performance

## License

IkaGo is licensed under [the MIT License](/LICENSE).
