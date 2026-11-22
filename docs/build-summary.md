# Future Week Navigator: Build Summary

*How we planned, designed and built a handheld guide for Dubai Future Week, and everything we used*

## 1. Overview

Future Week Navigator is a small, square handheld device that helps visitors find their way around Dubai Future Week. The venue map is preloaded. A visitor picks a place from a list, follows a 3D map with a glowing route and arrow, and the device says "You've arrived!" through its speaker. When they're done, they drop it into a dock, where it charges and resets for the next visitor.

The planning, design and research were done on 19 November 2026. The app was then built with Claude Code and installed on the CGOTCHI, and it works perfectly.

## 2. The Problem

Dubai Future Week is a large, busy event. Visitors get a printed map and a map on their phone, but many still get lost.

The reason is that a map shows the whole venue, but it doesn't answer the two questions people actually have:

- **Where am I right now?**
- **Which way do I turn to get where I want to go?**

Reading a flat map while walking through crowds is slow and confusing, especially for children, older visitors, and people who don't read the local language.

## 3. The Solution

A dedicated guide device that does the map-reading for you.

- **No map-reading:** It shows a route and an arrow, like Google Maps.
- **No GPS:** GPS doesn't work well indoors, so the device uses its own sensors and, later, the venue's Wi-Fi.
- **No phone:** No battery drain, no mobile data, no app to download.
- **Talks to you:** It announces when you've arrived.
- **Shared and reusable:** Devices live in docking stations around the venue.

## 4. How a Visitor Uses It

1. **Pick it up at the entrance.** The device comes out of a dock. The entrance is the starting point, called the first home zone, so the device already knows where you are.
2. **Choose a place.** The "Where to?" screen shows big tiles, such as Main Stage, Robotics Zone, Food Court and Restrooms, each with its distance. Tap one.
3. **Follow the 3D map.** A tilted map, like Google Maps driving mode, shows the halls, a glowing route line, and your position as a bright arrow. A card at the top gives the next instruction, such as "Turn right in 15 m", plus the distance and time left.
4. **Arrive.** Press "I arrived", or in version 2 the device detects it automatically. The speaker says "You've arrived!" and the screen shows a big tick.
5. **New home zone.** The place you just reached becomes your new starting point, and you can pick the next place from there.
6. **Return it.** Drop the device into any dock when you're finished.

## 5. How the Device Knows Where You Are

### 5.1 Step counting (version 1)

The CGOTCHI has a built-in accelerometer, a motion sensor that feels each step you take. The app counts your steps, multiplies by an average step length (about 0.7 m), and moves your arrow along the planned route.

The weakness of motion sensors is drift: small errors build up over long distances, a bit like walking with your eyes closed.

### 5.2 Home zones: the fix for drift

This was the key idea. Every time the visitor arrives somewhere and presses "I arrived", that exact spot becomes the new home zone, and the device starts fresh from a known position. Because each walk between places is short, the small errors never get the chance to build up.

### 5.3 Wi-Fi fingerprinting (version 2)

The venue already has many Wi-Fi routers. The device can scan them and measure how strong each signal is. Every spot in the venue has its own unique mix of signal strengths, like a fingerprint.

- **Before the event:** Someone walks the whole venue with a device and records the fingerprint at many spots. This is the main setup work.
- **During the event:** The device compares what it sees with the stored fingerprints to work out where you are, and updates automatically, like Google Maps indoors.
- **Accuracy:** Usually about 3 to 5 metres. That's good enough to know which hall or zone you're in, but not to pick out an exact stall.

### 5.4 The hybrid plan

The strongest version combines both:

- Wi-Fi tells you roughly where you are.
- Step counting fills in smooth movement between Wi-Fi readings.
- Each Wi-Fi reading corrects any drift.
- The "I arrived" button stays as a backup when the Wi-Fi signal is weak.

## 6. The Hardware: CGOTCHI

We chose the CGOTCHI, an ESP32-S3 AI development kit shaped like a small square cube. It already matched the square design almost perfectly.

