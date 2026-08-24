হ্যাঁ, অবশ্যই। **Linux-এও `ping` কমান্ড আছে**, এবং এটি Windows-এর `ping`-এর মতোই কাজ করে।

উদাহরণ:

```bash
ping github.com
```

অথবা

```bash
ping google.com
```

Output হতে পারে:

```text
PING github.com (140.82.114.4) 56(84) bytes of data.
64 bytes from 140.82.114.4: icmp_seq=1 ttl=57 time=25.4 ms
64 bytes from 140.82.114.4: icmp_seq=2 ttl=57 time=24.8 ms
64 bytes from 140.82.114.4: icmp_seq=3 ttl=57 time=25.1 ms
```

---

# Windows vs Linux-এর পার্থক্য

এখানে একটি গুরুত্বপূর্ণ পার্থক্য আছে।

### Windows

```cmd
ping github.com
```

ডিফল্টভাবে **৪টি Packet** পাঠিয়ে নিজে থেকেই বন্ধ হয়ে যায়।

---

### Linux

```bash
ping github.com
```

ডিফল্টভাবে **অনবরত Ping করতে থাকে** যতক্ষণ না আপনি বন্ধ করেন।

বন্ধ করতে:

```text
Ctrl + C
```

তারপর Summary দেখাবে:

```text
--- github.com ping statistics ---
5 packets transmitted, 5 received, 0% packet loss
```

---

# নির্দিষ্ট সংখ্যক Ping পাঠাতে

যদি Windows-এর মতো ৪টি Ping চান:

```bash
ping -c 4 github.com
```

এখানে:

- `-c` = count
    
- `4` = ৪টি Packet
    

Output শেষে নিজে থেকেই বন্ধ হবে।

---

# Localhost Ping

```bash
ping 127.0.0.1
```

অথবা

```bash
ping localhost
```

এটি আপনার নিজের কম্পিউটারকে Ping করবে।

---

# IP Address Ping

```bash
ping 8.8.8.8
```

এটি Google DNS Server-কে Ping করবে।

---

# DNS কাজ করছে কিনা পরীক্ষা

প্রথমে IP Ping করুন:

```bash
ping 8.8.8.8
```

যদি এটি কাজ করে, তারপর Domain Ping করুন:

```bash
ping google.com
```

- যদি IP কাজ করে কিন্তু Domain না করে, তাহলে সাধারণত **DNS সমস্যা**।
    
- যদি দুটোই না করে, তাহলে **Network বা Internet সমস্যা** হতে পারে।
    

---

# আরও কিছু দরকারী Network Command

|Command|কাজ|
|---|---|
|`ping github.com`|Server Reachable কিনা পরীক্ষা|
|`ip addr`|আপনার IP Address দেখুন|
|`ip route`|Routing Table দেখুন|
|`ss -tuln`|কোন Port Listen করছে দেখুন|
|`curl https://github.com`|HTTP Request পাঠান|
|`wget https://example.com/file`|File Download করুন|
|`traceroute github.com`|Packet কোন কোন Router দিয়ে যাচ্ছে দেখুন|
|`nslookup github.com`|Domain-এর IP বের করুন|
|`dig github.com`|বিস্তারিত DNS তথ্য দেখুন|

---

## Windows থেকে Linux-এ Command Mapping

| Windows               | Linux                                      |
| --------------------- | ------------------------------------------ |
| `ping github.com`     | `ping github.com` ✅                        |
| `ipconfig`            | `ip addr` বা `ifconfig`                    |
| `tracert github.com`  | `traceroute github.com`                    |
| `netstat -ano`        | `ss -tulnp` (বা `netstat` যদি ইনস্টল থাকে) |
| `nslookup github.com` | `nslookup github.com` বা `dig github.com`  |

আপনি যেহেতু Windows Background থেকে Linux শিখছেন, এই ধরনের Windows ↔ Linux Command Mapping শিখলে Linux Commands অনেক দ্রুত মনে রাখতে পারবেন।