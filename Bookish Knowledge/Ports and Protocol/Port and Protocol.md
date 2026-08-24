Sure. If you want it ordered by **port number in ascending order**, the sequence should be:

|**Port**|**Protocol**|**OSI Layer**|**Common Interview "Twist" / Trick Question**|
|--:|---|--:|---|
|**20 / 21**|**FTP**|Layer 7|Port **20** is for data transfer; Port **21** is for control/commands.|
|**22**|**SSH**|Layer 7|Secure alternative to Telnet; encrypts the command-line session.|
|**23**|**Telnet**|Layer 7|**Big Red Flag!** Sends credentials in plaintext; interviewer may ask why you shouldn't use it.|
|**25**|**SMTP**|Layer 7|Used for _sending_ emails between mail servers.|
|**53**|**DNS**|Layer 7|**Highly Asked:** Uses **UDP** for quick queries but switches to **TCP** for zone transfers and other cases.|
|**80**|**HTTP**|Layer 7|Unencrypted text; traffic can be sniffed/intercepted.|
|**110**|**POP3**|Layer 7|Downloads mail to a local device; traditionally associated with removing mail from the server.|
|**143**|**IMAP**|Layer 7|Keeps mail synchronized on the server across multiple devices.|
|**443**|**HTTPS**|Layer 7|Uses **TLS** for encryption. TLS is commonly associated with the Presentation Layer (Layer 6) in the OSI model, although modern TCP/IP implementations don't map perfectly to OSI layers.|
|**3389**|**RDP**|Layer 7|Used for **Windows Remote Desktop** connections.|

### 🔥 Easy way to memorize

**20/21 → FTP**  
**22 → SSH**  
**23 → Telnet**  
**25 → SMTP**  
**53 → DNS**  
**80 → HTTP**  
**110 → POP3**  
**143 → IMAP**  
**443 → HTTPS**  
**3389 → RDP**

One important interview point: **20/21 are both FTP**, so when sorting numerically, treat them as **20 and 21**, not as a single value.