| Part | Details |
| --- | --- |
| Brain | ESP32-S3 dual-core chip, up to 240 MHz, with Wi-Fi and Bluetooth |
| Memory | 16 MB flash, 8 MB PSRAM |
| Screen | 2.01" colour touchscreen, 296 x 240 pixels (JD9853 display, CST810 touch) |
| Sound | 3 W speaker (ES8311 audio chip with NS4150B amplifier), plus two microphones |
| Motion | Accelerometer (SC7A20 / LIS2DH12 type), used for step counting |
| Buttons | Three on top: Confirm, Left and Right |
| Battery | 600 mAh, about 3 to 4 hours, charged by USB-C |
| Size | 48 x 41 x 26 mm |
| Price | About Rs 3,999 on Amazon India |

What the CGOTCHI does not have:

- **A compass or gyroscope.** It can count steps but can't tell which way you're facing. Version 1 assumes you're walking along the route. A small compass module could be added later so the arrow turns as you turn.
- **A full-day battery.** Three to four hours is shorter than a full event day, which is another reason the docks matter: visitors can swap for a charged device.

## 7. The Software

### 7.1 MicroPixel SDK

The CGOTCHI runs MicroPixel firmware, and its official software kit lets you write apps in C++. Apps are compiled to WebAssembly and installed over USB, so there's no risky full-firmware flash.

The parts of the kit we used:

- **Display and graphics:** Drawing the screens.
- **Input:** Touch and the three buttons.
- **Audio:** Playing the "You've arrived!" voice clip (Ogg Opus format), with a cheerful tone sequence as a fallback.
- **Sensors:** Reading the accelerometer for step counting.
- **Mode 7 plane:** A built-in feature that draws a tilted 3D ground, which is exactly what the 3D map needs.

The basic workflow:

1. Install the MicroPixel tools on the laptop.
2. Check the device with `micropixel --transport usb device status`.
3. Create the app with `micropixel init`.
4. Build and install it with `micropixel --transport usb run`.

### 7.2 The venue map

The app uses the real Dubai Future Week map, loaded onto the device.

### 7.3 Voice

The "You've arrived!" clip is made with text-to-speech, converted to Ogg Opus, and bundled with the app.

## 8. The Screen Design

We designed four screens on a design canvas. The style is dark and futuristic, with no clutter:

