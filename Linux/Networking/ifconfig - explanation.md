```linux
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.27.101  netmask 255.255.240.0  broadcast 192.168.31.255
        inet6 fe80::215:5dff:fe90:571c  prefixlen 64  scopeid 0x20<link>
        ether 00:15:5d:90:57:1c  txqueuelen 1000  (Ethernet)
        RX packets 470  bytes 210559 (210.5 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1244  bytes 101209 (101.2 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

# আপনার Output থেকে যা বোঝা যাচ্ছে

- **eth0** আপনার প্রধান Network Interface।
- আপনার **IPv4 Address:** `192.168.27.101`
- **IPv6 Link-local Address:** `fe80::215:5dff:fe90:571c`
- **MAC Address:** `00:15:5d:90:57:1c`
- **MTU:** `1500`
- Interface **UP** এবং **RUNNING**, অর্থাৎ Network Interface সচল।
- এখন পর্যন্ত প্রায় **470টি Packet Receive (RX)** এবং **1244টি Packet Send (TX)** হয়েছে।
- **কোনো RX/TX Error, Collision বা Carrier সমস্যা নেই**, তাই Network Interface স্বাভাবিকভাবে কাজ করছে।
- **`lo` (Loopback)** Interface শুধুমাত্র নিজের Machine-এর সাথে যোগাযোগের জন্য ব্যবহৃত হয় (`127.0.0.1` বা `localhost`)।