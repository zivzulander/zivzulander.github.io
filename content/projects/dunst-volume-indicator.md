+++
title = "Dunst Volume Indicator for Pipewire"
author = ["Matt Morris"]
draft = false
+++

Date: <span class="timestamp-wrapper"><span class="timestamp">August 18, 2022</span></span>

Here I will be creating a clever volume indicator for linux systems without a dedicated desktop environment. The end result is to look nice:
![](/blog-images/volume.png)

Full script can be found on my GitHub.

It must serve the following:

-   work with `Pipewire`
-   work with `Dunst`
-   allow for incremental increases in volume called from keybinding such as `sxhkd`
-   round appropriately to save my ocd from seeing numbers like 37%
-   mute if volume reaches 0%

Basic setup

```bash
#!/bin/bash
set -eux
```

Arbitrary but unique message tag

```bash
msgTag="myvolume"
```

Query the current volume.

```bash
currentvolume="$(pactl get-sink-volume @DEFAULT_SINK@ | awk '{print $5}' | grep -Eo '[0-9]+' )"
```

Functions to round volume up or down to the nearest 10th if changing volume by large factor. (e.g. if volume is at 34%, increase it to 40% or decrease to 30%). Also, the function to mute.

```bash
round_down() {
     pactl set-sink-volume @DEFAULT_SINK@ $(( $currentvolume - ${currentvolume:(-1)} ))%
}

round_up() {
     pactl set-sink-volume @DEFAULT_SINK@ $(( 10 + $currentvolume - ${currentvolume:(-1)} ))%
}

mute() {
    pactl set-sink-volume @DEFAULT_SINK@ 10%
    pactl set-sink-mute @DEFAULT_SINK@ 1
}
```

If inc/decreasing volume, un-mute first.

```bash
if echo "$@" | grep -Eq '[+|-][0-9]+' ; then
    pactl set-sink-mute @DEFAULT_SINK@ off
fi
```

If changing by a large number (&ge;10) and the current volume is not a factor of 10 then it should round. Input is usually something like

`volumeindicator set-sink-volume @DEFAULT_SINK@ +10%`

so we check for those two conditions.

```bash
if [[ ${currentvolume:(-1)} != 0  && $(echo "$@" | grep -Eo '[0-9]+' ) -ge 10 ]] ; then
   [ -n "$(echo "$@" | grep -Eo '\-[0-9]+' )" ] && round_down
   [ -n "$(echo "$@" | grep -Eo '\+[0-9]+' )" ] && round_up
   else
       pactl $@
fi
```

Get the new volume.

```bash
newvolume="$(pactl get-sink-volume @DEFAULT_SINK@ | awk '{print $5}' | grep -Eo '[0-9]+' )"
```

If volume is 0% don't change the volume, just mute.

```bash
[[ $newvolume == 0 ]] && mute
```

Execute `dunst`

```bash
dunstify -a "Volume" -u low -i audio-volume-high -h string:x-dunst-stack-tag:$msgTag \
    -h int:value:"$newvolume" "Vol:"
```

Full script:

```bash
#!/bin/bash
set -eux
msgTag="myvolume"
currentvolume="$(pactl get-sink-volume @DEFAULT_SINK@ | awk '{print $5}' | grep -Eo '[0-9]+' )"

round_down() {
     pactl set-sink-volume @DEFAULT_SINK@ $(( $currentvolume - ${currentvolume:(-1)} ))%
}

round_up() {
     pactl set-sink-volume @DEFAULT_SINK@ $(( 10 + $currentvolume - ${currentvolume:(-1)} ))%
}

# unmute first if increasing vol
if echo "$@" | grep -Eq '[+|-][0-9]+' ; then
    pactl set-sink-mute @DEFAULT_SINK@ off
fi

if [[ ${currentvolume:(-1)} != 0  && $(echo "$@" | grep -Eo '[0-9]+' ) -ge 10 ]] ; then
   [ -n "$(echo "$@" | grep -Eo '\-[0-9]+' )" ] && round_down
   [ -n "$(echo "$@" | grep -Eo '\+[0-9]+' )" ] && round_up
   else
       pactl $@
fi

newvolume="$(pactl get-sink-volume @DEFAULT_SINK@ | awk '{print $5}' | grep -Eo '[0-9]+' )"

dunstify -a "Volume" -u low -i audio-volume-high -h string:x-dunst-stack-tag:$msgTag \
    -h int:value:"$newvolume" "Vol:"
```
