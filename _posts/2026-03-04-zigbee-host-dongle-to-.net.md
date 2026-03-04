---
title: Zigbee host dongle to .NET
date: 2026-03-04T17:52:00
---

ca

 Here's the straightforward path:

**The stack you want**

```plain
Zigbee dongle → Zigbee2MQTT → MQTT broker (Mosquitto) → your .NET 8 app
```

**Step 1: Install Mosquitto (MQTT broker)**

bash

```plain
sudo apt install mosquitto mosquitto-clients
sudo systemctl enable mosquitto
sudo systemctl start mosquitto
```

**Step 2: Install Zigbee2MQTT**

bash

```plain
# Install Node.js first if needed
sudo apt install nodejs npm

# Create directory and install
sudo mkdir /opt/zigbee2mqtt
sudo chown -R $USER /opt/zigbee2mqtt
git clone --depth 1 https://github.com/Koenkk/zigbee2mqtt.git /opt/zigbee2mqtt
cd /opt/zigbee2mqtt
npm ci
```

**Step 3: Find your dongle's port**

bash

```plain
ls /dev/serial/by-id/
```

It'll show something like `usb-ITead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_...`

**Step 4: Configure Zigbee2MQTT**

Edit `/opt/zigbee2mqtt/data/configuration.yaml`:

yaml

```plain
serial:
  port: /dev/ttyUSB0  # use your actual port from step 3

mqtt:
  base_topic: zigbee2mqtt
  server: mqtt://localhost

frontend:
  enabled: true  # gives you a web UI on port 8080
```

**Step 5: In your .NET 8 app**

Add the `MQTTnet` NuGet package and subscribe to topics:

csharp

```plain
var factory = new MqttFactory();
var client = factory.CreateMqttClient();

var options = new MqttClientOptionsBuilder()
    .WithTcpServer("localhost", 1883)
    .Build();

await client.ConnectAsync(options);

client.ApplicationMessageReceivedAsync += e =>
{
    var topic = e.ApplicationMessage.Topic;
    var payload = e.ApplicationMessage.ConvertPayloadToString();
    Console.WriteLine($"{topic}: {payload}");
    return Task.CompletedTask;
};

await client.SubscribeAsync("zigbee2mqtt/#");
```

Zigbee2MQTT publishes device data as JSON to topics like `zigbee2mqtt/your-device-name`.




. To get a device list, publish to the bridge topic:

**From the command line:**

bash

```plain
mosquitto_pub -t "zigbee2mqtt/bridge/request/devices" -m ""
```

Then listen for the response:

bash

```plain
mosquitto_sub -t "zigbee2mqtt/bridge/response/devices"
```

**From your .NET app:**

csharp

````plain
// Subscribe to the response first
await client.SubscribeAsync("zigbee2mqtt/bridge/response/devices");

// Then request the device list
var message = new MqttApplicationMessageBuilder()
    .WithTopic("zigbee2mqtt/bridge/request/devices")
    .WithPayload("")
    .Build();

await client.PublishAsync(message);
```

The response comes back as JSON with each device's IEEE address, friendly name, model, manufacturer, and whether it's interview complete.

**Alternatively**, Zigbee2MQTT also publishes the full device list automatically on startup to:
```
zigbee2mqtt/bridge/devices
````

So you can just subscribe to that topic and you'll get it whenever Zigbee2MQTT starts or restarts — no need to request it.

If you enabled the frontend in your config, you can also just visit `http://localhost:8080` to see all devices visually while you're developing.

[Claude is AI and can make mistakes. Please double-check responses.](https://support.anthropic.com/en/articles/8525154-claude-is-providing-incorrect-or-misleading-responses-what-s-going-on)
