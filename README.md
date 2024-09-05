# photovoltaic sensor
a punk light sensor with photovoltaic cell + rp2040 + circuitpython

| | MCU |
| :------------------- | :----------: |
| pot pin +                | 3v3 |
| pot pin data           | pin29 |
| pot pin -                | GND |
| cell +               | pin 27 |
| cell GND             | GND |


![photo du capteur](./capteur.jpg "détail du montage")

A simple Processing example with the great [theMidiBus](https://github.com/sparks/themidibus) lib

```Processing
import themidibus.*;

MidiBus bus;
int diametre= 10;

void setup() {
  size(800,800);
  MidiBus.list();
  // 1 -> numero dans liste des input
  //MidiBus(java.lang.Object parent, int in_device_num, int out_device_num)
  bus = new MidiBus(this, 1,2);
  bus.sendNoteOff(new Note(0,64,127));
  noStroke();
}

void draw() {
  background(220);
  fill(25);
  circle(width/2, height/2, diametre);
}

void controllerChange(int channel, int number, int value) {
    switch(number) {
      case 31:
        diametre = int(map(value, 0, 127, 0, width));
        println(diametre);
        break;
      default:
        println(number,value,"générique");
    }
}
```
The circuitPython script for a RP2040 Pico Zero from Waveshare
```python
import board
import analogio
import time
import simpleio

# midi part
import usb_midi
import adafruit_midi
from adafruit_midi.control_change import ControlChange
from adafruit_midi.note_off import NoteOff

# gestion de la led
import neopixel
led = neopixel.NeoPixel(board.NEOPIXEL, 1)
led.brightness = .3


# Créer un objet analogique sur la pin 27
cellule_in = analogio.AnalogIn(board.GP27)
# Créer un objet analogique sur la pin 29
pot_in = analogio.AnalogIn(board.GP29)
limite = 65535
led[0] = (255, 0, 0) # green

# init du midi
USB_MIDI_channel = 1  # pick your USB MIDI out channel here, 1-16

usb_midi = adafruit_midi.MIDI(
    midi_in=usb_midi.ports[0], in_channel=USB_MIDI_channel - 1,
    midi_out=usb_midi.ports[1], out_channel=USB_MIDI_channel - 1
)

print("--------------------------")
print("|    Flux MIDI enc.    |")
print("--------- @mrbbp ---------")
print("|   USB MIDI channel: {}  ]".format(USB_MIDI_channel))
print("--------------------------")

init_value_sent = False
#msg = usb_midi.receive()
midi_value = old_value = 128
addr = 31 # midi adress

while True:
    msg = usb_midi.receive()

    if msg is not None:
        if isinstance(msg, NoteOff) and msg.note == 64 and not init_value_sent:
            print("noteOff received - start to send midi datas")
            init_value_sent = True
            limite = pot_in.value
    # read analog values
    if init_value_sent:
        cel = cellule_in.value
        pot = pot_in.value
        seuil = int(simpleio.map_range(pot, 0, 65535, 0, 127)) # value 0-127
        # adjust cel max value to value 127
        if pot <= cel:
            led[0] = (0,255, 0) # red
        else:
            led[0] = (255,0,0) # green
            limite = pot
        # map cell value from 0-127
        midi_value = int(simpleio.map_range(cel, 2048, limite, 0, 127))
        if midi_value != old_value:
            #print("send midi value: ",midi_value)
            old_value = midi_value
            usb_midi.send(ControlChange(addr, midi_value))
            time.sleep(.03) # wait 30ms
    time.sleep(.01) # wait 10ms

```
