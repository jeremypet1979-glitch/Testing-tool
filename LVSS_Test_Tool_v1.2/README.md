# Sprout 1.2 LVSS Skid – Section 1.1.5 Functional Test Tool

This tool runs the Section 1.1.5 functional tests from the Sprout 1.2 LVSS Skid Test Plan. It records every result with a timestamp and builds the report package for the Owner and CxA. Each result carries a test ID that matches the plan (1.1.5-1.x MOXA IO, 1.1.5-2.x Network Switch, 1.1.5-3.x SPD), so the CxA can trace every line in the report back to the plan.

The tool handles the parts a computer can measure: Modbus reads, analog scaling math, polling statistics, ping, SNMP port and VLAN reads, and failover timing. For the parts a person has to look at or measure, it prompts the tech and records the answer in the same results file. Those parts include LEDs, mounting, lead lengths, continuity, and what the SCADA screen shows.

**Manual backup:** `Manual_Test_Procedure_1.1.5.pdf` in this folder performs every test by hand with vendor software, a meter and the Junos CLI. Use it to verify the tool on the first skid, as a backup on test day, or to re-test any item the Owner or CxA questions.

## Desktop app (recommended for field use)

`LVSS_Test_Tool.exe` runs the same tests with a window instead of a command prompt: test buttons grouped by section, prompts as large dialogs, a live color-coded results table, a Stop button, and one-click report and checklists. A yellow badge shows when it's in Simulation mode so dry-run data can't be mistaken for a real test.

**Build the .exe (once, on any Windows PC with Python):** double-click `build_exe.bat`. It installs what it needs, builds the app, runs a simulated self-check, and puts everything in `release\LVSS_Test_Tool_v<version>\` (plus a zip of it): the .exe, the `config` folder, the manual procedure PDF, and this README. Copy that folder or the zip to the test laptop. The laptop doesn't need Python.

**Try it without building:** double-click `launch_app.bat` (needs Python and `pip install -r requirements.txt`).

**License:** the first time the app (or `run_tests.py`) starts on a computer, it asks for the license number. Enter it once and that computer is remembered for that Windows user; copying the app to another computer asks again. Get the license number from whoever gave you the app. The number isn't stored in the program, and it doesn't appear on reports.

**Copying the app:** `build_exe.bat` makes a ready-to-copy folder, `release\LVSS_Test_Tool_v<version>`, and a zip of it. Copy either to any Windows computer. The `.exe` also works on its own: if there's no `config` folder next to it, it creates a starter `config\skid_config.xlsx` on first run.

**Using it:**
1. Check the config path at the top (it defaults to `config\skid_config.xlsx` next to the app) and enter your name in *Tested by*.
2. Plug into the skid switch and click **Check connections**. Every IO module and network device should show green.
3. Run tests one at a time from the left panel, or a whole section. Answer each dialog as you do the step; **Skip this item** records a SKIP.
4. **Stop test** ends a long test (polling, alarm logging, failover) early and keeps what was recorded.
5. Click **Build report** to write and open the Excel report for this run folder. **Start new run** begins a fresh folder, for example for a retest.

Results go to the `results` folder next to the app. Windows SmartScreen may warn about an unsigned app the first time it runs; choose *More info* then *Run anyway*, or have IT sign or allow it.

## PQMs and breakers (read and verify only)

The config's **Modbus_Devices** tab lists PQM1–PQM3 (Schneider PM8000) and the ABB breakers MAIN-A, AUTOXFMR, ACC-UPS and MAIN-B. The tool reads them over Modbus TCP and **never writes to them**: no setting changes and no open/close commands. Program the PQMs with Schneider ION Setup (configure PQM1, save a template, load it on PQM2 and PQM3; setting each meter's name to PQM1/PQM2/PQM3 lets the tool catch swapped IP addresses) and set breaker protection in ABB Ekip Connect per the coordination study; this tool then proves the result.

| Step | Command | What it checks | Pass rule |
|---|---|---|---|
| 2.7 Communication | `device-comms` | Every defined register is readable, then polls all devices | All registers read, 0 failed polls |
| 2.8 Configuration readback | `device-config` | Reads each setting (volts mode, PT/CT ratios, nominals, firmware) and compares it to Device_Expected | Matches; settings with no expected value are recorded for the report |
| 2.9 Measurements | `device-meas` | Reads V, I, kW, PF, Hz while the tech enters the reference reading from a test set or handheld meter | Error within `device_meas_tol_pct` (or the point's `tolerance_pct`) |
| 2.10 Status | `device-status` | Reads breaker open/closed/tripped and racked position; the tech confirms the breaker shows the same. Optionally the tech operates the breaker by its normal control and the tool confirms the change | Matches the equipment |

**Register addresses come from the manufacturers' Modbus maps** and go on the **Device_Points** tab, once per device type:
- PM8000: done. All rows are filled in from Schneider's `PM8000_Modbus_Map.xlsx` (Release 1 FW), and each row's description shows the map register number. Besides the measurements, the tool reads the meter name, model and firmware, volts mode, PT and CT primary/secondary, nominal frequency (expected 60 Hz), and CT/PT polarity (expected Normal). Enter each PQM's volts mode and PT/CT ratios on **Device_Expected** from the drawings; anything left blank is still read and recorded in the report.
- ABB: fill in the breaker status registers from ABB's *System Interface* document for your Ekip trip unit, then remove the `#`.

