Home Assistant controlled underfloor heating system via Apple HomeKit
============

This document will assist you on how to make your own (water-based) underfloor heating controller powered by Home Assistant, which enables you to use the temperature sensors and wall plugs (for the actuators on the underfloor heating manifold, i.e. the devices responsible for opening and closing each water circut) you want.

# Why do I need a smart heating system in the first place?
In short: You need a smart heating system in order to have room profiling and weather compensation. Traditional "dumb" heating components react to the current conditions, e.g. turn "off" when the desired temperature is reached or "on" when it's already too cold, which means that they're always lagging behind what is actually needed _right now_. Smart heating systems on the other hand create (or should, at least) a heating profile for each room so that the system knows how long adjustments take to have an effect, and can thus proactively adjust the temperature to better maintain the desired level. In addition, systems which offer wheather compensation can further improve upon the indoor climate by, e.g. turning "off" the heating at the correct time based on rising temperatures in the afternoon, etc.


# Purpose and components
The purpose of the project is to create a smart unified heating system which includes both radiators and underfloor heating, and which can work with the (my) current underfloor heating manifold.

> My outset for this project is my two-storey house which has radiators on the first floor, and water-based underfloor heating on the ground floor (and in the first floor bathroom) - totalling 4 radiators and 5 underfloor heating zones (water circuts) in all:
  >
  >  - Initially the radiator thermostats were "dumb" Danfoss thermostats, which I quickly replaced with [Tado Smart Radiator Thermostats](https://www.tado.com/dk-en/smart-radiator-thermostat-add-on).
  >  - The underfloor heating manifold and actuators are from Pettinaroli, the controller for the actuators and circulation pump is a no-name 868 MHz thingy, "Funk B 2070-2" (which appears to be used by Altech, Pettinaroli and others), with "dumb" [Altech](https://www.altechcorp.com) wireless thermostats.

In the case of this particular project, the components are:

1. [Home Assistant](https://www.home-assistant.io/) running on a Raspberry Pi for handling automations.
1. [Wireless Temperature Sensors](https://www.tado.com/dk-en/wireless-temperature-sensor) by Tado for measuring the temperature.
1. [Wall Plugs](https://www.fibaro.com/en/products/wall-plug/) (Apple HomeKit edition) by Fibaro for powering the actuators and the circulation pump.
1. Actuators which fit your underfloor heating manifold and which can be powered by the wall plugs. Note that your actuators need to be 230V so that they can be plugged into the wall plugs. 12V actuators will require an adaptor.

... but you can probably make it work with most vendors (for items 2 & 3). You need:

1. One "wireless temperature sensor" (with Tado you can actually add additional devices which can provide better readings for each room/underfloor heating zone).
1. One "wall plug".
1. One "actuator".

... for each underfloor heating zone.

> Yes, I know that there is probably a smarter approach than using wall plugs, but I haven't been able to find a relay (e.g. something running [Tasmota](https://tasmota.github.io/docs/)) which fit my needs, specifically 230V for the pump control. And, yes - I have also written to Tado and suggested that they make a controller for underfloor heating, but so for I've had no luck in that direction.


# Add devices to Home Assistant

## Adding Fibaro wall plugs to Home Assistant
If you've already had the Fibaro device paired with Apple HomeKit, you will need to reset it (and if not you can skip this part). To do this: 

1. Unplug the Fibaro device from the power socket.
1. Hold down the power button on the Fibaro device and reinsert it into the power socket, while continuing to hold down the power button. The Fibaro device will start cycling through its various colors (green, red, yellow).
1. When the Fibaro device displays a yellow color (which signifies resetting), let go of the power button and immediately click (and release) it again to confirm that you wish to reset it (after which it will briefly display a green color, and then start flashing blue). The Fibaro device will restart, after which it will flash yellow indicating that it's in paring-mode.

> The Fibaro device will not remain in pairing-mode indefinitely. If it stops flashing yellow, you can simply unplug it and reinsert it into the power socket. After its initial startup routine, it will automatically enter pairing-mode again.

Now, on your cell phone:

1. Open your cell phones Wi-Fi settings, which should now display the Fibaro device.
2. Select the Fibaro device and allow it to connect to your local Wi-Fi network.

> If the Fibaro device fails to connect to the Wi-Fi network, try resetting and connecting it to the network again. In one instance, I have had to repeat the process 3-4 times before it connected correctly.

Now, in Home Assistant:

1. Go to: "Settings > devices & services", which should now display the reset Fibaro device which Home Assistant has now been able to discover.
2. Click the "configure" button.
3. Enter the "paring code" (you don't need to type in the dashes) for the Fibaro device, and click "submit".
4. Choose the area in which the Fibaro device is related, e.g. "technical room", and click "finish".

The Fibaro device will now be grouped with other Apple HomeKit devices in a "HomeKit Device" group.


## Adding Tado wireless temperature sensors to Home Assistant
Set the Tado bridge into pairing mode and add it to Home Assistant. You may need to "reset" the Tado bridge, after which it will automatically show up in Home Assistant. The Tado bridge will also be grouped with other Apple HomeKit devices in the "HomeKit Device" group. When the bridge has been added to Home Assistant, all of the connected Tado devices will be available.

> In order to see the Tado devices in Apple's Home app again, you can add a "HomeKit Bridge" to Home Assistant, and choose which "entities" to share. Upon completing the configuration, the selected entities show up as normally in Apple's Home app.

### Regarding Tado, Home Assistant and Apple HomeKit
Home Assistant has a built-in plug-in for Tado which controls the said devices via Tado's public API which is available via the Internet, and which has a limited polling-rate which defaults to updating every 5 minutes. By pairing Tado directly with Home Assistant via Home Assistant's "HomeKit Device", updates from Tado devices are registered instantly in Home Assistant without any external dependencies.


# Home Assistant scripts
You need to create some scripts in Home Assistant which can be invoked by automations later on.


## Turn off pump
This script turns off the pump if its invoked and all of the wall plugs are either `off` or `unavailable` (or `unknown`) for some reason, as there's no point in running the circulation pump if no water circuts are open. By including <u>all</u> of the wall plugs in the script, when can simply call the same script from all of the automations which we will do later on.

### Script
1. Name: `Turn off pump`
1. Mode: `Single`

Sequence:
1. Template condition:
    1. Condition type: `Template`
    1. Value template:

    ```
    {{
          states('switch.wall_plug_1_wall_plug_outlet') in ['off', 'unavailable', 'unknown']
      and states('switch.wall_plug_2_wall_plug_outlet') in ['off', 'unavailable', 'unknown']
      and states('switch.wall_plug_3_wall_plug_outlet') in ['off', 'unavailable', 'unknown']
      and states('switch.wall_plug_4_wall_plug_outlet') in ['off', 'unavailable', 'unknown']
      and states('switch.wall_plug_5_wall_plug_outlet') in ['off', 'unavailable', 'unknown']
    }}
    ```

1. Device:
    1. Device: `Zone circulation pump`
    1. Action: `Turn off ...`


## Turn on pump
This script turns on the pump if its invoked and one or more of the wall plugs are `on`, i.e. we want the pump to circulate the water when one or more water circuts are open. Again, by including all of the wall plugs in the script, when can simply call the same script from all of the automations which we will do later on.

### Script
1. Name: `Turn on pump`
1. Mode: `Single`

Sequence:
1. Delay: {E.g. 2 minutes, thus giving the actuators time to open the water circuts before the pump starts circulating the water}

1. Template condition:
    1. Condition type: `Template`
    1. Value template:
    ```
    {{
         is_state('switch.wall_plug_1_wall_plug_outlet', 'on')
      or is_state('switch.wall_plug_n_wall_plug_outlet', 'on')
    }}
    ```

    Include a line for each wall plug.

1. Device:
    1. Device: `Zone circulation pump`
    1. Action: `Turn on ...`


# Home Assistant automations for circulation pump

## Zone circulation pump off (script)
The purpose of this script is to turn off the circulation pump when <u>all</u> water circuts are closed (actuators are inactive).

Triggers:
1. Add trigger: `Template`
1. Value template:

```
{{
      states('switch.wall_plug_1_wall_plug_outlet') in ['off', 'unavailable', 'unknown']
  and states('switch.wall_plug_n_wall_plug_outlet') in ['off', 'unavailable', 'unknown']
}}
```

Include a line for each wall plug.

Actions:
1. Add action: `Call service`
1. Service: `Script: Turn off pump`


## Zone circulation pump on (script)
The purpose of this script is to run the circulation pump when <u>one or more</u> water circuts are open (actuators are active).

Triggers:
1. Add trigger: `Template`
1. Value template:

```
{{
     is_state('switch.wall_plug_1_wall_plug_outlet', 'on')
  or is_state('switch.wall_plug_n_wall_plug_outlet', 'on')
}}
```

Include a line for each wall plug.

Actions:
1. Add action: `Call service`
1. Service: `Script: Turn on pump`


# Home Assistant automations for each zone
The automations in this section should exist for each zone, i.e. copy, paste, modify after creating the first one.

## Zone {number} thermostat idle
The purpose of this automation is to turn off an actuator when the corresponding thermostat stops requesting heat.

Triggers:
1. Add trigger: `Template`
2. Value template:

```
{{
  is_state_attr('climate.zone_n_tado_smart_thermostat', "hvac_action", "idle")
}}
```

Actions:
1. Add action: `Device`
2. Device: {select the wall plug actuator}
3. Action: `Turn off ...`


## Zone {number} thermostat heating
The purpose of this scautomationript is to turn on an actuator when the corresponding thermostat is requesting heat.

Triggers:
1. Add trigger: `State`
2. Value template:

```
{{
  is_state_attr('climate.zone_n_tado_smart_thermostat', "hvac_action", "heating")
}}
```

Actions:
1. Add action: `Device`
2. Device: {select the wall plug actuator}
3. Action: `Turn on ...`


## Zone {number} start-up
The purpose of this automation is to register when a said actuator (wall plug) becomes available to Home Assistant, and turn on if the corresponding thermostat is requesting heat.

Triggers:
1. Add trigger: `State`
1. Entity: {select the wall plug actuator}
1. From: `Unavailable`

Conditions:
1. Add condition: `Template`
1. Value template:

```
{{
  is_state_attr('climate.zone_n_tado_smart_thermostat', "hvac_action", "heating")
}}
```

Actions:
1. Add action: `Device`
1. Action: `Turn on ...`


## Zone {number} disconnect
The purpose of this automation is to register when a said actuator (wall plug) becomes uavailable to Home Assistant, e.g. if it's disconnected from the its power socket, and run the script to potentially stop the circulation pump (if no water circuts are open).

Triggers:
1. Add trigger: `Template`
1. Template value:

```
{{
  states('switch.wall_plug_1_wall_plug_outlet') in ['unavailable', 'unknown']
}}
```

Actions:
1. Add action: `Call service`
1. Service: `Script: Turn off pump`
