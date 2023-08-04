% Frigate NVR
% Robert Ward<robert@rtward.com>
%![](static/qrcode.png)<br/>Talk: [${TALK_URL}](${TALK_URL})<br/>Repo: [${REPO_URL}](${REPO_URL})

# Another NVR?

::: notes

Why do we need another NVR?
What does this do that BlueIris, ZoneMinder, Motion, etc. don't do?

:::

## AI!

::: notes

The primary selling point of Frigate is that it's not just recording based on motion or time, but can record based on when it detects things in your camera's view.

:::

## Hardware Acceleration

::: notes

Unlike some other NVRs that have detection built in, Frigate was built from the ground up to use hardware acceleration to make detections faster.

I'm currently running six cameras on one Google Coral unit, and the single CPU the VM running Frigate has sits as about three percent.

:::

## Home Assistant Integration

::: notes

Frigate has a first class integration with Home Assistant through MQTT.
By listening in on events you can respond to objects detected in different parts of your home.

:::

# Basic Deployment

## Config File

```
birdseye:
  enabled: True
  width: 1920
  height: 1080
  quality: 8
  mode: motion

record:
  enabled: True
  retain:
    days: 14
    mode: motion
  events:
    retain:
      default: 14

snapshots:
  enabled: True
  clean_copy: True
  timestamp: False
  bounding_box: True
  retain:
    default: 30

cameras:
  frontdoor:
    ffmpeg:
      inputs:
        - path: rtsp://wyze:password@192.168.123.124/live
          roles:
            - record
            - detect
            - rtmp

detectors:
  cpu:
    type: cpu

objects:
  track:
    - person
    - cat
    - car
```

## Birdseye

```
birdseye:
  enabled: True
  width: 1920
  height: 1080
  quality: 8
  mode: motion
```

::: notes

Birds eye is a UI piece of Frigate that will show you only cameras with activity
Configurable to trigger on motion or different types of events
e.g. only show me people in the dog kennel, not dogs

:::

## Record

```
record:
  enabled: True
  retain:
    days: 14
    mode: motion
  events:
    retain:
      default: 14
```

::: notes

Record options controls how long Frigate retains videos for.
This configuration retains all events and motion segments for two weeks.
You can override this at a camera level, or for an event type level.

:::

## Snapshots

```
snapshots:
  enabled: True
  clean_copy: True
  timestamp: False
  bounding_box: True
  retain:
    default: 30
```

::: notes

Snapshots are what Frigate will save when it detects something in your videos.
It's a snapshot of what it sees with a box showing you where it detected it
Can be useful for debugging.

:::

## Cameras

```
cameras:
  frontdoor:
    ffmpeg:
      inputs:
        - path: rtsp://wyze:password@192.168.123.124/live
          roles:
            - record
            - detect
            - rtmp
```

::: notes

The meat of the config. Where we define our camera inputs.
Can't get into all of the options here, this is for a simple RTSP camera.
Different cameras can require different options, there are suggestions in the project docs.

:::

## Detectors

```
detectors:
  cpu:
    type: cpu

objects:
  track:
    - person
    - cat
    - car
```

::: notes

These are the options for a CPU detector. You shouldn't use this except for playing around.
It'll consume too much CPU to be usable for real.
The objects arg is a list of the type of objects you want to detect.

:::

## Docker

::: notes

Fortunately / unfortunately docker is the only supported way to deploy Frigate

:::

## Manually Run

```
docker run -d \
  --name frigate \
  --restart=unless-stopped \
  --mount type=tmpfs,target=/tmp/cache,tmpfs-size=1000000000 \
  --device /dev/bus/usb:/dev/bus/usb \
  --device /dev/dri/renderD128 \
  --shm-size=64m \
  -v /path/to/your/storage:/media/frigate \
  -v /path/to/your/config.yml:/config/config.yml \
  -v /etc/localtime:/etc/localtime:ro \
  -e FRIGATE_RTSP_PASSWORD='password' \
  -p 5000:5000 \
  -p 8554:8554 \
  -p 8555:8555/tcp \
  -p 8555:8555/udp \
  ghcr.io/blakeblackshear/frigate:stable
```

::: notes

Don't do this

:::

## Compose File

```
version: "3.9"
services:
  frigate:
    container_name: frigate
    image: ghcr.io/blakeblackshear/frigate:stable
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /srv/frigate/config:/config:ro
      - /media:/media/frigate
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    ports:
      - "5000:5000"
      - "1935:1935"
      - "8554:8554"
```

::: notes

Do this

:::

# UI

## Cameras

![](static/cameras-view.png)

::: notes

Default frigate view
Only shows snapshots, not live

:::

## Birds Eye

![](static/birds-eye-view.png)

::: notes

Shows active cameras, can be configured for what "active" means

:::

## Events

![](static/events-view.png)

::: notes

Shows the full list of events that your cameras have recorded.
Can be filtered by camera, type, etc.

:::

## Recordings

![](static/recordings-view.png)

::: notes

Shows the list of recordings for a specific camera
Will show the events under the segment where their recording is

:::

## Storage

![](static/storage-view.png)

::: notes

Breakdown of how much storage your videos are using

:::

## Debug

![](static/debug-view.png)

::: notes

Will show a live view of your camera, highlighting detected objects

:::

## Debug Options

![](static/debug-view-options.png)

