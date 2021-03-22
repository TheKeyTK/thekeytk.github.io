---
title: Capture all android network traffic
layout: post
---

So you are performing a pentest on an android app and you have got into a situation where basic certificate pinning bypass doesn’t work. Or you have been dealing with custom protocol instead of good ol’ HTTP. The goal of this post is to teach you how to capture any network traffic on your android device (no root required).

How does it work you ask? We are going to use a fantastic app, provided by Andrey Egorov([@egorovandreyrm](https://github.com/egorovandreyrm){:target="_blank"}.), [pcap remote](https://play.google.com/store/apps/details?id=com.egorovandreyrm.pcapremote){:target="_blank"}.
It works by creating a VPN connection and capturing all the traffic going through that connection and redirecting it to the wireshark where we can analyze it in real-time.

<br/>
Prerequisites:
* android device/emulator
* [apktool](https://ibotpeaches.github.io/Apktool/install/){:target="_blank"} 
* [wireshark](https://www.wireshark.org/download.html){:target="_blank"} (version 3.0+)
* [pcap remote](https://play.google.com/store/apps/details?id=com.egorovandreyrm.pcapremote){:target="_blank"}

<br/>
Let's start..

If you are testing on an android version greater than 7.0 you are going to need to tamper with an apk a little, since google changed network security policy and made it "harder" for us to play. 
Basically what we need to do is to modify the application to accept any self-signed CA so we can intercept and decrypt the traffic.
For this example, I'm going to use 'twitter' android app. Let me show you.

<br/>
## Modify target app
Use apktool to decompile your apk.
```text
apktool d twitter.apk
```
	
In AndroidManifest.xml edit application tag and add 'networkSecurityConfig' parameter. Your target application might already have this value set, in that case you can skip this step.
```text
android:networkSecurityConfig="@xml/network_security_config"
```
![twitter-android-manifest-modified](/assets/twitter-andrmanifest-modified.png)


Create/edit network security configuration file. It is stored at location specified in the application tag in AndroidManifest. By default its location is:  'res/xml/network_security_config.xml'. 

Edit it to look like this:
```text
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="true">
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

Recompile the apk, sign it and install it on the device.
```text
apktool b twitter -o twitter-patch.apk
keytool -genkey -v -keystore key.keystore -alias sign -keyalg RSA -keysize 2048 -validity 10000
jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore key.keystore twitter-patch.apk sign
```


<br/>
## Setup pcap remote
Now let's setup pcap remote.

First we are going to install ssl certificate. Tap on 3 dots in top right corner, then settings and then 'Install' under 'SSL Certificate' category, follow the installation dialogs and set the device password if prompted.
![pcap-remote-install-cert](/assets/pcap-remote-install-cert.png){: .center-image }

Make sure to select 'SSH Server' as capturing mode and toggle 'Make HTTPS/TLS connections decryptable' .
![pcap-setup](/assets/pcap-setup.png){: .center-image }

On the top click on the triangle icon with number one in it and select your target application.
![pcap-setup](/assets/pcap-triangle.png){: .center-image }
![pcap-setup](/assets/pcap-chose-app.png){: .center-image }


<br/>
## Setup wireshark
Now it's time to connect wireshark with pcap remote.  
**NOTE: If you are using android emulator for testing, make sure to portforward the port.**
```text
adb forward tcp:15432 tcp:15432
```
Open wireshark and select 'SSH remote capture: sshdump'.
![pcap-setup](/assets/wireshark-sshdump.png){: .center-image }

Enter your phone's IP address (or 127.0.0.1 if you are working with an emulator) and port that pcap remote is running on. Also on the 'Authentication' tab enter any ssh username and password and click start.
![pcap-setup](/assets/wireshark-ssh-server.png){: .center-image }
![pcap-setup](/assets/wireshark-ssh-auth.png){: .center-image }

Start your application and analyze decrypted traffic in realtime.
![pcap-setup](/assets/wireshark-decrypted-1.png){: .center-image }
![pcap-setup](/assets/wireshark-decrypted-2.png){: .center-image }

The end!
