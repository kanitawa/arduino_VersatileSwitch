# VersatileSwitch
This is a library for Arduino to easily handle momentary switches, such as tact switches.

This library performs "debouncing" of switch contacts and can detect various switch actions such as "single/double clicks", "long presses", and "automatic repetitions" when a switch is pressing continuously. 

These events can be detected by getter functions such as “isClicked()”. The detection like "Event-Driven" style is also possible by assigning a callback function to each switch action.

## How to use
Needs to create an instance of ```VersatileSwitch``` for each switch. The required argument is a "pin number". All digital input pins can be used for switch.

```C++
VersatileSwitch mySwitch(4);
```

If only the “pin number” is specified, the switch is set in
- “connection mode” of switch is ```INPUT_PULLUP```
- "pin voltage" is ```LOW```, when switch is ```ON```

These “connection mode” and “pin voltage” can be given as optional arguments of constructor.

```C++
VersatileSwitch mySwitch1(3, INPUT_PULLUP);

VersatileSwitch mySwitch2(4, INPUT, LOW);
```

The “connection mode” is the 2nd argument, and either ```INPUT``` or ```INPUT_PULLUP``` is given (default value is ```INPUT_PULLUP``` ).

The "pin voltrage" is the 3rd argument, and one of ```LOW```, ```HIGH```, or ```AUTO``` is given (default value is ```AUTO``` ).

The 3rd argument ```AUTO``` means the following: 

- ```HIGH``` when the 2nd argument is ```INPUT``` .
- ```LOW``` when the 2nd argument is ```INPUT_PULLUP``` .

For all instances, ```poll()``` is called periodically to check and update the switch state. This will be usually done within ```loop()```. 

After calling ```poll()``` to check and update the switch state, they can be retrieved with the getter function.

```C++
void loop() {
    mySwitch.poll();

    if (mySwitch.isClicked()) {

        Serial.println("Clicked.");

    } else if (mySwitch.isLongClicked()) {

        Serial.println("Long-clicked.");

    }
}
```

Also, assigning a callback function to a switch operation in ```setup()``` , the callback function will be called automatically in ```poll()``` when those operations are performed.

```C++
void setup() {

    mySwitch.attachCallback_Clicked(on_switch_clicked);
    
    mySwitch.attachCallback_LongClicked(on_switch_long_clicked);

}

void loop() {

    mySwitch.poll();

}

void on_switch_clicked() {

    Serial.println("Clicked.");

}

void on_switch_long_clicked() {

    Serial.println("Long-clicked.");

}
```

## Getter functions
### ```isOn()```
### ```isPressed()```
Returns ```true``` if the switch is in the "**Pressed** ( ```ON``` )" state.

### ```isOff()```
### ```isReleased()```
Returns ```true``` if the switch is in the "**Released** ( ```OFF``` )" state.

### ```isHeld()```
Returns ```true``` if the switch continues to be pressed and is in the “**automatic repetitions**” state.

Note: In this “**automatic repetitions**” state, ``isPressed()`` returns ```false``` .

### ```isClicked()```
### ```isDoubleClicked()```
### ```isLongClicked()```
Returns ```true``` if the switch action "click / double-click / long-click" has been determined in ```poll()``` just before calling this function. These functions is transient, returning ```true``` only after the ```poll()``` when the switch action is finalized.

## Attach callback functions
### ```attachCallback_Pressed()```
### ```attachCallback_Clicked()```
### ```attachCallback_Held()```
### ```attachCallback_Repeated()```
### ```attachCallback_LongClicked()```
### ```attachCallback_DoubleClicked()```
### ```attachCallback_Released()```
### ```attachCallback_Finalized()```

Attach a callback function for each switch action. The type of the argument is ```void(*)(void)```  pointer of function, like a following:

```C++
void func() {
    ...
}
```

Each function is callbacked from within ```poll()``` when the switch action confirmed.

```attachCallback_Finalized()``` is specially callbacked when a sequencial action such as a click or double-click is finalized. See "**About inner-works and timing charts of callback**" below for details.

## Setter functions for time constatnt
### setTimeParalyze(uint32_t)
Sets the time constant for debouncing, called as "time of paralyze ( T<sub>Paralyze</sub> )", in msec. The default value is "5".