Addresses are 0-based (register number minus 1 if the map counts from 1). Set the data type, word order and any status bit as the map shows. Put each PQM's expected PT/CT ratios on **Device_Expected**. SCADA checks can use device values too: write the tag as `PQM1.V_AB`.

## 1. One-time setup (command line)

1. Install Python 3.10 or newer.
2. In this folder, run `pip install -r requirements.txt`.
3. Install MOXA **ioSearch / ioAdmin**. It's needed for the channel-configuration check and config exports on the ioLogik E1210.
4. Test the install without any hardware:
   ```
   python run_tests.py all --simulate --auto --duration 5 --new-run
   ```
   This produces a sample report marked **SIMULATION – NOT VALID TEST DATA**. `sample_output/` holds one.

## 2. Fill in the config (once per skid design, then per serial)

Open `config/skid_config.xlsx`. Yellow cells are inputs. Replace every EXAMPLE row, and confirm every value marked **PLACEHOLDER**:

| Tab | What you need | Where it comes from |
|---|---|---|
| Skid_Info | Project, serial, owner, test plan rev | Job file |
| Settings | Tolerances and limits (AI accuracy, noise, failover ms, ping, continuity) | Spec / approved submittal. Agree with the CxA |
| IO_Modules | ioLogik E1210 IP and firmware | Submittal, ioSearch |
| IO_Points | Tag, channel, alarm flag and normal state for each DI | IO list. On the E1210, DI-00 to DI-15 are `discrete_input` addresses 0–15 |
| Network_Devices | Every IP device on the skid network | Network drawing / IP schedule |
| Switches / Switch_Ports | Switch IPs, SNMP community, SSH user, approved config file, expected port status, speed, PVID, tagged/untagged VLANs. Ports use Junos names (ge-0/0/0, mge-0/0/0, xe-0/2/0) | Switch config / network drawing |
| Redundant_Links | Each ring or redundant link to pull, and a monitor tag that must stay reachable | Network drawing |
| SPDs | SPD list, max lead length, alarm DI tag, how to simulate a fault | SPD cut sheets / wiring diagrams |
| SCADA_Points | Points to verify on the SCADA screen, with the max update time | SCADA points list |

Then run `python run_tests.py check-config` and fix anything it lists.

For a new serial number of the same design, copy the config, change `Skid Serial` and any IPs, and you're done.

**Hardware notes for this skid (ioLogik E1210 + Juniper EX4100-H):**
- The E1210 is a 16-channel digital input module. It has no outputs or analog channels, so tests 1.1.5-1.2, 1.3 and 1.4 are recorded as N/A unless other IO is added.
- The Juniper switches are read two ways. Port status comes over SNMP v2c, which must be enabled read-only (for example `set snmp community <name> authorization read-only` plus a `clients` restriction to the test laptop). The version, running config and VLAN membership come over SSH with a normal, non-root Junos login. The password is asked for at run time and never saved. You can set `LVSS_SWITCH_USER` and `LVSS_SWITCH_PASSWORD` environment variables instead.
- Put each switch's approved config in `config/approved/` in Junos `display set` format (see the README in that folder). The tool compares it line by line with the running config.

## 3. Before test day

- Run `python run_tests.py checklists`. It prints blank SPD, IO, SCADA, and failover sheets from the config, which work as a paper backup and a witness copy.
- Equipment: laptop with a static IP on the skid subnet, loop calibrator (mA / V source and measure), DMM, LOTO kit, patch cables, camera.
- Export and save the approved configs for the switches and ioLogik modules. You'll compare against these.

## 4. Test-day procedure

Every command appends to the same run folder for this serial. Pass `--tech "Name"` so the report shows who ran each step. Use `--new-run` to start a fresh folder, for example for a retest.

### 1.1.5-1 MOXA IO (`python run_tests.py io`, or one step at a time)

