# dashcast-docker-MQTT
Persistent Chromecast dashboards with dashcast and pychromecast in a docker container with the URL updated via MQTT.

Credit to [madmod/dashcast-docker](https://github.com/madmod/dashcast-docker), used as a base for this docker. Credit to [mukowman/MQTT-DashCast-Docker](https://github.com/mukowman/MQTT-DashCast-Docker) for inspiration & portions of MQTT code.

My original goal was to replace the [Backdrop](https://www.google.com/chromecast/backdrop/) on my [Chromecast](https://store.google.com/product/chromecast_2015) with an [AppDaemon](http://appdaemon.readthedocs.io/) [Dashboard](http://appdaemon.readthedocs.io/en/latest/DASHBOARD_CREATION.html). I do not want to interfere with operation of the Chromecast in any other way.

This docker will launch DashCast on the specified Chromecast, then subscribe to an MQTT topic for a new URL to display.

If another application takes over the Chromecast, DashCast will not launch until the other application has ended and Backdrop is active again, or a valid MQTT message was received.

## Update dashboard url over MQTT

`DISPLAY_NAME` is the name of the Chromecast you wish to display dashboards on. Multiple Chromecasts will require multiple instances of this container.

This container will subscribe to the following MQTT topic

	chromecast/DISPLAY_NAME/command/dashcast

To change the URL, publish a json array to `chromecast/DISPLAY_NAME/command/dashcast`

`{"url":"http://192.168.0.10:5050/tv_doorbell?skin=raging","force":false,"takeover":true}`

Only 'url' is required.

The optional 'force' boolean is used for some sites that forbid loading using the default method. Default is False. Use this option to force them to load.  Doing so makes DashCast loose control of the Chromecast.

The optional 'takeover' boolean is used to take over from another casting application. Default is False. If this is unset or False, the dashboard url is updated. If DashCast is displaying, it will update immediately. If another casting application is active, the new dashboard url will wait to be used when the Chromecast is idle.

## Docker Image

Network has to be 'host' for Chromecast discovery to work. 'depends_on' for docker-compose is helpful for avoiding an error when rebooting your server. This can be omitted if you're displaying dashboards from another source.

#### Required Environment Variables:
* DEFAULT_DASHBOARD_URL = URL to be shown on the Chromecast after starting this container
* DISPLAY_NAME = Name of the Chromecast to control
* MQTT_SERVER = Address of MQTT server

#### Optional Environment Variables:
* IGNORE_CEC = default True
* DEFAULT_DASHBOARD_URL_FORCE = default False
* MQTT_USERNAME = username for MQTT server. Not required if no server auth
* MQTT_PASSWORD = password for MQTT server

### docker run

```console
$ docker run -d \
      --name DashCast \
      --restart=always \
      --network="host" \
      -e DEFAULT_DASHBOARD_URL="http://192.168.0.10:5050/tv?skin=raging" \
      -e DEFAULT_DASHBOARD_URL_FORCE="False" \
      -e DISPLAY_NAME="ChromecastName" \
      -e IGNORE_CEC="True" \
      -e MQTT_SERVER="192.168.0.10" \
      -e MQTT_USERNAME="user" \
      -e MQTT_PASSWORD="password" \
      ragingcomputer/dashcast-docker-mqtt
```

### docker-compose

```yaml
version: '3'

services:

  dashcast:
    image: ragingcomputer/dashcast-docker-mqtt
    container_name: "DashCast"
    restart: always
    depends_on:
      - appdaemon
    network_mode: host
    environment:
      - DEFAULT_DASHBOARD_URL=http://192.168.0.10:5050/tv?skin=raging
      - DEFAULT_DASHBOARD_URL_FORCE=False
      - DISPLAY_NAME=ChromecastName
      - IGNORE_CEC=True
      - MQTT_SERVER=192.168.0.10
      - MQTT_USERNAME=user
      - MQTT_PASSWORD=password
```

# Home Assistant Example

In this example, I'm running AppDaemon on 192.168.0.10 with dashboards on port 5050. I've built dashboards named "tv", "tv_doorbell", "laundry", and "den"

I have an input_select that lists a dash character "-" and my dashboards. I can drop down and select the dashboard to show through DashCast.

There are 2 automations.

dashcast_den_change_url listens for state change of the input_select, if it isn't "-", use a template to build the json array containing the URL of the dashboard to display, and sends the json array over MQTT. Finally, it sets the active item to "-"

dashcast_den_display_doorbell listens for state change of my doorbell, then publishes a MQTT message forcing takeover of the Chromecast with the tv_doorbell dashboard.


#### excerpts from configuration.yaml

```yaml
input_select:
  dashcast_den_dash_select:
    name: DashCast Den Display
    icon: mdi:tablet
    initial: "-"
    options:
    - "-"
    - tv
    - tv_doorbell
    - laundry
    - den

automation:
  - id: dashcast_den_change_url
    alias: DashCast Den Change URL
    trigger:
      platform: state
      entity_id: input_select.dashcast_den_dash_select
    condition:
      condition: template
      value_template: '{{ not is_state("input_select.dashcast_den_dash_select", "-") }}'
    action:
    - service: mqtt.publish
      data:
        topic: 'chromecast/ChromecastName/command/dashcast'
        payload_template: '{"url":"http://192.168.0.10:5050/{{ states("input_select.dashcast_den_dash_select") }}?skin=raging"}'
    - service: input_select.select_option
      data:
        entity_id: input_select.dashcast_den_dash_select
        option: "-"

  - id: dashcast_den_display_doorbell
    alias: display doorbell camera on DashCast in den when doorbell activated
    trigger:
      platform: state
      entity_id: binary_sensor.door_bell
      to: 'on'
    action:
    - service: mqtt.publish
      data:
        topic: 'chromecast/ChromecastDen/command/dashcast'
        payload_template: '{"url":"http://192.168.0.10:5050/tv_doorbell?skin=raging","takeover":true}'
```
