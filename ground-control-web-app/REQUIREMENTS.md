# Requirements

## Deployment

Requirement: Build a docker image that corresponds to the Tech Stack:

*   [Python3](https://www.python.org/) - main programming language
*   [UV python package manager](https://docs.astral.sh/uv/): modern package manager that turns
package management hell into pleasure
*   [NiceGUI](https://github.com/nicegui/nicegui): Modern and easy to use Web Framework for Python
    * With some TypeScript patches to handle extra things such as Gamepad API
*   [Gamepad API](https://developer.mozilla.org/en-US/docs/Web/API/Gamepad_API): For USB controller input
*   [Python MQTT Paho library](https://pypi.org/project/paho-mqtt/): MQTT5 library with WebSocket support
*   [Docker](https://www.docker.com/): Containerization platform

---

The application uses `uv` as the python package manager. Therefore, the image build process shall be staged.
The application is build as a docker image. To build an image run:

```sh
docker build -t ground-control-web-app .
```

To run the image, execute the following

```sh
docker run -d --name ground-control-app --net host ground-control-web-app
```

## UI design

I need you to create a web application that performs the following:

1. Splits vertically the screen in a proportion of 10% for the left pane 
(called a menu), and the right pane. The left pane is collapsible, on a click of a button
in the top left corner

2. The menu is meant to provide 3 buttons: Chassis, Manipulator, Science.
The behavior for each button shall be provide in section "Business behavior" below.
Only one button can be in active state. The button shall be highlighted when active.

3. The right side pane shall be split horizontally in 80% (called a playground pane) to 
20% ratio (called a telemetry pane).

4. The playground pane shall display widgets associated with any of the active buttons.

5. The telemetry pane shall display text and widgets associated with the active state.

6. The UI shall support both the PC screen with minimal resolution: 1366 x 768, 
and a mobile app screens in both vertical and horizontal orientation.

7. The mobile screen shall accept multi touch.

## Controllers

1. The application shall use UI as a source of MQTT events,

2. The application shall use two USB controllers at once:

   2.1. A gamepad (Xbox Wireless Controller) 

   2.2. A joystick (Logitech Extreme 3D Pro). This controller comes with 'Z-axis', unlike the gamepad

3. Both the gamepad and the joystick are hot-swappable and always active whenever connected.

4. The USB controllers are prioritized in input, which means whenever these are connected,
the input provided by the panes are ignored.

5. The USB controllers are mapped in a way to correspond the UI widgets.
Refer to 'Business logic' section for detailed mapping

6. The USB controllers shall be probed at frequency of 50Hz. This value shall 
be easily configurable

## Pane Designs

This section introduces instructions for development of the mentioned 3 playground panes:
Chassis, Manipulator, Science..

### Playground pane: Chassis

1. The Chassis pane is a seperate widget, located in a separte python file.

2. The core of of the widget is a draggable area, known as `ui.joystick` 
in NiceGUI framework. It comprises a section of a solid background arranged 
in a shape of a rectangle.

3. The `ui.joystick` produces events that form a vector. The axis left-right is called 
the X-axis, and up-down is called the Y-axis. Both axes produce events whenever the 
knob is dragged a value of a 2D vector in range [-1, 1].
Therefore, whenever the knob is moved, it generates an event with X and Y values.
Expected mapping:
```
X Axis: max left: -1, max right: 1
Y Axis: max up: -1, max down: 1
```
The event can be subscribed later. The event shall be represented as `"left_stick": [x_value_real, y_value_real]`

3. Under the `ui.joystick` widget, you will place another `ui.joystick` known as rotation slider. The slider reflect
the Z-axis in the joystick such as Logitech Extreme 3D Pro.
The slider shall use values between -100 and 100. Whenever the user drops the slider button, it is automatically reset to 0, exactly in the middle of the slider. The values shall map
to `axes[2]` in Gamepad API, which corresponds to `rotate` field in the 
'Chassis JSON message' below.

4. At the bottom of the widget there are 4 buttons, namely X, Y, A, B.
They will be colored as the one found in the Xbox Controller.
These buttons generate events that correspond to a click (a brief press and release):
```
"button_x": true|false,
"button_y": true|false, 
"button_a": true|false,
"button_b": true|false
```

5. Whenever any state associated with the widget is triggered, 
emit all current values within the payload, no more frequent than 50Hz (this should be 
a configurable value). Therefore there will be only one event to subscribe to.

### Playground pane: Manipulator

Do not implement

### Playground pane: Science

Do not implement

## Business Logic

### Chassis active state

The user taps the Chassis button in the menu on the left side pane. 
It opens the playground side pane with the chassis controller widget,
and the telemetry pane with in the bottom section.

The bottom section name shall display a label "Mode: Chassis", and accept inbound traffic
from an MQTT topic `orion/topic/chassis/outbound`. Once the user leaves from the Chassis mode, unsubscribe from the topic.

The application shall accept input from both a physical gamepad and the UI. The application
shall use the following:

* analog stick (the knob): extract analog values X and Y
* 'X' button': pressed value
* send a JSON-serialized message onto the topic `orion/topic/chassis/controller/inbound`

Chassis JSON message:

```json
{
  "eventType": "chassis",
  "payload": {
    "stick": [x_value_real, y_value_real],
    "button_x": true|false,
    "button_y": true|false,
    "button_a": true|false,
    "button_b": true|false,
    "rotate": [z_value_real]
  }
}
```

Joystick mapping (Gamepad API):

* axes[0] and axes[1] correspond to `stick` field
* axes[2] corresponds to `rotate` field, applies retrieved analog value from the joystick
an the slider value as described in 'Playground pane: Chassis' section.
* buttons[0] corresponds to `button_x` field
* buttons[1] corresponds to `button_y` field
* buttons[2] corresponds to `button_a` field
* buttons[3] corresponds to `button_b` field

## Manipulator active state

Display a label in the playground pane: "Manipulator functionality is not implemented yet"

## Science active state

Display a label in the playground pane: "Science functionality is not implemented yet"

# MQTT configuration

Apply the following MQTT configs:
* Broker address: ws://mqtt5:9001
* Protocol version: 5
* Automatic reconnection attempts: yet, at intervals of 5 seconds
* Client-id: ground-control-web-app-${timestamp}
* User: user
* Password: user
* Serialization format: JSON

The application shall be accessible to any device in a local network. The broker
address shall be therefore configurable