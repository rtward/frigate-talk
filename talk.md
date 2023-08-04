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

Frigate has a first class integration with Home Assistant through MQTT. By listening in on events you can respond to objects detected in different parts of your home.

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

## Record

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

# UI

## Cameras

![](static/cameras-view.png)

## Birds Eye

![](static/birds-eye-view.png)

## Events

![](static/events-view.png)

## Recordings

![](static/recordings-view.png)

## Storage

![](static/storage-view.png)

## Debug

![](static/debug-view.png)

## Debug Options

![](static/debug-view-options.png)

## Mask Creator

![](static/mask-creator.png)

## Mask Output

![](static/mask-creator.png)

# Advanced Deployment

## Hardware Acceleration

Config File

```
detectors:
  coral:
    type: edgetpu
    device: pci
```

## Hardware Acceleration

Dockerfile

```
devices:
  - /dev/apex_0:/dev/apex_0
  - /dev/dri/renderD128:/dev/dri/renderD128
```

## MQTT

```
mqtt:
  host: 192.168.123.456
  user: frigate
  password: password
```

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

## Automations

[https://github.com/SgtBatten/HA_blueprints](https://github.com/SgtBatten/HA_blueprints)

## Dashboard

```
show_state: true
show_name: true
camera_view: auto
type: picture-entity
entity: camera.doorbell
```

# General Advice

## Google Coral

## Dedicated Machine

## Hardwired Cameras

# What Else ya Got?

## Face Recognition

https://github.com/skrashevich/double-take

## Bird ID

https://github.com/mmcc-xx/WhosAtMyFeeder

# The End

---

Robert Ward<robert@rtward.com>

![](static/qrcode.png)

Talk: [${TALK_URL}](${TALK_URL})

Repo: [${REPO_URL}](${REPO_URL})
