# nozzle-cleaner
[![Stencil Fix Portable YouTube video](https://i.ytimg.com/vi/U1onfGroUJI/maxresdefault.jpg)](https://youtu.be/U1onfGroUJI "Stencil  Fix Portable")

## Built
![asdf](images/cleaner_render.png)

Print `nozzle_purge_bumper_holder.stl` and `nozzle_purge_sheet_endstop_holder.stl`. In addition, print `ptfe_tube_cutter.stl` for cutting the PTFE tube. And choose one of the buckets form the [original mod](https://github.com/VoronDesign/VoronUsers/tree/main/orphaned_mods/edwardyeeks/Decontaminator_Purge_Bucket_%26_Nozzle_Scrubber/STLs) (any of `purge_bucket_*mm_rev4.stl`).

The [YouTube video](https://youtu.be/U1onfGroUJI) shows how to install the mod.

## BOM
 | Part | Amount | Comment |
 |---|---|---|
 | M3*16 socket head screw | 2 | For build plate to index against |
 | M3*8 socket head screw | 3 | Mounting mod to plate |
 | M2*20 socket head screw | 1 | To mount PTFE tube |
 | 12.1mm PTFE tube | 1 |  |
 | 6x3mm round magnets | 2 | For the purge bucket to connect to screws |
 | M3*16 socket head screw | 1 | [optional] For PTFE cutter |
 | M3 nut | 1 | [optional] For PTFE cutter |

## Config
The macro for cleaning the nozzle can be found in `clean_nozzle.cfg`. I use it in the print start macro as follows

```
[include clean_nozzle.cfg]

# from https://github.com/jontek2/A-better-print_start-macro
[gcode_macro PRINT_START]
gcode:
  # This part fetches data from your slicer. Such as bed temp, extruder temp, chamber temp and size of your printer.
  {% set target_bed = params.BED|int %}
  {% set target_extruder = params.EXTRUDER|int %}
  {% set target_chamber = params.CHAMBER|default("40")|int %}
  {% set x_wait = printer.toolhead.axis_maximum.x|float / 2 %}
  {% set y_wait = printer.toolhead.axis_maximum.y|float / 2 %}

  # Homes the printer, sets absolute positioning and updates the Stealthburner leds.
  STATUS_HOMING         # Sets SB-leds to homing-mode
  G28                   # Full home (XYZ)
  G90                   # Absolut position

  ##  Uncomment for bed mesh (1 of 2)
  BED_MESH_CLEAR       # Clears old saved bed mesh (if any)

  # Checks if the bed temp is higher than 90c - if so then trigger a heatsoak.
  {% if params.BED|int > 90 %}
    SET_DISPLAY_TEXT MSG="Bed: {target_bed}c"           # Displays info
    STATUS_HEATING                                      # Sets SB-leds to heating-mode
    M106 S255                                           # Turns on the PT-fan

    ##  Uncomment if you have a Nevermore.
    SET_FAN_SPEED FAN=BedFans SPEED=1.0                 # Turns on the nevermore

    G1 X{x_wait} Y{y_wait} Z15 F9000                    # Goes to center of the bed
    M190 S{target_bed}                                  # Sets the target temp for the bed and wait
    SET_DISPLAY_TEXT MSG="Heatsoak: {target_chamber}c"  # Displays info
    TEMPERATURE_WAIT SENSOR="temperature_sensor chamber_temp" MINIMUM={target_chamber}   # Waits for chamber to reach desired temp

  # If the bed temp is not over 90c, then it skips the heatsoak and just heats up to set temp with a 5min soak
  {% else %}
    SET_DISPLAY_TEXT MSG="Bed: {target_bed}c"           # Displays info
    STATUS_HEATING                                      # Sets SB-leds to heating-mode
    G1 X{x_wait} Y{y_wait} Z15 F9000                    # Goes to center of the bed
    M190 S{target_bed}                                  # Sets the target temp for the bed and wait
    # not convinced that is needed for PLA and the likes
    # SET_DISPLAY_TEXT MSG="Soak for 5min"                # Displays info
    # G4 P300000                                          # Waits 5 min for the bedtemp to stabilize
  {% endif %}

  # Go to purge bucket, heat to target temp, purge, cool to 150, wipe nozzle
  clean_nozzle PURGE={target_extruder} CLEAN=150

  # Heating nozzle to 150 degrees. This helps with getting a correct Z-home
  SET_DISPLAY_TEXT MSG="Hotend: 150c"          # Displays info
  M109 S150                                    # Heats the nozzle to 150c

  # Quad gantry level AKA QGL
  SET_DISPLAY_TEXT MSG="QGL"      # Displays info
  STATUS_LEVELING                 # Sets SB-leds to leveling-mode
  quad_gantry_level               # Levels the buildplate via QGL
  G28 Z                           # Homes Z again after QGL

  # Bed mesh (2 of 2)
  SET_DISPLAY_TEXT MSG="Bed mesh"    # Displays info
  STATUS_MESHING                     # Sets SB-leds to bed mesh-mode
  bed_mesh_calibrate                 # Starts bed mesh

  # Go to purge bucket, heat to target temp, wipe nozzle
  clean_nozzle CLEAN={target_extruder}

  # Heats up the nozzle up to target via data from slicer
  SET_DISPLAY_TEXT MSG="Hotend: {target_extruder}c"             # Displays info
  STATUS_HEATING                                                # Sets SB-leds to heating-mode
  G1 X{x_wait} Y{y_wait} Z15 F9000                              # Goes to center of the bed
  M107                                                          # Turns off partcooling fan
  M109 S{target_extruder}                                       # Heats the nozzle to printing temp

  # Gets ready to print by doing a purge line and updating the SB-leds
  SET_DISPLAY_TEXT MSG="Printer goes brr"          # Displays info
  STATUS_PRINTING                                  # Sets SB-leds to printing-mode
  G0 X{x_wait - 50} Y4 F10000                      # Moves to starting point
  G0 Z0.4                                          # Raises Z to 0.4
  G91                                              # Incremental positioning 
  G1 X100 E20 F1000                                # Purge line
  G90                                              # Absolut position
```

The macro can also be called from the command line for testing like so:

```
clean_nozzle PURGE=215 CLEAN=150 # Go to purge bucket, heat to target temp, purge, cool to 150, wipe nozzle
clean_nozzle PURGE=215           # Go to purge bucket, heat to target temp, purge, wipe nozzle
clean_nozzle                     # Go to purge bucket, wipe nozzle
```

## Acknowledgement
This mod is based on the [Decontaminator Purge Bucket & Nozzle Scrubber](https://github.com/VoronDesign/VoronUsers/tree/main/orphaned_mods/edwardyeeks/Decontaminator_Purge_Bucket_%26_Nozzle_Scrubber) mod from @edwardyeeks. The macro is also heavily inspired by Edward's excellent work. Thanks a bunch ❤️