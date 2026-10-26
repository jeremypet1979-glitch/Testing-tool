Put each switch's APPROVED configuration here, in Junos "set" format, named as on the
Switches tab (e.g. SW-01.set).

To capture it from a switch that has the approved config loaded:
  show configuration | display set | no-more
Copy the output into SW-01.set (or: show configuration | display set | save /var/tmp/SW-01.set
and copy the file off with SCP).

The test tool compares the running config line-by-line against this file, ignoring comment
lines and the "set version" line, and writes any differences to <switch>_config_diff.txt.
