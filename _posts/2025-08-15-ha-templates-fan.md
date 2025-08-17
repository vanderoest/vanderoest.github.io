---
layout: post
title: Home Assistant Template fan
subtitle: Digging HA Dungeons
cover-img: /assets/img/home-automation-banner1.jpg
thumbnail-img: /assets/img/homeassistant-logo.png
share-img: /assets/img/home-automation-banner1.jpg
tags: [homeassistant]
author: Arjan
---

Because I kept running into quirks with Home Assistant, and because most how-tos floating around are still based on the old HA config style, I decided to revive my nerd blog. Mainly to document my own struggles and fixes, so others (or future me) can benefit.

First case: my dumb Orcon MVS-15RH mechanical ventilator. Ages ago I wanted to integrate it into my home automation setup. Orcon doesn’t provide anything useful for that, so I started sniffing around to see how I could either hijack one of their remotes or mess with the control board. Long story short: I discovered the motor can be driven directly with a 0-10V or PWM signal.

That made the solution straightforward: I used a Qubino Flush 0-10V sourcing dimmer (important: the motor expects a source signal, so don’t grab a sinking type dimmer). Integrated it into my modest Z-wave network, connected it to Home Assistant, and then exposed it via the HomeKit add-on.

This worked, but it came with a long-standing irritation. Since the Qubino is technically a dimmer, Home Assistant tagged it as a light. HomeKit happily followed, so in the app it showed up as a lamp while it was actually ventilation. Commands like “set ventilation to 20%” worked fine because of the name, but the big issue: when I said “switch off all lights,” the ventilation also died completely. Not great.

Time to fix that. The device domain needed to become fan instead of light. You can’t just flip that directly, but you can create a fan template that controls the dimmer. Then you only export the fan device to HomeKit. Problem solved.
The template syntax changed recently and it threw me off for a while, but after some digging this turned out to be the way forward:

```
template:
  - fan:
      - name: "Ventilatie Zolder"
        unique_id: ventilatie_zolder
        state: "{{ states('light.ventilatie_zolder') }}"
        percentage: >
          {{ (state_attr('light.ventilatie_zolder', 'brightness') / 255 * 100) | int }}
        turn_on:
          action: light.turn_on
          target:
            entity_id: light.ventilatie_zolder
        turn_off:
          action: light.turn_off
          target:
            entity_id: light.ventilatie_zolder
        set_percentage:
          action: light.turn_on
          target:
            entity_id: light.ventilatie_zolder
          data:
            brightness: >
              {{ (percentage / 100 * 255) | int }}
```

Note that Visual Code Studio will warn on the second line with `property fan not allowed`, but this is not true. I [raised an issue](https://github.com/hassio-addons/addon-vscode/issues/1022) with the maintainer of the VSCode-addon.

As reference, this used to be the old but now deprecated method using the fan template:

```
fan:
  - platform: template
    fans:
      ventilatie_zolder:
        unique_id: ventilatie_zolder
        friendly_name: "Ventilatie Zolder"
        value_template: "{{ states('light.ventilatie_zolder') }}"
        percentage_template: >
          {{ (state_attr('light.ventilatie_zolder', 'brightness') / 255 * 100) | int }}
        turn_on:
          action: light.turn_on
          target: 
            entity_id: light.ventilatie_zolder
        turn_off:
          action: light.turn_off
          target:
            entity_id: light.ventilatie_zolder
        set_percentage:
          action: light.turn_on
          target: 
            entity_id: light.ventilatie_zolder
          data:
            brightness: >
              {{ ( percentage / 100 * 255) | int }}  
```


