# Dell XPS 15 L502X P11F Laptop Keyboard

## Context

I have harvested a laptop keyboard from an old Dell XPS 15 laptop. I wanted to turn it into a standalone keyboard. I have done a similar project before. So this round it was a lot easier and I decided to document the process in case anyone else is interested in doing something similar.

## Basic concepts

Laptop keyboards are simple, at least all the ones I have seen. They are passive parts. I.e. there's no chip in them. They have a set of wires. When a key is pressed, two of the wires are shorted. And that's all there is to it. Most pre-USB era keyboards work the same way. 

Unlike USB keyboards, they need a chip and a model specific driver to work. The wires are divided into two groups, rows and columns. Each key is connected to one row wire and one column wire. When pressed, these two wires are connected. The driver *scans* the wires at a high frequency, pulling each row high sequentially, and read all the columns. If a key is pressed, its corresponding column will read high when its column is pulled high. This way, the driver can determine which key is pressed. Of course, the driver can also reverse the logic and pull the rows low if their normal state is high. It can also by writing the columns and reading the rows.

Note that the terms of *row* and *column* has nothing to do with the physical layout. In fact, they usually don't resemble the physical layout at all. The reason is that laptop keyboards don't have a PCB per se. All the wires are laid out in one layer. I.e. no two wires can cross each other. This makes laying out the wires quite challenging. If you're into mechanical keyboards, you have probably noticed that the rows and columns of a mechanical keyboard more or less reflect the physical layout. That is because they have two sides of a PCB to work with, making wire crossing possible. 

Also, laptop keyboards lack diodes which are found in mechanical keyboards. So, anti-ghosting can only be achieved by using a sparse matrix. This, combined with the layout challenge, means that there are usually much more wires in a laptop keyboard than mechanical keyboards although they work very similarly. In particular, this 86-key keyboard has 25 wires. A typical 75% mechanical keyboard would have 19 (9x10 = 90) or at most 20. Even with this many more wires, it's not possible to completely avoid ghosting. But the designer usually strategically assign the rows and columns such that ghosting only happens in key combos that don't normally occur during normal usage. Thus, these keybaords are normally not suitable for gaming.

## Overall approach

To make use of this keyboard, my plan is to first determine the matrix. I.e. what key corresponds to what wires and the division of rows and columns. After that, I will use a microcontroller to interface between the keyboard and the host as a USB device. Then I will 3D print an enclusure. 

## Determine the matrix

### Algorithm
The algorithm for determining the matrix is simple. I connect all wires to a microcontroller's GPIO pins. Then I iterate through all these pins, pulling one pin low at a time, leaving all other pins pulled up. Then I read all other pins. If any of them is low, it must be connected to the first pin via a key. So I output those two pins on the seraial console. The program runs indefinitely. I press the keys one by one and manually write down the corresponding wires on a sheet of paper.

### Hardware

* 1mm FPC ribbon breakout
* Breadboard
* Microcontroller. I used a Raspberry Pi Pico, which has just enough GPIO pins
* Jumper wires

### Code
I used Arduino studio and its framework to write the scaner code. The array `gpio` is a mapping from wire numbers to GPIO pin numbers. On the serial console, I print out the wire numbers.

```
const int pins = 25;
int gpio[pins] = {0, 28, 2, 27, 3, 26, 4, 22, 5, 21, 6, 20, 7, 19, 8, 18, 9, 17, 10, 16, 11, 14, 12, 15, 13};

void setup() {
  Serial.begin(115200);
  Serial.println("Starting to scan...");
  delay(200);
}

void loop() {
  for (int i = 0; i < pins; i ++) {
    pinMode(gpio[i], OUTPUT);
    digitalWrite(gpio[i], LOW);
    for (int j = i + 1; j < pins; j ++) {
      pinMode(gpio[j], INPUT_PULLUP);
      int value = digitalRead(gpio[j]);
      if (value == LOW) {
        Serial.print(i + 1);
        Serial.print(" - ");
        Serial.print(j + 1);
        Serial.println();
      }
    }
    digitalWrite(gpio[i], HIGH);
  }
  delay(100);
}
```

For this particular keyboard, the key-wire association looks like this.

* ESC: 8 - 14
* F1: 7 - 15
* F2: 7 - 16  
...