### setTimeUntilHold(uint32_t)
Sets the time constant to change the state from "start of pressing" to "**automatic repetitions**", called as "time of pressing ( T<sub>Pressing</sub> )", in msec. The default value is "500".

### setTimeRepeatInterval(uint32_t)
Sets the interval in "**automatic repetitions**", called as "time of repeating ( T<sub>Repeat</sub> )", in msec. The default value is "500".

### setTimeAcceptDoubleClick(uint32_t)
Sets the time constant, called as "time for accept double click ( T<sub>Accept</sub> )", needed to confirm that successive clicks are double clicks, not two single clicks, in msec. The default value is "200".

## About inner-works and timing charts of callback
### Debouncing of the switch and ```Pressed``` / ```Released``` callback

For each poll(), checks the switch position [ ```ON```, ```OFF``` ], and if it is different from the prevoius one, state of the switch will be "**Paralyzed**". This "Paralyzed" state will be continued during " T<sub>Paralyze</sub> " and no action are occured.

At the first ```poll()``` after the " T<sub>Paralyze</sub> " has expired, the switch position is checked again and ```Pressed``` or ```Released``` is callbacked depending on its position. The status of the switch will accordingly change to “**RELEASED**” or “**PRESSED**”.

Note that callbacked a```Released```, just means that the switch is released, not that a action such as a click or double-click is finalized.

![Figure_debauncing](https://github.com/kanitawa/VersatileSwitch/blob/images/figure_debauncing.png)


### Continuous presses of switch and ```Held``` / ```Repeated``` / ```LongClicked``` callback
After the status has been changed to “**PRESSED**”, if the switch was still being pressed at the first ``poll()`` after the “T<sub>Pressing</sub>” time expired, the status will be changed to “**HELD**”. At that timing ```Held``` -> ```Repeated``` will be callbacked.

Thereafter, ```Repeated``` is callbacked for each first ``poll()`` beyond “T<sub>Repeat</sub>” time.

Then, ```Released``` -> ```LongClicked``` -> ```Finalized``` are callebacked at the first ```poll()``` after the switch is released.

![Figure_repeating](https://github.com/kanitawa/VersatileSwitch/blob/images/figure_repeat.png)

### Click dicision and ```Clicked``` / ```DoubleClicked``` callback
After the status becomes “**PRESSED**”, If the switch is released within “T<sub>Pressing</sub>” time , the status will be changed to the special status “**RELEASED_AFTER_CLICK**”.

In “**RELEASED_AFTER_CLICK**” state, if the switch is not re-pressed within “T<sub>Accept</sub>” time, the single-click is confirmed and ```Released``` -> ```Clicked``` -> ```Finalized``` are callbacked, then the status returns to "**RELEASED**".

![Figure_click](https://github.com/kanitawa/VersatileSwitch/blob/images/figure_click.png)

In the “**RELEASED_AFTER_CLICK**” state, if the switch is pressed again within the “T<sub>Accept</sub>” time, the status changes to another special state called “**PRESSED_AFTER_CLICK**".

After “**PRESSED_AFTER_CLICK**”, if the switch is released within “T<sub>Pressing</sub>”, the double-click is confirmed and ```Released``` -> ```DoubleClicked``` -> ```Finalized``` are callbacked, then the status returns to "**RELEASED**".

![Figure_double_click](https://github.com/kanitawa/VersatileSwitch/blob/images/figure_double_click.png)

### "Click & Hold" and ```Finalized``` callback
After a single-click was confirmed and became in the “**PRESSED_AFTER_CLICK**”, if the switch is not released within “T<sub>Pressing</sub>”, this sequencial action is confirmed as “Click & Hold”. The ```Clicked``` -> ```Held``` -> ```Repeated``` are callbacked, and then the status is changed to "**HELD***".

Note that ```Finalized``` is not callbacked after ```Clicked```, unlike in the normal single-click case.

After this, it is the same as the normal "continuous press": ```Repeated``` is callbacked for each “T<sub>Repeat</sub>” time and ```Released``` -> ```LongClicked``` -> ```Finalized``` are callebacked at the timing of releasing switch.

![Figure_click_and_hold](https://github.com/kanitawa/VersatileSwitch/blob/images/figure_click_and_hold.png)
