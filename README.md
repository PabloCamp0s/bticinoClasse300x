# BTicino C100X/C300X modified firmware

## Firmware preparation and flashing

### 1. Prepare and create firmware via python script

[![asciicast](https://asciinema.org/a/514007.svg)](https://asciinema.org/a/514007)

(Using GNU/Linux):

```bash
git clone https://github.com/fquinto/bticinoClasse300x.git
cd bticinoClasse300x
sudo python3 -m pip install --upgrade pip
sudo python3 -m pip install -r requirements.txt
sudo python3 main.py
```

(Using Docker):

```bash
git clone https://github.com/fquinto/bticinoClasse300x.git
cd bticinoClasse300x
docker compose run bticino
```

### 2. Flash firmware using MyHomeSuite

- Download and install "My Home Suite" configuration software (Windows only) from [the official page](https://www.homesystems-legrandgroup.com/en/home/-/productsheets/2493426)

- Unmount the intercom from the wall and, while still connected to the SCS 2-wire bus, connect to the intercom via the Mini-USB port located on the back. See the following technical sheets for USB port location: [C100X](https://www.homesystems-legrandgroup.com/MatrixENG/liferay/bt_mxLiferayCheckout.jsp?fileFormat=generic&fileName=ST-00000695-EN.pdf&fileId=58107.23188.48156.49690) - [C300X](https://www.homesystems-legrandgroup.com/MatrixENG/liferay/bt_mxLiferayCheckout.jsp?fileFormat=generic&fileName=ST-00000361-EN.pdf&fileId=58107.23188.48057.62932)

- Follow the steps shown in the following animation to select your specific device model and flash it with firmware generated by the script in step 1
![](/myhome_flash.gif)

---

## Device integration capabilities

**NOTE**: <intercom_ip> is the IP address of the video intercom door entry unit.

### 1. Via any generic MQTT broker

To better understand MQTT implementation, have a look [here](/mqtt_scripts/README.md)

Once you flashed the new firmware, establish a connection with your intercom via SSH

```sh
ssh root2@<intercom_ip>
```

If you're using a mac (OSx)

```sh
# First create a RSA key if you never done before
ssh-keygen -t rsa

# Do the connection
ssh -oHostKeyAlgorithms=+ssh-rsa root2@<intercom_ip>
```

Proceed with all the following

```sh
# Move to the folder
cd /etc/tcpdump2mqtt

# Make the filesystem writable.
mount -oremount, rw /  

# Modify the config file with your MQTT parameters (server, user and password)
vi TcpDump2Mqtt.conf 

# Make the filesystem read-only again.
mount -oremount, ro /

# Restart the video door entry unit.
reboot    
```

Our intercom is now sending / receiving any commands to the MQTT broker.

### 2. Homeassisant via an MQTT broker

**NOTE**: In order to manage MQTT topics in Homeassistant it is necessary to have the MQTT integration installed and have followed the instructions in the previous step

* Basic configuration

    In the Homeassistant **configuration.yaml** file, in the **mqtt:** block it is necessary to insert the following lines to instruct MQTT to receive / transmit on the topics we have defined in the firmware creation script:

    ```yaml
    mqtt:
      sensor:
        - unique_id: '14532784978700'
          name: "Video intercom TX"
          state_topic: "Bticino/tx"
          availability_topic: "Bticino/LastWillT"
          icon: mdi:phone-outgoing

        - unique_id: '13454564689485'
          name: "Video intercom RX"
          state_topic: "Bticino/rx"
          availability_topic: "Bticino/LastWillT"
          icon: mdi:phone-incoming
    ```

* Automations

    We need to create automations that allow us to interact with the video door entry unit.

    - Open the door
    
        The following automation creates a button that allows the gate to be opened and creates a notification in the Homeassistant notification area.

        ```yaml
            - id: '1656918057723'
              alias: Apertura Cancelletto Pedonale
              description: ''
              trigger:
              - platform: state
                entity_id:
                - input_button.cancelletto_pedonale
              condition: []
              action:
              - service: notify.persistent_notification
                data:
                  message: Il cancello pedonale è aperto
              - service: mqtt.publish
                data:
                  topic: Bticino/rx
                  payload: '*8*19*20##'
              - delay:
                  hours: 0
                  minutes: 0
                  seconds: 1
                  milliseconds: 0
              - service: mqtt.publish
                data:
                  payload: '*8*20*20##'
                  topic: Bticino/rx
              mode: single
        ```

    - Recognize the commands
    
        The following automation recognizes some commands received from the video door entry unit and notifies the event. Obviously the notification scripts shown in the automation will have to be replaced with the one you want.

        ```yaml
            - id: '1657896199804'
              alias: Notifiche dal citofono
              description: ''
              trigger:
              - platform: state
                entity_id:
                - sensor.video_intercom_tx
              action:
              - choose:
                - conditions:
                  - condition: state
                    entity_id: sensor.video_intercom_tx
                    state: '*8*21*10##'
                  sequence:
                  - service: script.notifica_voce_evento
                    data:
                      notification_message: "La luce scala è stata attivata"
                  - service: script.notifica_testo_evento
                    data:
                      notification_message: "La luce scala è stata attivata"
                - conditions:
                  - condition: state
                    entity_id: sensor.video_intercom_tx
                    state: '*8*19*20##'
                  sequence:
                  - service: script.notifica_voce_evento
                    data:
                      notification_message: "Il cancelletto è stato aperto"
                  - service: script.notifica_testo_evento
                    data:
                      notification_message: "Il cancelletto è stato aperto"
                default: []
              mode: single
        ```

### 3. Via remote shell scripting

- Example one:

    ```sh
    #!/usr/bin/expect -f
    spawn ssh <intercom_ip>
    expect "assword:"
    send "pwned123\r"
    expect "root@C3X-00-03-50-xx-xx-xx-yyyyyyy:~#"
    send "echo *8*19*20## |nc 0 30006\r"
    send "sleep 1\r"
    send "echo *8*20*20##|nc 0 30006\r"
    send "exit\r"
    interact
    ```

- Example two:

    ```sh
    #!/bin/bash
    sshpass -p pwned123 ssh -o StrictHostKeyChecking=no <intercom_ip> "echo *8*19*20## |nc 0 30006; sleep 1; echo *8*20*20##|nc 0 30006"
    ```

- Example 3 (direct test):

    ```sh
    ssh <intercom_ip> 'echo *8*19*20## | nc 0 30006; sleep 1; echo *8*20*20## | nc 0 30006'
    ```

### Home Assistant via shell scripts

Depending on your configuration, you may need to create a script to write commands in HA.

In next example are both. You can use only one (you don't need both if one it's running ok).

- Add your script in `shell_components` folder

- Configure in HA:
  ```yaml
  shell_command:
    openbuildingdoor: "/home/homeassistant/.homeassistant/shell_commands/openbuildingdoor.sh"
    openbuildingdoor2: "ssh bticino 'echo *8*19*20## |nc 0 30006; sleep 1; echo *8*20*20##|nc 0 30006'"
  ```

- In your `ui-lovelace.yaml`:
  ```yaml
  cards:
    - type: button
        name: Open building door CMD1
        show_state: false
        show_name: true
        show_icon: true
        tap_action:
          action: call-service
          service: shell_command.openbuildingdoor
    - type: button
        name: Open building door CMD2
        show_state: false
        show_name: true
        show_icon: true
        tap_action:
          action: call-service
          service: shell_command.openbuildingdoor2
  ```

## OpenWebNet commands

### Known commands / notifications

| Command | Description |
| --- | --- |
| \*8\*19\*20## | main door opening button pressed |
| \*8\*20\*20## | main door opening button released |
| \*8\*19\*21## | secondary door opening button pressed |
| \*8\*20\*21## | secondary door opening button released |
| \*8\*21\*10## | stairs light activated |
| \*8\*1#5#4#20\*10## | intercom camera on |
| \*8\*3#5#4\*420##  | intercom camera off |
| \*8\*1#1#4#21\*10## | incoming external call |
| \*8\*1#1#4#21\*11## | incoming external call |
| \*8\*1#1#4#21\*16## | incoming external call |
| \*7\*59#12#0#0\*## | incoming internal call |

### Explore new commands

You can discover new command in the following ways

- Via MQTT sniffing: use any MQTT listening app / HA integrated MQTT topic listener

- Via TCP sniffing:
  * Go to your home: `cd ~`
  * On your Linux computer type: 
  `ssh root2@192.168.1.97 '/usr/sbin/tcpdump -i lo -U -w - "not port 22"' > recordingsFILE`
  * Use the App on your mobile and open the door or whatever you want to "learn"
  * Stop data recording with CTRL+C
  * Open recordingsFILE: `wireshark ~/recordingsFILE`

---

## Telegram Channel from which all of this originated

https://t.me/bTicinoClasse300x