## Mask Creator

![](static/mask-creator.png)

::: notes

Used to create masks and zones to customize your detections.

:::

## Mask Output

![](static/mask-output.png)

::: notes

Can be copied and pasted into the config file to define your zones and masks

:::

# Advanced Deployment

::: notes

So that gets you to a functional, but basic deployment of Frigate.
Here's the fun stuff.

:::

## Hardware Acceleration

Config File

```
detectors:
  coral:
    type: edgetpu
    device: pci
```

::: notes

This is the config file for using a Coral TPU.
If you have more than ten or so cameras, you may need more than one.
If that's the case, you have to specify the PCI path manually

:::

## Hardware Acceleration

Dockerfile

```
devices:
  - /dev/apex_0:/dev/apex_0
  - /dev/dri/renderD128:/dev/dri/renderD128
```

::: notes

You also need to pass through the Coral (apex_0)
And the video hardware acceleration if desired.
The above is for a QuickSync device

:::

## MQTT

```
mqtt:
  host: 192.168.123.456
  user: frigate
  password: password
```

::: notes

This is how Frigate will communicate with HA
Or any other MQTT capable service

:::

## Split Feeds

```
cameras:
  doorbell:
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:8554/doorbell?video=copy&audio=aac
          input_args: preset-rtsp-restream
          roles:
            - record
        - path: rtsp://127.0.0.1:8554/doorbell_sub?video=copy&audio=aac
          input_args: preset-rtsp-restream
          roles:
            - detect
            - rtmp
```

::: notes

Some cameras offer a full quality feed as well as alow quality one
This example is only recoring the high quality, and using the low quality for object detection (faster) and restreaming to other devices.

:::

## Objects per Camera

```
cameras:
  garage:
    objects:
      track:
        - person
        - cat
        - dog
```

::: notes

You can configure which types of objects are reported for each camera

:::

## Masks & Zones

```
cameras:
    doorbell:
        motion:
          mask:
            - 0,86,93,45,310,42,483,82,501,271,455,280,292,297,149,298,73,301,0,324
            - 360,480,640,480,640,480,640,449,360,449
        objects:
          filters:
            car:
              mask:
                - 0,86,93,45,310,42,483,82,501,271,455,280,292,297,149,298,73,301,0,324
        zones:
          porch:
            coordinates: 316,413,341,459,228,480,640,480,640,480,640,384,640,384,640,187,640,0,476,81,493,253,474,417,427,380
          driveway:
            coordinates: 473,414,500,295,396,278,279,292
          street:
            coordinates: 0,96,61,71,165,50,237,43,343,59,461,94,476,79,492,166,492,272,401,284,253,296,145,304,64,300,0,335
          left_yard:
            coordinates: 281,291,55,288,0,329,0,416,155,480,338,458,325,419,435,387
          right_yard:
            coordinates: 492,266,491,299,404,277
```

::: notes

This is where you would copy the coords from the zone editor we showed off earlier.
You can mask of motion areas, so they won't trigger detection
Mask of certain areas for certain objects (This is so my doorbell won't trigger on cars in the street)
Make zones so your alerts can specify where an event happened.

:::

# Home Assistant

## Automations

```
trigger:
  - platform: mqtt
    topic: frigate/events
    payload: "doorbell/new"
    id: frigate-event
condition:
  - "{{ trigger.payload_json['after']['camera'] == 'doorbell' }}"
  - "{{ trigger.payload_json['after']['label'] == 'person' }}"
```

::: notes

This is a simple example of a trigger and condition to fire off some action in HA

:::

## Automations

[https://github.com/SgtBatten/HA_blueprints](https://github.com/SgtBatten/HA_blueprints)

::: notes

If you want notifications though, there's a great bluebrint that makes it super easy.

:::

## Dashboard

```
show_state: true
show_name: true
camera_view: auto
type: picture-entity
entity: camera.doorbell
```

::: notes

You can use the default picture card to get the current snapshot of a camera on your dashboard

:::

## Dashboard

[https://github.com/dermotduffy/frigate-hass-card](https://github.com/dermotduffy/frigate-hass-card)

::: notes

But this card gives you live video, the ability to see events, and swap between cameras

:::

# General Advice

## Google Coral

- [https://coral.ai/](https://coral.ai/)
- Buy One

::: notes

No notes.
If you want to run Frigate, you should invest in one.
The're under $50 and come in USB, PCIE, MiniPCI

:::

## Dedicated Machine

- (Much) Easier Hardware Config
- Storage Options

::: notes

I didn't do this. I wish I had.
Does not like to run in a VM.
Hard to get running in LXC on Proxmox.

:::

## Hardwired Cameras

- Preserve WiFi Bandwidth
- Security
- Reliability

# What Else ya Got?

## Face Recognition

https://github.com/skrashevich/double-take

## Bird ID

https://github.com/mmcc-xx/WhosAtMyFeeder

# Further Reading

## Official Page

[https://docs.frigate.video/](https://docs.frigate.video/)

## Frigate HA Integration

[https://github.com/blakeblackshear/frigate-hass-integration](https://github.com/blakeblackshear/frigate-hass-integration)

# The End

---

Robert Ward<robert@rtward.com>

![](static/qrcode.png)

Talk: [${TALK_URL}](${TALK_URL})

Repo: [${REPO_URL}](${REPO_URL})
