---
description: >-
  In this tutorial, we are going to execute  an Evil-twin/portal attack to
  attract users to a false WiFi hotspot and then use an evil portal to harvest
  user credentials.
cover: https://img-c.udemycdn.com/course/750x422/2948934_985b_4.jpg
coverY: 42
layout:
  cover:
    visible: true
    size: hero
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
---

# 🛜 WiFi Evil Twin/Portal Attack with a WiFi Pineapple

## How an Evil Twin attack works?

<figure><img src="https://www.researchgate.net/publication/321122614/figure/fig5/AS:631949064421377@1527679806852/Illustration-of-an-Evil-Twin-Attack-The-attacker-can-successfully-lure-a-victim-into.png" alt=""><figcaption><p>Evil Twin</p></figcaption></figure>

An evil twin attack is a type of WiFi attack that creates a hotspot  mimicking or cloning a legitimate WiFi hotspot.  The main objective of this attack is to trick users into connecting to the fraudulent WiFi hotspot instead of the real one. Once a user is connected to the evil twin, an attacker can perform various attacks, such as spying on user's traffic or conducting a Man in the Middle (MitM) attack.

## How an Evil Portal attack works?

An evil portal attack involves redirecting a user to a captive portal page once they are connected to a malicious network. This captive portal can imitate the captive portal of the internet service provider, a hotel login page, etc. Generally speaking, the main purpose of the malicious captive portal (evil portal) is to recollect user sensitive information, such as login credentials. To carry out an evil twin attack, you need to follow the next steps:

1. **Establishing a connection**: First, the attacker has to trick a user into connecting to a malicious hotspot controlled by the attacker.
2. **Traffic Redirection**: Once a user is connected to the evil hotspot, the user's web traffic is redirected using techniques like DNS manipulation or modifying iptables rules. This will force the user to visit a specific page, like the captive portal mentioned before.
3. **Evil Portal Presentation**: The user sees the evil portal, and if the portal is convincingly designed, the user will input sensitive information in an HTML form, thinking  the portal is legitimate.
4. **Data Stealing**: The evil portal logs the user's credentials, and now the attacker can see this sensitive information.

## How a WiFi hotspot redirects users web traffic to a captive portal?

The redirection to a captive portal is achieved through a combination of firewall rules and DNS manipulation within the hotspot or the server that controls network traffic. Here's a high level description:

### **1. Firewall Rules**:&#x20;

* Upon a new device connection to the network, firewall rules are set to redirect all HTTP/HTTPS traffic from this device to the captive portal IP address. This is often done using iptables or other tools for packet manipulation.

### **2. DNS Manipulation**:&#x20;

* In addition to the firewall rules, a fake DNS server is set up or manipulated to resolve all requests  to the IP address of the captive portal. Thus, when  the user tries to access any website, the DNS request is redirected to the captive portal.&#x20;

### **3. Session and Authentication**:&#x20;

* Once the user  completes the interaction with the captive portal, firewall rules and DNS settings are updated to allow the user's device to connect to the network normally.

## Evil Portal Module

The Evil Portal module allows us to configure and deploy an Evil-Portal attack automatically . To use this module, we first need to install it  from the modules section on the left panel.&#x20;

<figure><img src="../../../.gitbook/assets/imagen (1) (1).png" alt=""><figcaption></figcaption></figure>

Once the module is installed, we need to clone the following repository to our machine:

* [https://github.com/kleo/evilportals](https://github.com/kleo/evilportals)

After cloning the repository, we need to trasnfer it to the WiFi Pineapple using SCP to the `/root/portals` path. Once this is done, we should see all captive portal templates in the Evil Portal module on the wifi pineapple.

<figure><img src="../../../.gitbook/assets/imagen (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

Now, to enable the evil portal, we need to start the web service and select the portal we want to show to the users. After activating the web server and the desired captive portal template,  we should start the evil portal using the "Start" button located above the "start web server" button.

<figure><img src="../../../.gitbook/assets/imagen (2) (1).png" alt=""><figcaption></figcaption></figure>

## Configuring the Open AP

First, I recommend to hide the Open Access Point to  avoid unwanted attention. It's important to enable the option that allows the APs in the Spoofed AP Pool to respond to probe requests. With this setting enabled, all clients can connect to the APs listed in the Spoofed AP Pool.

<figure><img src="../../../.gitbook/assets/imagen (3).png" alt=""><figcaption></figcaption></figure>

## Add the SSID to the Spoofed AP Pool

Now we need to add the SSID that we want to impersonate to the Spoofed AP Pool. When an SSID is added to the Spoofed AP Pool, the WiFi Pineapple  will broadcast this SSID as if it were a legitimate WiFi network. This means that the WiFi Pineapple will transmit the specified SSID as an open WiFi network or as a WiFi that appears  to be legitimate. In this way, any device nearby that is searching for the specified SSID could potentially connect to the fake AP.

Here's what happens at a technical level:

1. **Beacon Frames Transmission:** The WiFi Pineapple starts to send "beacon frames" with the SSID specified in the Spoofed AP Pool. These beacon frames are packets that announce the existence of a WiFi network to every device nearby.
2. **Scanning and Connection**: All devices scan near WiFI networks searching for known networks. If, after scanning, those devices find a known SSID that is in the Spoofed AP Pool, they are likely to connect to the fake AP.
3. **Establishing a Connection**: Once a device attempts to establish a connection with the broadcasted  SSID, the WiFi Pineapple allows the connection, acting as an Access Point.
4. **Traffic Interception**: Now that the target device is connected to the WiFi Pineapple, all of its network traffic passess through the WiFi Pineapple. This setup allows the WiFi Pineapple to perform various attacks like MitM, packet sniffing and other malicious activities.

In this case, we are going to add the SSID  "Star Bucks Free Wifi" to the Spoofed AP Pool.

<figure><img src="../../../.gitbook/assets/imagen (4).png" alt=""><figcaption></figcaption></figure>

While Scanning nearby WiFi networks, we can see the SSID we just added to the Spoofed AP Pool.

<figure><img src="../../../.gitbook/assets/imagen (5).png" alt=""><figcaption></figcaption></figure>

When a device connects to the fake AP that we just started, the device is going to be redirected to the evil portal where a login credential will be requested from the user.

<figure><img src="../../../.gitbook/assets/imagen (6).png" alt=""><figcaption></figcaption></figure>

If the user introduce his credentials, a notification will appear in the WiFi Pineapple telling us that someone has been fooled and we can see his credentials from the Evil Portal Module.

<figure><img src="../../../.gitbook/assets/imagen (7).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/imagen (8).png" alt=""><figcaption></figcaption></figure>
