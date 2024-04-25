---
description: >-
  Here, you will learn WiFi hacking techniques for WPS using a wireless
  interface.
cover: >-
  ../../../.gitbook/assets/DALL·E 2024-04-24 22.21.04 - A close-up digital
  illustration showing a laptop on a desk, with a prominent Wi-Fi dongle
  connected to one of its USB ports. The laptop screen display.webp
coverY: -475
---

# 🛜 WiFi WPS Hacking

***

## Disclaimer

The information provided on this blog is for educational and research purposes only. The techniques and tools discussed are intended to improve network security and should only be used on networks where explicit permission has been granted. The owner of this blog does not endorse illegal activities and will not be held responsible for misuse of the information contained herein. By using this blog, you agree that you will not use the information for unlawful purposes and that you will comply with all applicable laws and regulations.

***

## Requirements

* WiFi Dongle (capable to set monitor mode)
* Operative System: Linux ([Kali Linux](https://www.kali.org/) Recommended)
* [Reaver](https://www.kali.org/tools/reaver/) tool installed

***

## WPS&#x20;

WiFi Protected Setup or WPS allow devices to connect to a Wi-Fi network using an 8 digit pin, which sometimes is predefined by the manufacturer of the AP. During the connection establishment process, interchange of hashes are made, particularly the messages M1 to M8. Pixie Dust attack focus specifically in the messages M3 and M4, where Diffie-Hellman elements are transferred.&#x20;

Incredibly, a lot of Internet Service Providers here in Panamá are installing Access Points with WPS set by default, leaving them vulnerable and without chance to change this setting because some ISP lock their access to the router admin page to his users. Imagine exposing your home network to everyone. Sounds terrible right?

I tested different WPS attacks in my home network and also in some of my family members networks to see if they are vulnerable, and guess what? they are. In just a couple of seconds, i managed to get the Wi-Fi password using a simple Pixie Dust attack with **reaver** tool.&#x20;

Here i'll show you how to perform WPS attacks to see if your network or you neighbour network is vulnerable so you can login in your neighbour's router admin page and disable WPS for him :wink:. Not all heroes wear capes... Just kidding, don't do that without permission.

***

## Detecting targets with wash

Wash is a utility for identifying WPS enabled access points. It can survey from a live interface or it can scan a list of pcap files. Wash is included in the Reaver package.

First we need to know which wireless interface are we gonna use. To do this, we can use the command `iw dev` that will list all wireless interfaces.

<figure><img src="../../../.gitbook/assets/imagen (23).png" alt=""><figcaption></figcaption></figure>

The wireless interface that we are going to use is `wlan0`. Now we have to put this interface in monitor mode with `airmon-ng`.

```bash
# Killing processes that could interfere
$ sudo airmon-ng check kill

# after executing the previous command it is normal to lose internet connection.
# because some of the processes stoped, are processes that handle our wireless 
# interface

# starting monitor mode
$ sudo airmon-ng start wlan0
```

We can also put the interface in monitor using `iw`:

```bash
$ ifconfig wlan0 down
$ iw dev wlan0 set type monitor
$ ifconfig wlan0 up
```

Now we can execute `wash`:

<figure><img src="../../../.gitbook/assets/imagen (24).png" alt=""><figcaption></figcaption></figure>

We need to save the BSSID and channel of the network that we are going to attack with reaver.

***

## WPS Pixie Dust Attack with reaver

Now that we have our target, the only thing left to do is to start the attack.

<figure><img src="../../../.gitbook/assets/imagen (25).png" alt=""><figcaption></figcaption></figure>

***

## Recommendations for Enhancing Wi-Fi Security

**Disable WPS**

* **Check Your Router Settings:** The first and most crucial step is to access your router’s settings. Look for the Wi-Fi Protected Setup (WPS) option, and ensure it is disabled. This prevents the WPS PIN method, which is vulnerable to attacks, from being a potential entry point for attackers.&#x20;

**Update Your Router Firmware**

* **Regular Updates:** Manufacturers often release firmware updates to fix security vulnerabilities and improve functionality. By keeping your router updated, you can protect against known exploits that attackers might use to gain unauthorized access.

**Use Strong and Unique Passwords**

* **Complexity Matters:** Ensure your Wi-Fi network’s password is strong. Use a combination of upper and lower case letters, numbers, and special characters. Avoid common words and phrases to reduce the risk of brute-force attacks.

**Monitor Network Activity**

* **Stay Informed:** Regularly check the devices connected to your network. Many routers offer the ability to view a list of connected devices. If you notice any unfamiliar devices, it could indicate that your network security has been compromised.

**Educate Yourself and Others**

* **Awareness is Key:** Stay informed about the latest security threats and protection strategies. Share this knowledge with friends and family to ensure they are also taking steps to secure their networks.

**Legal and Ethical Considerations**

* **Always Seek Permission:** Never test security vulnerabilities on networks you do not own or without explicit permission from the owner. Doing so is illegal and unethical. If you want to help others secure their networks, offer your assistance and ensure they understand the process and give consent before you proceed.
