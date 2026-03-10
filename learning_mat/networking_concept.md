# Basic Networking Concepts — Interview Cheat Sheet

> Super simple explanations of core networking concepts for interviews.

---

## 1. How Computers Talk (OSI & TCP/IP Models)

Networks use layered models to break down how data is sent. The **OSI Model** (7 layers) is theoretical, while the **TCP/IP Model** (4 layers) is what the internet actually uses.

### The "Sending a Letter" Analogy ✉️

| Layer | What It Does | Letter Analogy |
|-------|-------------|----------------|
| **Application** | What the user sees (HTTP for web, SMTP for email) | You **write** the letter |
| **Transport** | How to send it (TCP for reliable, UDP for fast) | Choose **registered post** (TCP) or **postcard** (UDP) |
| **Network (Layer 3)**| IP Addresses and Routing across the internet | The **postal sorting office** figures out the city |
| **Data Link (Layer 2)**| MAC Addresses and Switches on the local network | The **local postman** delivers to the exact house |
| **Physical** | Cables, Wi-Fi signals (1s and 0s) | The **mail truck** driving on the road |

---

## 2. Layer 2 — The Local Network (Switches & MACs)

Layer 2 handles communication **within the same local network** (like computers in the same office or your home Wi-Fi).

* **MAC Address**: The physical, permanent "hardware ID" of a network card (e.g., `AA:BB:CC:11:22:33`).
* **Switch**: A device that connects computers together. It learns MAC addresses and sends data only to the specific computer it’s meant for.
* **VLAN (Virtual LAN)**: A way to split one physical switch into multiple virtual, isolated networks (e.g., separating "Guest Wi-Fi" from "Employee Wi-Fi").
* **ARP**: The protocol that asks, "Who has this IP address? Tell me your MAC address!"

---

## 3. Layer 3 — The Internet (Routers & IPs)

Layer 3 handles communication **between different networks** (like your home network talking to Google's network).

* **IP Address**: Your computer's "logical address" on the network (e.g., `192.168.1.10`). It can change depending on what network you connect to.
* **Router**: A device that connects different networks together. It reads the IP address and decides the best path to send the data.
* **Default Gateway**: The IP address of your router. If your computer doesn't know where to send data, it sends it to the default gateway.
* **NAT (Network Address Translation)**: Your home router does this. It lets all your home devices share a single "public" internet IP address while keeping "private" IPs internally.

---

## 4. Layer 4 — Transport (TCP vs UDP)

Once data reaches the right computer, how should it be delivered to the app?

### TCP (Transmission Control Protocol)
* **What it is**: Reliable, ordered, and checks for errors.
* **Analogy**: A phone call. You say "Hello", they say "Hello back" (Handshake). If they don't hear you, you repeat it.
* **Used for**: Loading web pages (HTTP), sending emails, downloading files.

### UDP (User Datagram Protocol)
* **What it is**: Fast, unordered, and best-effort (no error checking or guarantees).
* **Analogy**: Shouting across a crowded room. You just yell the message and hope they hear it. If they miss a word, you don't repeat it; you just keep talking.
* **Used for**: Live video calls, online gaming, live streaming.

---

## 5. Basic Protocols You Should Know

### DNS (Domain Name System)
* **What it does**: It's the "Phonebook of the Internet".
* **Example**: Translates `www.google.com` into an IP address like `142.250.190.46` so your computer can connect to it.

### DHCP (Dynamic Host Configuration Protocol)
* **What it does**: Automatically gives out IP addresses.
* **Example**: When you connect to Starbucks Wi-Fi, DHCP automatically assigns your phone an IP address so it can browse.

### HTTP / HTTPS
* **What it does**: The protocol for loading web pages. HTTPS is the secure, encrypted version.
* **Common Status Codes**:
  * **200 OK**: Success!
  * **401 Unauthorized**: You need to log in.
  * **403 Forbidden**: You are logged in, but don't have permission.
  * **404 Not Found**: The page doesn't exist.
  * **500 Internal Server Error**: The server crashed.

---

## 6. Common Interview Questions Answered Simply

**Q: What happens when you type `google.com` into your browser?**
1. **DNS**: Your computer asks DNS to turn `google.com` into an IP address.
2. **TCP**: Your computer shakes hands (connects) with Google's server.
3. **HTTP**: Your computer asks for the webpage.
4. **Response**: Google sends the webpage back.
5. **Render**: Your browser draws the page on your screen.

**Q: What's the difference between a Hub, a Switch, and a Router?**
* **Hub**: Dumb. Receives data and copies it to *every* plugged-in computer.
* **Switch**: Smart. Learns MAC addresses and sends data *only* to the correct computer on the local network.
* **Router**: The Boss. Connects entirely different networks together using IP addresses.

**Q: Why use UDP instead of the reliable TCP?**
* Because for things like live video, **speed matters more than perfection**. If a single frame of video is lost, you'd rather the video keep playing (UDP) instead of pausing the stream to wait for the lost frame to be resent (TCP).
