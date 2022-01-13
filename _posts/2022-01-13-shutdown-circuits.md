---
layout: post
title: Raspberry Pi Graceful Shutdown Circuits
subtitle: A look into graceful shutdown circuits for automotive purposes
tags: [OnBoardPi]
categories: [Software, Hardware]
---



In the times before covid I was taking a class in Computing Machinery where we were given a Raspberry Pi 4 with the intention of bare-metal programming in a lab settings. Unluckily, that semester covid came to town and all in-person teaching was suspended. 
The Pi since sat in my drawer with not much use other than collecting dust. Most recently I have been involved in developing web applications for data acquisition and visualization at an internship with [SVSControls][2], a small industrial automation 
company specializing in PLC programming and industrial computer networks. Toward the end of my placement I thought I should use my Raspberry Pi for a small-scale data server for my vehicle. My passion and interest in web-based systems drove me 
to start [OnBoardPi][1], a Raspberry Pi based OBD (On-Board Diagnostics) web server. This project began with testing many OBD libraries and tackling the challenge of actually installing the Pi in a vehicle. 

## The Problem with Automotive Power

Most modern cars (if not a diesel or an older 6V system) have a typical 12V system running all the electronics in the vehicle. Many of the electronic parts are ready made to run on 12 volts since their application does not stray much from the automotive world. 
Further, the actual day-to-day usage of a car is intermittent and unpredictable. Sure you could install a Raspberry Pi on directly to the battery (voltage regulated to 5V of course but this is another topic of discussion) and have it running all the time. 
Car batteries do not last forever and although the current draw of a Pi is relatively low, if you are not driving consistently and recharging your battery via the alternator, you will eventually be left stranded with a dead battery. Thus is the need for a way to 
indicate to the Pi it is time to shutdown when you the driver shuts the car off. 

Automotive circuitry provides us with a existing switched power source called ACC (accessory) or ignition power. This is the circuit you will see in action when you put you turn your key without actually starting the car. Most of the internal devices will illuminate such as your dash cluster,
radio and more. This is the perfect circuit to piggy back off of to tell the Pi to turn off. 

## The solution: Relays

As I began testing the software I needed to think about a graceful shutdown for my OnBoardPi. Yes, you may be able to pull the plug on your Pi and not risk any corruption but its better to be safe than sorry. The ultimate solution is to utilize the Pi's GPIO and the ACC power source to 
implement the shutdown circuit. I began researching for a solution and stumbled across this [page][3] on a way to implement a graceful shutdown. The solution was right in front of me the whole time: a relay. The only other times I had used a relay was to wire up external lights 
on my car that require a relay and rocker switch wiring harness to power the lights directly off of the battery. My circuitry and automotive electrical knowledge is limited, but a relay contains a coil which when energized operates a switch. 

I simulated the circuit with some wires and sure enough it worked so I ordered some [5 pin automotive relays][4], modified the [referred shutdown script][5] and this is what I came up with:

## Wiring Diagram
![]({{site.baseurl}}/assets/images/onboardpi/shutdown-circuit-diagram.png)
*Here is an low-fidelity wiring diagram, I am not an electrical engineer but you will hopefully get the point.* 

### Circuit Description
As said above we are relying on the ACC circuit to energize the relay. So, 

- Connect 12V+ (ACC) to pin 86 and ground it to the chassis through pin 85.
- Connect COM (relay pin 30) to ground, or to a GPIO ground pin. 
- Connect NC (relay pin 87a) to GPIO27 
- Connect NO (relay pin 87) to GPIO3

For this circuit to work, the Pi must be connected to a 5V power source such as a cigarette lighter to USB plug, like the one you charge your phone on. When you get into your car and turn your key, ACC will energize the relay. 
The switch will flip from NC to NO. Grounding GPIO3 wakes a Pi (already given 5V) from a halted state to boot. When you shut the vehicle off, ACC power is terminated and the relay goes back to its resting state i.e COM and NC closed. 
This simulates the pressing of a button or switch; by grounding GPIO27, the script will call your favourite Linux shutdown command and the Pi will gracefully turn off. 

If you desire to install a cooling fan, you can connect its negative lead to the connection from relay pin 87 and GPIO3. This ensures the fan is only powered with ignition. If you do not want the fan to shutoff for some reason, you can connect it to any empty GPIO ground pin.

### Notes
- This circuit can easily be implemented with a 4 pin automotive relay, you will need to flip some logic in how the Pi interprets the "Button" press.
- This circuit requires the Pi be provided constant 5V power for the wake feature to work.
    - A halted Pi will use less power than a running one so the shutdown effort is not obsolete. The red power indicator LED will remain on but the Pi is indeed off.
    - If you are worried about halted power consumption, wire a switch into the power supply circuit that you can manually turn off when you are not driving for an extended period of time. 
    Or, locate a delayed circuit (like auto-off headlights), or install a timer-relay to cut power to the Pi.

## Shutdown Script
I made some modifications to the existing [script][5] to both simplify and better suit my use-case. Here:
- The Pi will shutdown after 10 seconds of no ACC power, I have since reduced my own version to something like 3-5 seconds.
- If the key is returned to ACC after at least 5 seconds but noe more than 10, the Pi will reboot.

<script src="https://gist.github.com/bgunson/7646d2030991d40f6debd2de3dc783b8.js"></script>

In your own version you can substitute GPIO27 for any other pin if it makes more sense, just change it in the script.

From here you should have this script run as a service, so use systemd or similar to run this script at boot.

## Conclusion

Using relays to automate a graceful shutdown can be extended to not only Raspberry Pi's but other single board computers running an OS that require a diligent shutdown procedure in intermittent settings.
This application has its use beyond cars as well, many industrial scenarios like mobile hydro-vac excavation, oil and gas drilling and fracturing and more may require on board data acquisition using devices like Raspberry Pi's and I believe this to be
an extremely cost effective and functional solution to avoiding corruption.






[1]: https://github.com/bgunson/onboardpi
[2]: https://svscontrols.com/
[3]: https://github.com/opencardev/crankshaft/wiki/Boot,-reboot-and-shutdown-the-Pi-with-ignition-key
[4]: https://www.amazon.ca/gp/product/B07C9CKX1B/ref=ppx_yo_dt_b_asin_title_o06_s00?ie=UTF8&psc=1
[5]: https://github.com/scruss/shutdown_button