### Assigning rows and columns
Next, I divide the wires into two sets: rows and columns. Each key is associated with one row and one column.  Essentially, this is a [bigraph](https://en.wikipedia.org/wiki/Bipartite_graph) algorithm. Each wire is a vertex and each key is an edge connecting two vertices. All edges are between rows and columns. There's no edge within rows or columns. One can write a program to do this. But I just did it manually.

* First, assign one wire to the row set. This is completely arbitray.
* For each wire in the row set, iterate through its edges, assigning the other end of each edge to the column set.
* Do the same for the column set.
* Repeat the previous two steps untill all wires are assigned.

In my case, here's the assignment.

* Rows: wires 1 - 8
* Columns: wires 9 - 25

Huh, that was neat, wasn't it?!

## Build firmware

There are a few ways to do the firmware. The simplest one is using Arduino HID library with a supported microcontroller. You just scan the matrix similar to the code above. When you have detected a key press, send it to the host. You need to remember the state of each key and do your own debouncing. It's pretty straight-forward but the functionality is also very limited.

A much more powerful way is to use mechanical keyboard firmware. There are a few open source options out there. I am using QMK, which has been the most popular one among mechanical keyboard enthusiasts and I have been playing with it for quite a while. Another option that's been gaining popularity is ZMK.

If you don't need anything super fancy and customized, you can create a keyboard by defining just two files.

* A json file describing meta data of the keyboard and phyisical layout and GPIO pin connections.
* A C file mapping physical keys to key codes.

The json file defines the row and column GPIO pins. 

```
"matrix_pins": {
    "rows": ["GP22", "GP4", "GP26", "GP3", "GP27", "GP2", "GP28", "GP0"],
    "cols": ["GP19", "GP8", "GP18", "GP9", "GP17", "GP10", "GP16", "GP11", "GP14", "GP12", "GP15", "GP13", "GP5", "GP21", "GP6", "GP20", "GP7"]
},
```

The row and column ordering is arbitrary. You just need to make sure they match the matrix definition later. There's one small consideration though. I like assiging ESC key row 0 and column 0 so that I can easily take advantage of the [*bootmagic*](https://docs.qmk.fm/features/bootmagic) feature of QMK, for easy flashing. In my case, I'm calling wire 8, connected to GPIO 22, row 0, wire 7, ceonncted to GPIO 4, row 1, and so on.

Now we need to tell the row and column numbers of each key. It has no knowledge of our "wire numbers". E.g. the wire association

* ESC: 8 - 14
* F1: 7 - 15
* F2: 7 - 16 

is translated into row-col association

* ESC: r0 c0
* F1: r1 c1
* F2: r1 c2

And this is how to describe it in the json file.

```
"layouts": {
    "LAYOUT_ansi": {
        "layout": [
            {"matrix": [0, 0], "x": 0, "y": 0, "label": "ESC"},
            {"matrix": [1, 1], "x": 0, "y": 0, "label": "F1"},
            {"matrix": [1, 2], "x": 0, "y": 0, "label": "F2"},
...
```

Next, we map these keys to keycodes that the OS understands. In QMK's terms, it's called a keymap. How you define it is highly subjective. With a standard keyboard, I like having two layers, moving some keys closer to the home position in the extra layer.

```
#include QMK_KEYBOARD_H

enum keyboard_layers {
  _BL,
  _LOWER,
};

#define LOWER MO(_LOWER)
#define CAPS_LT LT(_LOWER,KC_CAPS)

const uint16_t PROGMEM keymaps[][MATRIX_ROWS][MATRIX_COLS] = {
    [_BL] = LAYOUT_ansi(
      KC_ESC,  KC_F1,   KC_F2,   KC_F3,   KC_F4,   KC_F5,   KC_F6,   KC_F7,   KC_F8,   KC_F9,   KC_F10,  KC_F11,  KC_F12,  KC_VOLD, KC_VOLU, KC_MUTE, KC_DEL,
      KC_GRV,  KC_1,    KC_2,    KC_3,    KC_4,    KC_5,    KC_6,    KC_7,    KC_8,    KC_9,    KC_0,    KC_MINS, KC_EQL,  KC_BSPC, KC_HOME,
      KC_TAB,  KC_Q,    KC_W,    KC_E,    KC_R,    KC_T,    KC_Y,    KC_U,    KC_I,    KC_O,    KC_P,    KC_LBRC, KC_RBRC, KC_BSLS, KC_PGUP,
      CAPS_LT, KC_A,    KC_S,    KC_D,    KC_F,    KC_G,    KC_H,    KC_J,    KC_K,    KC_L,    KC_SCLN, KC_QUOT, KC_ENT,  KC_PGDN,
      KC_LSFT, KC_Z,    KC_X,    KC_C,    KC_V,    KC_B,    KC_N,    KC_M,    KC_COMM, KC_DOT,  KC_SLSH, KC_RSFT, KC_UP,   KC_END,
      KC_LCTL, LOWER,   KC_LGUI, KC_LALT, KC_SPC,  KC_RALT, KC_RCTL, KC_APP,  KC_LEFT, KC_DOWN, KC_RGHT
    ),
    [_LOWER] = LAYOUT_ansi(
      _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______,
      _______, KC_F1,   KC_F2,   KC_F3,   KC_F4,   KC_F5,   KC_F6,   KC_F7,   KC_F8,   KC_F9,   KC_F10,  KC_F11,  KC_F12,  KC_DEL,  _______,
      _______, KC_1,    KC_2,    KC_3,    KC_4,    KC_5,    KC_6,    KC_7,    KC_8,    KC_9,    KC_0,    _______, _______, _______, _______,
      KC_TILD, KC_UNDS, KC_PLUS, KC_LCBR, KC_RCBR, KC_PIPE, KC_LEFT, KC_DOWN, KC_UP,   KC_RGHT, _______, _______, _______, _______,
      KC_GRV,  KC_MINS, KC_EQL,  KC_LBRC, KC_RBRC, KC_BSLS, KC_HOME, KC_END,  KC_PGUP, KC_PGDN, _______, _______, KC_PGUP, _______,
      _______, _______, _______, _______, KC_ENT,  _______, _______, _______, KC_HOME, KC_PGDN, KC_END
    ),
};
```

The full source code is [here](https://github.com/delingren/qmk_firmware/tree/myboards/keyboards/delingren/xps15).

## Enclosure
TBD