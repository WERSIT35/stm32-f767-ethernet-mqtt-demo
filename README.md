# STM32F767 Ethernet MQTT Demo

A working STM32F767 Ethernet + MQTT demo using **lwIP** and a local **Mosquitto MQTT broker**.

The STM32 connects to a Mosquitto broker over Ethernet, publishes test messages, subscribes to a command topic, and controls an onboard LED through MQTT commands.

This project is a small but important milestone for a larger smart-building controller system.

---

## Features

- STM32F767 Ethernet communication
- lwIP TCP/IP stack
- MQTT client using lwIP MQTT API
- Local Mosquitto broker support
- STM32 → broker MQTT publishing
- Broker/PC → STM32 MQTT command receiving
- LED control over Ethernet using MQTT
- UART debug logs
- Basic lwIP tuning for MQTT stability

---

## Hardware Used

- STM32F767 / Nucleo-F767ZI board
- Ethernet cable
- PC running Mosquitto broker
- USB cable or external 5V power supply
- LAN/router connection

> UART is used only for debug logs. MQTT commands are received through Ethernet.

---

## System Flow

```text
PC / Mosquitto Broker
        ↓
   Ethernet / LAN
        ↓
STM32F767 + lwIP MQTT Client
        ↓
GPIO LED Control
```

Command example:

```text
PC sends MQTT command: LED_ON
STM32 receives command over Ethernet
STM32 turns LED ON
```

---

## MQTT Topics

### STM32 publishes to:

```text
home/stm32/test
```

Example payload:

```text
hello from STM32
```

### STM32 subscribes to:

```text
home/stm32/cmd
```

Supported command payloads:

```text
LED_ON
LED_OFF
```

---

## Mosquitto Setup

Install Mosquitto on Windows.

Example local development `mosquitto.conf`:

```conf
listener 1883 0.0.0.0
allow_anonymous true
```

Start Mosquitto in verbose mode:

```cmd
cd "C:\Program Files\Mosquitto"
mosquitto -c mosquitto.conf -v
```

Expected broker log:

```text
mosquitto version 2.1.2 running
Opening ipv4 listen socket on port 1883
```

---

## Windows Firewall

If STM32 cannot connect to Mosquitto, allow port `1883`:

```cmd
netsh advfirewall firewall add rule name="Mosquitto MQTT 1883" dir=in action=allow protocol=TCP localport=1883
```

---

## Finding Your PC Broker IP

On Windows:

```cmd
ipconfig
```

Find your IPv4 address, for example:

```text
IPv4 Address . . . . . . . . . . : 192.168.1.251
```

Then update the broker IP in STM32 code:

```c
IP4_ADDR(&broker_ip, 192, 168, 1, 251);
```

---

## STM32 MQTT Connection Code

Main MQTT start function:

```c
static void mqtt_start(void)
{
    IP4_ADDR(&broker_ip, 192, 168, 1, 251); // PC Mosquitto broker IP

    mqtt_client = mqtt_client_new();

    if (mqtt_client == NULL)
    {
        printf("mqtt_client_new failed\r\n");
        return;
    }

    struct mqtt_connect_client_info_t ci;
    memset(&ci, 0, sizeof(ci));

    ci.client_id = "stm32_f767";
    ci.keep_alive = 60;

    printf("Connecting MQTT...\r\n");

    err_t err = mqtt_client_connect(mqtt_client,
                                    &broker_ip,
                                    1883,
                                    mqtt_connection_cb,
                                    NULL,
                                    &ci);

    printf("mqtt_client_connect returned: %d\r\n", err);

    if (err != ERR_OK)
    {
        printf("MQTT connect start failed\r\n");

        mqtt_client_free(mqtt_client);
        mqtt_client = NULL;
    }
}
```

---

## Important lwIP Configuration

In `lwipopts.h`, increase the timeout pool:

```c
#define MEMP_NUM_SYS_TIMEOUT 40
```

This prevents this lwIP assertion:

```text
Assertion "sys_timeout: timeout != NULL, pool MEMP_SYS_TIMEOUT is empty"
```

Recommended values:

```c
#define MEMP_NUM_SYS_TIMEOUT 40
#define MEMP_NUM_TCP_PCB 10
#define MEMP_NUM_TCP_SEG 32
#define TCP_SND_QUEUELEN 16
```

---

## Running a Subscriber on PC

To see messages published by STM32:

```cmd
cd "C:\Program Files\Mosquitto"
mosquitto_sub -h localhost -t home/stm32/test -v
```

Expected output:

```text
home/stm32/test hello from STM32
```

---

## Sending Commands to STM32

Turn LED on:

```cmd
mosquitto_pub -h localhost -t home/stm32/cmd -m LED_ON
```

Turn LED off:

```cmd
mosquitto_pub -h localhost -t home/stm32/cmd -m LED_OFF
```

---

## Expected STM32 UART Log

Successful boot, MQTT connect, subscribe, and publish:

```text
BOOT OK
AFTER LWIP
IP ready, starting MQTT
Connecting MQTT...
mqtt_client_connect returned: 0
MQTT connected
mqtt_subscribe returned: 0
Subscribe result: 0
Publish result: 0
Publish result: 0
```

Successful command receive:

```text
Incoming topic: home/stm32/cmd, len: 6
MQTT RX: LED_ON
LED ON
```

---

## Expected Mosquitto Log

```text
New connection from 192.168.1.68 on port 1883.
New client connected from 192.168.1.68 as stm32_f767.
Received PUBLISH from stm32_f767 'home/stm32/test'
```

---

## Power Notes

For this prototype:

```text
Ethernet cable = network/data only
USB or external 5V = STM32 power
```

Do not inject random 5V into Ethernet wires.

For a future production system, use either:

- Proper PoE hardware, or
- Separate 24V power distribution with local buck converters

---

## Current Status

Working:

- Ethernet link
- IP address assignment
- MQTT connection
- MQTT publish
- MQTT subscribe
- PC command to STM32
- LED control over Ethernet

---

## Future Improvements

- Add reconnect logic
- Add JSON command payloads
- Add relay control
- Add sensor publishing
- Add temperature/humidity sensor support
- Add RS485 node communication
- Add MQTT status topic
- Add cloud MQTT broker support
- Add MQTT over TLS
- Add watchdog and fault recovery

---

## Example Future Topic Structure

```text
home/main/status
home/main/sensors
home/main/cmd
home/main/relay/1
home/main/relay/2
home/main/alarm
```

---

## Project Goal

This project proves the base communication layer for a smart building controller:

```text
STM32 main controller
→ Ethernet
→ MQTT broker
→ remote commands
→ hardware control
```

It can now be extended into a real smart-building system with sensors, relays, RS485 nodes, and cloud communication.
## License

My application code is MIT licensed, see [LICENSE](LICENSE). The vendor code (STM32 HAL/CMSIS, lwIP and the middleware under `Drivers/` and `Middlewares/`) keeps its original licenses.
