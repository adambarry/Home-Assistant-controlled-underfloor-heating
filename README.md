Home Assistant controlled underfloor heating system via Apple HomeKit
============

This document will assist you on how to make your own (water-based) underfloor heating controller powered by Home Assistant, which enables you to use the temperature sensors and wall plugs (for the actuators on the underfloor heating manifold, i.e. the devices responsible for opening and closing each water circut) you want.


# Why do I need a smart heating system in the first place?
In short: You need a smart heating system in order to have room profiling and weather compensation. Traditional "dumb" heating components react to the current conditions, e.g. turn "off" when the desired temperature is reached or "on" when it's already too cold, which means that they're always lagging behind what is actually needed _right now_. Smart heating systems on the other hand create (or should, at least) a heating profile for each room so that the system knows how long adjustments take to have an effect, and can thus proactively adjust the temperature to better maintain the desired level. In addition, systems which offer wheather compensation can further improve upon the indoor climate by, e.g. turning "off" the heating at the correct time based on rising temperatures in the afternoon, etc.


# Purpose and components
The purpose of the project is to create a smart unified (locally running) heating system which includes both radiators and underfloor heating, and which can work with the (my) current underfloor heating manifold.

> My outset for this project is my two-storey house which has radiators on the first floor, and water-based underfloor heating on the ground floor (and in the first floor bathroom) - totalling 4 radiators and 5 underfloor heating zones (water circuts) in all:
  >
  >  - Initially the radiator thermostats were "dumb" Danfoss thermostats, which I quickly replaced with [Tado Smart Radiator Thermostats](https://www.tado.com/dk-en/smart-radiator-thermostat-add-on).
  >  - The underfloor heating manifold and actuators are from Pettinaroli, the controller for the actuators and circulation pump is a no-name 868 MHz thingy, "Funk B 2070-2" (which appears to be used by Altech, Pettinaroli and others), with "dumb" [Altech](https://www.altechcorp.com) wireless (and displayless) thermostats.

In the case of this particular project, the components are:

1. [Home Assistant](https://www.home-assistant.io/) running on a Raspberry Pi for handling automations.

1. [Wireless Temperature Sensors](https://www.tado.com/dk-en/wireless-temperature-sensor) by Tado for measuring the temperature.

1. [Waveshare 8-channel Ethernet relay module (B) with digital inputs](https://www.waveshare.com/modbus-poe-eth-relay-b.htm.htm) (hereafter referred to as the "control-unit") for controlling the actuators and circulation pump. I have specifically chosen this version as it support "power over Ethernet" (PoE), which means that although each circut needs its own power-supply, the relay itself can be powered and controlled via an Ethernet-cable, thus eliminating the need for a dedicated powersupply for the control-unit.

    > If you don't have a switch which supports PoE, you can either acquire a PoE-adapter to put between your switch and the control-unit, or you can acquire a `2.1 mm` DC power-adapter in the 7-36 Volt range, e.g. `12V 27W 2.25A`, as the control-unit supports a wide range of options.
    
    > Of ALL the *other* control-units I have researched only a few support **[Modbus TCP](https://en.wikipedia.org/wiki/Modbus)** and they all do so over WiFi, which naturally entails, that if changes to the wireless network occur, e.g. the network name or password, your underfloor heating stops working. Hence, my preference for wired Ethernet.

    > Note that the first of these control-units I bought (from a reseller in another country, hence no return) was defective: It was clarly possible for Home Assistant (VirCom etc., see below) to communicate with it, but the communication didn't change the state of the relays. Hence, I bought a couple more relays directly from [Waveshare](https://www.waveshare.com) and they work as intended.

1. Actuators which fit your underfloor heating manifold and which can be powered by the wall plugs. Note that your actuators need to be 230V so that they can be plugged into the wall plugs. 12V actuators will require an adaptor.

... but you can probably make it work with most vendors (for items 2 & 3). What you essentially need is:

1. One "wireless temperature sensor" (with Tado you can actually add additional devices which can provide better readings for each room/underfloor heating zone).
1. One controllable relay.
1. One "actuator".

... for each underfloor heating zone.


# Configuring the control-unit
First and foremost, the control-unit needs to be connected to your network in order for you to configure it. Hence, plug an Ethernet-cable into the control-unit's Ethernet-port and ensure that the device is powered on.

By default, Waveshare's control-unit comes preconfigured with its IP-address set to `192.168.1.200` and to change this there are a couple of options:

## Modify control-unit using built-in web-server
1. Launch a web-browser and go to: http://192.168.1.200.
  
    > Notice that the address is non-HTTPS, so it's possible that you'll need to try out some different web-browsers in order to find one which will let you access the site, e.g. Brave doesn't work, but Safari (macOS) did it for me.

1. On the log-in screen, just press "login" (the device doesn't have a password).

1. Under "network settings", set:
    1. Device port: `502` (standard Modbus-port)
    1. IP mode: `DHCP`

        > You can obviously configure the network-settings as needed. In my case, I use my router to assign a predetermined IP-address for the control-unit instead of hard-coding it into the device itself.

    1. Click "submit".
    1. Click "restart" to restart the device.

... and you're done.

## Modify control-unit using "VirCom"
If modification via the built-in web-server doesn't work for you, you can head to [control unit's support page](https://www.waveshare.com/wiki/Modbus_POE_ETH_Relay_(B)) and download the [VirCom configuration software](https://files.waveshare.com/upload/4/42/VirCom_en.rar) for Microsoft Windows. Once you've got the software unpacked, run it and:

1. From the main screen, click "device". This will open up the "device management" dialog and locate the device on your network.

    > If it doesn't, e.g. if you're in another IP-subnet, you can click "add manually", and specify the "start IP" and "end IP" in order to widen the search-scope, after which you can click "auto search" to initiate a new search.

1. When the device has been located, double-click on the row for the item to open up "device settings". From here, set:

    1. IP mode: `DHCP`

        > As above, you should configure the network-settings as needed.

    1. Port: `502`

    1. Click "modify setting" to save the changes (I think)

        > It's possible that you also need to click "restart dev(ice)" in order for the changes to take effect.

... and you're done.


# Add devices to Home Assistant
## Adding Modbus support to Home Assistant
To enable Modbus support in Home Assistant, you need to edit Home Assistant's `configuration.yaml` file, and to do this, you first need the [File editor](https://www.home-assistant.io/common-tasks/supervised/#installing-and-using-the-file-editor-add-on) add-on.

### Installing the file editor add-on
In Home Assistant:

1. Go to: "Settings > Add-ons".

1. Click "Add-on store".

1. Search for `file editor` to easily locate the item, and click it.

1. Click "install".

> You may need to restart Home Assistant after the add-on completes installing.

After the add-on completes installing, you can modify the component to "show in sidebar" so that you can easily locate it.

### Modifying Home Assistant's configuration file
In Home Assistant:

1. Launch the "File editor" (if needed, you will be prompted to start the add-on).
1. Click the folder-icon in the top-left corner.
1. Click `configuration.yaml` to open the configuration file.
1. Paste the following into the configuration file:


```
# Configure Modbus TCP-connection to Waveshare control-unit 
modbus:
  - name: waveshare_relay
    type: rtuovertcp
    host: {IP-address of relay}
    port: 502

    # Switches
    ## Relays
    switches:
    - name: Relay 1
      unique_id: ws_r1
      slave: 1
      address: 0
      write_type: coil
    - name: Relay 2
      unique_id: ws_r2    
      slave: 1
      address: 1
      write_type: coil
    - name: Relay 3
      unique_id: ws_r3
      slave: 1
      address: 2
      write_type: coil
    - name: Relay 4
      unique_id: ws_r4
      slave: 1
      address: 3
      write_type: coil
    - name: Relay 5
      unique_id: ws_r5
      slave: 1
      address: 4
      write_type: coil
    - name: Relay 6
      unique_id: ws_r6
      slave: 1
      address: 5
      write_type: coil
    - name: Relay 7
      unique_id: ws_r7
      slave: 1
      address: 6
      write_type: coil
    - name: Relay 8
      unique_id: ws_r8
      slave: 1
      address: 7
      write_type: coil
    - name: Relay (toggle all)
      slave: 1
      address: 255
      write_type: coil 
    - name: Relay (on/off flip)
      slave: 1
      address: 511
      write_type: coil

    ## Flash: On
    - name: Relay 1 flash on
      slave: 1
      address: 512
      write_type: coil
    - name: Relay 2 flash on
      slave: 1
      address: 513
      write_type: coil 
    - name: Relay 3 flash on
      slave: 1
      address: 514
      write_type: coil 
    - name: Relay 4 flash on
      slave: 1
      address: 515
      write_type: coil 
    - name: Relay 5 flash on
      slave: 1
      address: 516
      write_type: coil 
    - name: Relay 6 flash on
      slave: 1
      address: 517
      write_type: coil 
    - name: Relay 7 flash on
      slave: 1
      address: 518
      write_type: coil 
    - name: Relay 8 flash on
      slave: 1
      address: 519
      write_type: coil 

    ## Flash: Off
    - name: Relay 1 flash off
      slave: 1
      address: 1024
      write_type: coil 
    - name: Relay 2 flash off
      slave: 1
      address: 1025
      write_type: coil 
    - name: Relay 3 flash off
      slave: 1
      address: 1026
      write_type: coil
    - name: Relay 4 flash off
      slave: 1
      address: 1027
      write_type: coil 
    - name: Relay 5 flash off
      slave: 1
      address: 1028
      write_type: coil
    - name: Relay 6 flash off
      slave: 1
      address: 1029
      write_type: coil 
    - name: Relay 7 flash off
      slave: 1
      address: 1030
      write_type: coil 
    - name: Relay 8 flash off
      slave: 1
      address: 1031
      write_type: coil

    # Sensors
    ## States
    binary_sensors:
      - name: Relay 1 status
        address: 0             
        input_type: coil   
        slave: 1  
        scan_interval: 1
      - name: Relay 2 status
        address: 1
        input_type: coil
        slave: 1
        scan_interval: 1
      - name: Relay 3 status
        address: 2             
        input_type: coil   
        slave: 1  
        scan_interval: 1
      - name: Relay 4 status
        address: 3
        input_type: coil
        slave: 1
        scan_interval: 1
      - name: Relay 5 status
        address: 4            
        input_type: coil   
        slave: 1  
        scan_interval: 1
      - name: Relay 6 status
        address: 5
        input_type: coil
        slave: 1
        scan_interval: 1
      - name: Relay 7 status
        address: 6             
        input_type: coil   
        slave: 1  
        scan_interval: 1
      - name: Relay 8 status
        address: 7
        input_type: coil
        slave: 1
        scan_interval: 1

    ## Inputs
      - name: Relay input 1
        address: 0
        input_type: discrete_input
        slave: 1
        scan_interval: 1       
      - name: Relay input 2 
        address: 1
        input_type: discrete_input
        slave: 1
        scan_interval: 1
      - name: Relay input 3
        address: 2
        input_type: discrete_input
        slave: 1
        scan_interval: 1       
      - name: Relay input 4
        address: 3
        input_type: discrete_input
        slave: 1
        scan_interval: 1
      - name: Relay input 5
        address: 4
        input_type: discrete_input
        slave: 1
        scan_interval: 1       
      - name: Relay input 6  
        address: 5
        input_type: discrete_input
        slave: 1
        scan_interval: 1
      - name: Relay input 7
        address: 6
        input_type: discrete_input
        slave: 1
        scan_interval: 1       
      - name: Relay input 8
        address: 7
        input_type: discrete_input
        slave: 1
        scan_interval: 1
```

1. Restart Home Assistant to apply the changes
  
  > You can restart Home Assistant directly from the "File editor": Click the gear/cog-icon and choose "Restart Home Assistant" from the context-menu.
  >
  > If you only want to reload the "Modbus" module, you can navigate to: "Developer tools > YAML" and click "MODBUS".

The Modbus-integration should now be visible in Home Assistant under: "Settings > Devices & services", along with its entities.

## Adding Tado wireless temperature sensors to Home Assistant
Set the Tado bridge into pairing mode and add it to Home Assistant. You may need to "reset" the Tado bridge, after which it will automatically show up in Home Assistant. The Tado bridge will also be grouped with other Apple HomeKit devices in the "HomeKit Device" group. When the bridge has been added to Home Assistant, all of the connected Tado devices will be available.

> In order to see the Tado devices in Apple's Home app again, you can add a "HomeKit Bridge" to Home Assistant, and choose which "entities" to share. Upon completing the configuration, the selected entities show up as normally in Apple's Home app.

### Regarding Tado, Home Assistant and Apple HomeKit
Home Assistant has a built-in plug-in for Tado which controls the said devices via Tado's public API which is available via the Internet, and which has a limited polling-rate which defaults to updating every 5 minutes. By pairing Tado directly with Home Assistant via Home Assistant's "HomeKit Device", updates from Tado devices are registered instantly in Home Assistant without any external dependencies.


# Home Assistant scripts
You need to create some scripts in Home Assistant which can be invoked by automations later on.

> The script section is located in Home Assistant udner: "Settings > automations & scenes > scripts".

> In my case, I use relays 1-5 for actuators, and have the circulation pump attached to **"Relay 8"**.


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
          states('binary_sensor.relay_1_status') in ['off', 'unavailable', 'unknown']
      and states('binary_sensor.relay_2_status') in ['off', 'unavailable', 'unknown']
      and states('binary_sensor.relay_3_status') in ['off', 'unavailable', 'unknown']
      and states('binary_sensor.relay_4_status') in ['off', 'unavailable', 'unknown']
      and states('binary_sensor.relay_5_status') in ['off', 'unavailable', 'unknown']
    }}
    ```

1. Add action: `Switch: Turn off`
    1. Choose entity: `Relay 8`


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
         is_state('binary_sensor.relay_1_status', 'on')
      or is_state('binary_sensor.relay_2_status', 'on')
      or is_state('binary_sensor.relay_3_status', 'on')
      or is_state('binary_sensor.relay_4_status', 'on')
      or is_state('binary_sensor.relay_5_status', 'on')
    }}
    ```

1. Add action: `Switch: Turn on`
    1. Choose entity: `Relay 8`


# Home Assistant automations for circulation pump

## Zone circulation pump off (script)
The purpose of this script is to turn off the circulation pump when <u>all</u> water circuts are closed (actuators are inactive).

Triggers:
1. Add trigger: `Template`
1. Value template:

```
{{
      states('binary_sensor.relay_1_status') in ['off', 'unavailable', 'unknown']
  and states('binary_sensor.relay_2_status') in ['off', 'unavailable', 'unknown']
  and states('binary_sensor.relay_3_status') in ['off', 'unavailable', 'unknown']
  and states('binary_sensor.relay_4_status') in ['off', 'unavailable', 'unknown']
  and states('binary_sensor.relay_5_status') in ['off', 'unavailable', 'unknown']
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
     is_state('binary_sensor.relay_1_status', 'on')
  or is_state('binary_sensor.relay_2_status', 'on')
  or is_state('binary_sensor.relay_3_status', 'on')
  or is_state('binary_sensor.relay_4_status', 'on')
  or is_state('binary_sensor.relay_5_status', 'on')
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
1. Add action: `Switch: Turn off`
2. Choose entity: {select the corresponding relay switch}


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
1. Add action: `Switch: Turn on`
2. Choose entity: {select the corresponding relay switch}


## Zone {number} start-up
The purpose of this automation is to register when a said actuator (wall plug) becomes available to Home Assistant, and turn on if the corresponding thermostat is requesting heat.

Triggers:
1. Add trigger: `State`
1. Entity: {select the corresponding relay switch}
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
1. Add action: `Switch: Turn on`
    1. Choose entity: {select the corresponding relay switch}


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