- **Colours:** Near-black background (#0B0E13), white text, and one glowing teal accent (#5EF2D6). Blue and orange are alternative accents.
- **Fonts:** Space Grotesk for headings, Inter for everything else.
- **Shape:** A square device with rounded corners and three buttons on top, like the real CGOTCHI.

The screens:

1. **Pick a place:** "You are at Entrance", a Wi-Fi lock indicator, "Where to?" and a 2x2 grid of large tappable tiles with distances.
2. **Follow the arrow:** A full-screen tilted 3D map with hall blocks, a glowing route, a destination label and an arrow puck. The instruction card is at the top with a close button, and "I arrived" is at the bottom right.
3. **Arrived:** The map dims behind a big teal tick, the place name, a speaker icon with sound bars and "You've arrived!", "New home zone set" and a "Pick next place" button.
4. **In the dock:** The device sits in its dock showing the battery charging, "Map up to date" and "Home reset to Entrance", with a green "Ready for next visitor" light on the dock.

How the design changed along the way:

- It started as a tall, phone-like device with a big compass arrow.
- It became square, and the arrow screen became a 3D map like Google Maps.
- Once we knew it was the CGOTCHI with a touchscreen, the on-screen button row was removed so the tiles and map could be bigger, and the speaker icon was added to the Arrived screen.

## 9. Docking Stations

Docks sit at the entrance, the exit, and a few spots in between.

- **Charging:** Devices recharge while they wait for the next visitor.
- **Easy returns:** Visitors drop the device in any dock instead of queuing at a desk.
- **Fresh reset:** Docking sets the device back to the entrance as its home zone.
- **Map updates:** If a talk moves or a stall changes, the new map goes to every docked device at once.
- **Status light:** Green means charged and ready, red means still charging.

## 10. How We Built the Plan, Step by Step

1. **The idea:** A device with the map loaded in, because printed and phone maps weren't working.
2. **The flow:** Preloaded map, no GPS, built-in battery, choose from a list.
3. **Positioning:** An IMU for motion tracking, then the home zone idea to stop drift, then the "I arrived" button.
4. **Presentation:** A first presentation to explain the idea to Papa.
5. **Wi-Fi tracking:** Using the venue's routers like Google Maps indoors, and the hybrid with step counting.
6. **Docks:** For charging, returns and resets.
7. **Updated plan doc:** Everything brought together, with open questions.
8. **Futuristic UI:** Four screens designed, then made square with a 3D map.
9. **Choosing the hardware:** The CGOTCHI, then researching its chips and software kit.
10. **Touch and voice:** The screens redesigned for touch, and the spoken arrival added.
11. **Claude Code prompt:** Written so Claude Code can set up the tools, build the app screen by screen, and install it on the CGOTCHI.
12. The working app: Claude Code set up the tools, built the app and installed it on the CGOTCHI with the real Future Week map. It works perfectly.

## 11. Tools and Resources We Used

- **Claude:** For brainstorming, planning, research and writing.
- **Docs:** The presentation for Papa and the updated plan.
- **Design canvas:** The four device screens.
- **Web research:** The CGOTCHI website and Amazon listing, the official GitHub kit, and an existing map project on the same board.
- **Claude Code:** For building and installing the app.

Links:

- [Presentation for Papa](https://claude.ai/artifact/CV4avDRo8TkpQJrgjpmV6D)
- [Updated plan](https://claude.ai/artifact/54iLkZzhpsnS7ZPWyGpGsy)
- [Screen designs](https://claude.ai/artifact/7AGMn8cgRVsWMJWb9eUkzS)
- [CGOTCHI website](https://cgotchi.com/)
- [Official CGOTCHI software kit on GitHub](https://github.com/ALTIO-AI-PRIVATE-LIMITED/micropixel-gotchi)
- [Map navigation project on the same board](https://github.com/chaitanyaasati/mapnavigation)

## 12. Challenges and Limitations

- **No compass:** The arrow can't turn as you turn. Version 1 follows the planned route instead.
- **Wi-Fi scanning:** The app kit doesn't seem to offer Wi-Fi signal scanning to apps, so Wi-Fi tracking may need extra work. That's why it's version 2.
- **Crowds:** Thousands of people absorb Wi-Fi signals, so readings on event day may differ from the survey. Step counting and the "I arrived" button help here.
- **Router changes:** If the organisers move or switch off routers, the fingerprints need re-recording.
- **Battery life:** Three to four hours, so docks and swapping are important.
- **Getting devices back:** We still need to decide between a deposit, an ID scan or simply trusting visitors.

## 13. Current Status

- **Done:** The idea, the user flow, the positioning plan, the presentation, the updated plan, the screen designs, the hardware choice and research, the Claude Code prompt, and the working app on the CGOTCHI.
- **Working: The app is installed on the CGOTCHI and works perfectly**.

### Second version: Waveshare AMOLED

There is also a second working version on the Waveshare ESP32-S3-Touch-AMOLED-1.8, built and flashed with Claude Code. It has a 1.8" AMOLED touchscreen, a speaker and microphone, and a motion sensor with a gyroscope. It connects to the venue Wi-Fi and uses the Google Geolocation API to find where you are, with step counting filling in between location fixes. The CGOTCHI version stays fully offline, so the two versions show both approaches: no internet at all, or Wi-Fi-assisted location.

## 14. Next Steps

1. Find out whether Wi-Fi scanning is possible for version 2.
2. Design and build a simple dock.
3. Decide how to make sure devices are returned.

Already done: pitched it to the Dubai Future Week organisers.
