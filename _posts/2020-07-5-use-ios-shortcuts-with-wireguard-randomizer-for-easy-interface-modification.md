---
layout: post
title: "Make Integrations work for you: iOS Shortcuts app with Wireguard Randomizer"
description: "You've setup PiVPN, Mullvad & Wireguard Randomizer. Now create integrations to make your life easier."
category: articles
tags: [iOS, Shortcuts, PiVPN, Wireguard, Mullvad, RaspberryPiOS, RaspberryPi, DietPi, Networking]
feature-img: "assets/img/2020-07-5-1.png"
---

To complete this tutorial, a Wireguard Randomizer script is required. I recommend reading [my post on setting up a Wireguard Randomizer](https://archern9.github.io/articles/2020/06/29/randomize-mullvad-interface-with-cron.html).

Integrations make life easier. I've had experiences setting up IFTTT integrations for some of my use cases and an alternative to that is the iOS Shortcuts application. It lets one create automated procedures that are executed by a trigger (usually a button press from the app).

I have iteratively been building a home lab for my diverse requirements primarily driven by a Raspberry Pi 4 or a Pi Zero W. Of those requirements, one of them dictated that I [set up a PiVPN server that redirects the traffic to a commercial VPN](https://archern9.github.io/articles/2020/06/28/route-pivpn-traffic-via-mullvad.html). However, this set up had a drawback. In contrast to the Mullvad desktop & mobile applications which specifically inform the user which server the VPN is connected to, what is the IP and what region the server resides in, my setup informs no such thing. During the first few days I setup my PiVPN + Mullvad combo, I developed a morning ritual. I would use the Termius application on my iPhone to SSH into the server, and check which interface my setup was connected to using the `ip a` command. This had scope for automation.

I suggest viewing this YouTube video for comprehending [what is the iOS Shortcuts applications](https://www.youtube.com/watch?v=-NJAUmc4y-A).

## Understanding Shortcuts

Shortcuts created may be executed through three entry points.
* First, the shortcuts application of course where created shortcuts show up on an endless dual-column list;
* Second is a shortcuts app widget on the widget screen for swift access;
* The third is via Siri. This gives the advantage of executing shortcuts even without using the device. I suggest naming the shortcuts in such a way that they're easier to pronounce.

It is important to note that a shortcut is a collection of multiple actions that are executed sequentially to perform a task thereby automating it. (I took a while to wrap my head around this. My initial understanding was each action is a different shortcut & never got anywhere with my automation.)

## Shortcut - What is my IP?

When on a VPN, chances of searching for "ip address" on DuckDuckGo or a similar IP revealing services are recurrent. Complexity is added when a raspberry pi is running headless and does not have access to a browser.

Open the shortcuts application on the iOS device and tap on the *Create Shortcut* card or the *plus* icon on the top right corner. The new screen is where numerous actions are collated. Tap on *Add Action*, search for & select **Run Script over SSH**. This action, as the name states, connects to a server through SSH and executes a series of BASH commands.

* Enter the hostname of the Raspberry Pi or the IP address. If PiHole is installed on the same server, just enter *pi.hole*.
* Port 22
* The username & password of a user which can log in

In the space provided at the bottom to enter bash commands, enter the following (ignore the lines with # - They're just for illustration):

```
# This returns a country name.
country=$(curl https://am.i.mullvad.net/country)

# Lookup IP positioning on the web to understand this one (and the previous?). The API returns a City name.
city=$(curl https://am.i.mullvad.net/city)

# An alternative to this is https://am.i.mullvad.net/connected. That would warrant use of cut and other commands. I wanted to keep this simple.
ip=$(curl ifconfig.me)

# An echo at the end is the output of the script and the value is brought back to shortcut app for succeeding actions. This is important. A good addition are conditions - "If mullvad is not connected to their servers, the echo statement should not suggest it is"
echo "Mullvad is currently connected to $city, $country with IP $ip"
```

Add a new action, select *Scripting*, scroll down to the *Notification* section and select *Show Result*. The *Show Result* section should be pre-populated with *Show Shell Script Result*. Either tap on the *Play* icon on the bottom right to test or save to run later. Remember to use a name that's easier for Siri to understand (because of dialect). I named mine, **Check IP**.

![Lookup IP]({{ site.url }}/assets/img/2020-07-5-2.png)

## Shortcut - Randomize

To re-iterate, follow [my post on setting up a Wireguard Randomizer](https://archern9.github.io/articles/2020/06/29/randomize-mullvad-interface-with-cron.html) before attempting to create this shortcut. Our default setup for Randomizer is to execute the script everyday, at 0400hrs in the morning. However, retaining the ease to execute it at will is a bonus. This may be for various reasons - Connections to the current server may be running slow. Paranoia may take over and a server switch may be a whim.

Follow the steps in the previous section to create the action *Run a script over SSH*. In the space provided at the bottom to enter bash commands, enter the following.
```
# In contrast to the previous section, this implementation delegates logic to a shell script stored on the server and simply executes it. This is a good way to create complex shell scripts that may difficult to write on the app; It also enables portability. When the shortcut is shared & executed by numerous people in a household.
/opt/Wireguard-Randomizer./mullvad.sh
```

The second action focusses on re-usability. Add a new action, search for & select *Shortcut* and select *Run Shortcut*. On the shortcut card, select the shortcut created above. I chose **Check IP**. Either tap on the *Play* icon on the bottom right to test or save to run later. Remember to use a name that's easier for Siri to understand (because of dialect). I named mine **Randomize Mullvad**.

![Randomize Mullvad]({{ site.url }}/assets/img/2020-07-5-3.png)

Pro Tip: Go ahead and dictate Siri, "Hey Siri, Check IP" or "Hey Siri, Randomize Mullvad".