| Step | Command | What the tech does | Pass rule |
|---|---|---|---|
| 1.1.5-1.5 Channel config | `channel-config` | Opens each module in ioAdmin, compares it to the list printed on screen, enters the firmware version, and saves a config export into the run folder | Reachable, firmware matches, settings match |
| 1.1.5-1.1 Digital inputs | `di` | Operates each input ON, then OFF, when prompted | Each state is read correctly within `di_timeout_s` |
| 1.1.5-1.2 Digital outputs | `do` | Confirms it's safe first, then watches the field device. The tool always leaves the output OFF | Readback matches the command and the device responds |
| 1.1.5-1.3 Analog inputs | `ai` | Injects each test value with the calibrator | Error ≤ `ai_tolerance_pct_span`, and sample spread ≤ `ai_noise_limit_pct_span` (the "no signal distortion" check) |
| 1.1.5-1.4 Analog outputs | `ao` | Measures each output with the calibrator or DMM and types in the reading | Error ≤ `ao_tolerance_pct_span` |
| 1.1.5-1.7 Alarm logging | `alarms` | Triggers each alarm DI and returns it to normal while the tool logs | normal → alarm → normal is logged for every alarm point |
| 1.1.5-1.6 Ethernet polling | `poll` | Leaves it running (default 10 min) | Failed reads ≤ `poll_max_failures` (0 = no packet loss) |

### 1.1.5-2 Network switches (`python run_tests.py network`)

| Step | Command | What happens | Pass rule |
|---|---|---|---|
| 2.1 Config / firmware | `switch-config` | Reads the Junos version and running config over SSH, saves the config to the run folder, and compares it with the approved file | Version matches, 0 config differences |
| 2.3 Port status | `ports` | Reads admin and oper status and speed over SNMP | Matches Switch_Ports |
| 2.2 VLAN tagging | `vlan` | Reads access/trunk mode, VLAN members and native VLAN from the Junos config, then the tech confirms cross-VLAN isolation | Matches Switch_Ports, and isolation is verified |
| 2.4 Ping | `ping` | Pings every device and switch (default 100 each) | Loss and average latency within limits |
| 2.5 Failover | `failover` | Reads the monitor point every 10 ms. The tech presses Enter, pulls the link, and later presses Enter and reconnects it | Longest communication interruption ≤ `max_failover_ms` on both break and restore |
| 2.6 SCADA | `scada` | Tech changes each point at the field end and compares with SCADA (the tool shows the field value) | Match, within the update time |

**How failover is measured:** the tool timestamps every successful Modbus reply from a device on the far side of the link. The interruption is the longest gap between replies minus the normal sample interval. The raw numbers go to `failover_log.csv`. A Windows laptop adds a few ms of timing jitter, so repeat each link 3 times and record the worst. Set the limit to match the redundancy protocol in the network design (ERPS is designed for sub-50 ms recovery; RSTP recovery varies with topology), and agree it with the CxA.

### 1.1.5-3 SPDs (`python run_tests.py spd`)

For each SPD, the tool prompts for placement and mounting (3.1), measured lead length (3.2), status LED and monitoring module (3.3), and the aux alarm contact (3.4). For the aux contact, it reads the MOXA DI while the tech simulates a fault and restores it. If the contact isn't wired to the MOXA, the tech verifies it with a meter instead. Last comes continuity (3.5), which the tool only asks for after the tech confirms LOTO.

### Report

```
python run_tests.py report
```

This writes `<serial>_1.1.5_Functional_Test_Report.xlsx` into the run folder. It contains:

- **Cover:** skid info, summary by section, overall PASS/FAIL/INCOMPLETE, and sign-off lines for the manufacturer, the Owner witness, and the CxA
- **Results:** every line, color-coded
- **Open_Items:** failed and skipped items, with columns for corrective action and retest
- **Attachments:** the raw logs in the folder

Put photos, screenshots, and config exports in the run folder too, and they'll be listed under Attachments. Print or save to PDF from Excel for submittal.

## 5. Other commands

```
python run_tests.py -h              list all commands
python run_tests.py poll --duration 1800      longer polling test
python run_tests.py failover        re-run one step (appends to the same run folder)
python run_tests.py io --run-dir results/SN-0012_20261002_0815
```

The tool never mixes simulation data into a real run folder. If you try, it refuses.

## 6. Safety notes

- The tool writes to the network only in the DO and AO tests, and only after the tech confirms each output is safe to operate. Everything else is read-only.
- Get site or Owner approval before connecting the laptop to a live control network.
- Continuity and physical SPD checks are done de-energized under LOTO, per your company procedure.
