# Alert Framework – Manual Installation Guide

Manual install for a fresh Home Assistant instance. Allow ~30 minutes plus one full restart (step 8).
Work through the steps in order and tick off the checks at the end of each.

> **Status:** these steps are assembled from the current framework files. Steps marked
> **(unverified)** have not yet been tested on a clean instance – please report anything
> that doesn't match.

## Prerequisites

- Home Assistant with the **Companion app** on at least one phone (notification actions need it).
- A way to edit config files (File editor / Studio Code Server / Samba).
- Optional, for the dashboard card: HACS frontend cards **auto-entities** and **button-card**.
- A sensor/logic that says whether anyone is home (see step 4).

## Files in this package

| File | Goes to |
|---|---|
| `alert_binary_sensor_blueprint.yaml` | `config/blueprints/template/mark1foley/` |
| `alert_counter_sensor.yaml` | optional YAML alternative to the step 5 helper (has the icon template) |
| `alert_notify_groups.yaml` | `config/alert_notify_groups.yaml` |
| `alert_notify_script.yaml`, `alert_clear_script.yaml` | append to `scripts.yaml` |
| `alert_automations.yaml` | merge into `automations.yaml` |
| `input_selects_alerts.yaml` | example status helpers (`input_select:`) |
| `binary-sensor-blueprint_alerts.yaml` | your alert entries, in `config/template-sensors/` |
| `low_battery_alerts.yaml` | example alert entries (low battery) |
| `alert_dashboard_card.yaml` | dashboard card (step 12) |

The blueprint folder name `mark1foley` must match the `path:` used in every `use_blueprint` entry.

## Step 1 – Install the blueprint

From the SSH terminal add-on (the folder must exist first, curl won't create it):

```
mkdir -p /config/blueprints/template/mark1foley
curl -fsSL -o /config/blueprints/template/mark1foley/alert_binary_sensor_blueprint.yaml \
  https://raw.githubusercontent.com/mark1foley/ha_alerts/main/alert_binary_sensor_blueprint.yaml
head -3 /config/blueprints/template/mark1foley/alert_binary_sensor_blueprint.yaml
```

The last line should print `# Alert Binary Sensor Blueprint`. (`-f` makes curl fail on a wrong URL
instead of saving a "404" page.) Alternatively copy the file there by any other means.
HA only picks up a new or changed blueprint file after a full restart; step 8 covers this, so don't restart yet.

## Step 2 – Create the framework helpers

Settings → Devices & services → Helpers → Create helper. Entity IDs must match exactly:

| Type | Name | Resulting entity ID |
|---|---|---|
| Toggle | Automated Alert Startup Complete | `input_boolean.automated_alert_startup_complete` |
| Toggle | Automated Alert Suppressor | `input_boolean.automated_alert_suppressor` |
| Text | Automated Alert Default Dashboard | `input_text.automated_alert_default_dashboard` |
| Group → Binary sensor group | Automated Alert Sensors | `binary_sensor.automated_alert_sensors` |

- Suppressor: **on** = alerts flagged `is_suppressible` are held back while someone is home.
- Default dashboard: a path such as `/lovelace/alerts`, used when an alert sets no `dashboard_url`.
- The group starts empty (you add each alert to it in step 10). Set its mode to "any".
  **(unverified)** `groups_example.yaml` shows a YAML `group.` version; the framework references
  `binary_sensor.automated_alert_sensors`, so use the binary sensor group helper.

## Step 3 – Notification audiences

1. Download the file into `/config/` and edit the device names. Find yours under Developer tools →
   Actions → search "notify.mobile_app". Enter the name **without** the `notify.` prefix, e.g.
   `action: mobile_app_my_phone` (not `notify.mobile_app_my_phone`, which HA rejects as an invalid slug).
   To send to more than one device, add one `- action:` line per device under `services:`:
   ```
   curl -fsSL -o /config/alert_notify_groups.yaml \
     https://raw.githubusercontent.com/mark1foley/ha_alerts/main/alert_notify_groups.yaml
   ```
2. In `configuration.yaml`: `notify: !include alert_notify_groups.yaml`
   (or merge into your existing `notify:` section).

This creates `notify.minor_audience`, `notify.major_audience`, `notify.security_audience` and
`notify.critical_audience`. Each severity goes only to its own group, so list a device in every
group it should hear. Group notify platforms need a full restart; step 8 covers this, so don't restart yet.

## Step 4 – "Someone is home" sensor

Create a template helper whose state is **on** when someone is home:

1. Settings → Devices & services → Helpers → Create helper → Template → **Template a binary sensor**.
2. **Name:** `Automated Alert Home` (this gives the entity ID `binary_sensor.automated_alert_home`;
   check it under the entity's settings, and rename the entity ID if it differs).
3. **State template:** paste a template that returns true when someone is home, for example:

   ```
   {{ is_state('group.family', 'home') }}
   ```

   Replace `group.family` with your own presence logic (person entities, a group, etc.).
4. Leave device class and unit blank, then Submit.

Use whatever presence logic you already trust (person entities, a group, etc.). If this entity is
missing or unavailable the framework assumes **home**.

## Step 5 – Active alert counter

Counts active alerts (excluding security). It triggers the State Change automation and drives the app badge.

1. Settings → Devices & services → Helpers → Create helper → Template → **Template a sensor**.
2. **Name:** `Automated Alert Active Count` (entity ID should be `sensor.automated_alert_active_count`;
   check it and rename the entity ID if it differs).
3. **State template:**

   ```
   {{ expand('binary_sensor.automated_alert_sensors')
       | selectattr('state', 'eq', 'on')
       | rejectattr('attributes.severity', 'eq', 'security')
       | list
       | count }}
   ```

4. Leave unit of measurement, device class and state class blank (the form only offers a fixed list
   of units, and the framework doesn't use one), then Submit.
5. Optional: open the entity's settings and set an icon (for example `mdi:shield-alert`).
   The template helper form has no icon template, so the YAML version's icon that changes with the
   count (`mdi:shield-check` at 0, and so on) isn't available this way. It is cosmetic only; the
   framework doesn't use it. The `template:` YAML in `alert_counter_sensor.yaml` still has it if you want it.

**Check:** the sensor reads 0 (verified on an empty group).

### Organise: assign a category

Give everything the framework creates the same category so it's easy to find and manage:

1. Settings → Devices & services → **Entities** (the Helpers tab works too).
2. Select the framework helpers (suppressor, startup complete, default dashboard, home, active count,
   and the `Automated Alert Sensors` group), then use the **Add category** option in the selection
   menu and create a category called **Alerts Framework**.
3. Do the same later for the scripts (step 6) and automations (step 7), and for each alert sensor and
   status helper you add. Categories are set per list, so the Scripts and Automations pages each need
   their own "Alerts Framework" category.

Result on the Entities page, grouped by category:

![Alerts Framework helpers](alert_framework_helpers.png)

**(unverified)** I'm describing the category UI from how it's normally laid out; menu wording may differ in your HA version.

## Step 6 – Scripts

Each script file starts with its script key (`alert_notify:` / `alert_clear:`), with no `script:` line,
because `configuration.yaml` already has `script: !include scripts.yaml`.

- [alert_notify_script.yaml](https://github.com/mark1foley/ha_alerts/blob/main/alert_notify_script.yaml) → `script.alert_notify`
- [alert_clear_script.yaml](https://github.com/mark1foley/ha_alerts/blob/main/alert_clear_script.yaml) → `script.alert_clear`

**Easiest:** open each file, click **Copy raw file**, and paste it at the end of `scripts.yaml`
(File editor / SSH), then reload scripts (Developer tools → YAML → Scripts).

**Via the UI:** Settings → Automations & scripts → Scripts → Create script → New script → ⋮ →
Edit in YAML, then paste only the lines from `alias:` down, with the leading indent removed.
Set the entity ID to `script.alert_notify` / `script.alert_clear` if it differs.
**(unverified)** The UI route is not tested with these files.

## Step 7 – Automations

[alert_automations.yaml](https://github.com/mark1foley/ha_alerts/blob/main/alert_automations.yaml)
contains six automations (ids start `alert_framework_`): State Change, Repeat, Acknowledge,
Badge Sync, Startup, Periodic Badge Sync.

- **Easiest:** paste the whole list at the end of `automations.yaml` (File editor / SSH), then reload automations.
- **Via the UI:** the automation editor takes one automation at a time, so create six automations,
  each with ⋮ → Edit in YAML and one entry pasted.

## Step 8 – Restart

Full restart (Developer tools → YAML → Restart), not just a reload. Then turn
`input_boolean.automated_alert_startup_complete` **on** once; afterwards the Startup automation manages
it (it turns it off on boot and back on after a 2-minute delay, so alerts don't flood on restart).

**Check:** the four `notify.*_audience` services exist; `script.alert_notify` and `script.alert_clear` exist;
the six automations are enabled; `sensor.automated_alert_active_count` reads 0.

## Step 9 – Status helpers (one per alert)

Every alert except `info` needs a Dropdown helper with these options **in this order**:

`idle, skipped, suppressed, notified, actioned, acknowledged`

Entity ID convention: `input_select.alert_status_<tag>`. Two ways to create them:

**YAML (best for many alerts).** Download the template file, add one entry per alert, and include it:

```
curl -fsSL -o /config/input_selects_alerts.yaml \
  https://raw.githubusercontent.com/mark1foley/ha_alerts/main/input_selects_alerts.yaml
```

Entries look like this:

```yaml
alert_status_garage_door_open:
  name: Alert Status Garage Door Open
  options: [idle, skipped, suppressed, notified, actioned, acknowledged]
  icon: mdi:alert
```

Add this line to `configuration.yaml` (if you already have an `input_select:` section, merge the
entries into your existing file instead). HA accepts the template file with every entry still commented out:

```
input_select: !include input_selects_alerts.yaml
```

Adding or changing entries in this file needs a **full restart**, so for a first install add the file,
your entries and the include **before step 8** to keep it to a single restart.

**UI (best for one or two).** Settings → Helpers → Create helper → Dropdown, with the options above in that
order and the name `Alert Status <description>`. No restart needed.

> The status helper is effectively **mandatory for all non-info alerts**. Without one, the alert
> can re-notify on every change in the active count. A guard for this is planned but not yet implemented.

## Step 10 – Add each alert

Alert sensors are defined as template entities in YAML. The recommended layout is a folder with one
file per kind of template, so the alerts stay separate from your other template sensors.

If your configuration doesn't already load template YAML, create the folder and add this line to
`configuration.yaml`:

```
mkdir -p /config/template-sensors
```

```
template: !include_dir_merge_list template-sensors/
```

Download the template file from the repo into that folder, as in step 9, and add your alerts to it:

```
curl -fsSL -o /config/template-sensors/binary-sensor-blueprint_alerts.yaml \
  https://raw.githubusercontent.com/mark1foley/ha_alerts/main/binary-sensor-blueprint_alerts.yaml
```

Each file in the folder must be a YAML list, and this one is just a list of `use_blueprint` entries with no
`template:` key of its own. If you already use `template:` with a different include (a single
`templates.yaml`, say), keep it and put the entries there instead.
**(unverified)** If this is the first time `template:` appears in `configuration.yaml`, do it before the
step 8 restart rather than relying on a reload.

For each alert add a `use_blueprint` entry to `binary-sensor-blueprint_alerts.yaml`:

```yaml
- use_blueprint:
    path: mark1foley/alert_binary_sensor_blueprint.yaml
    input:
      trigger_entity: cover.garage_door
      trigger_state: 'open'          # quote 'on'/'off'
      severity: major                # info | minor | action | major | security | critical
      title: Garage Door Open
      message: The garage door has been open and no one is home
      tag: garage_door_open
      status_helper: input_select.alert_status_garage_door_open
  unique_id: alert_garage_door_open
```

Rules:
- `unique_id` goes on the `use_blueprint` entry (not inside `input`), convention `alert_<tag>`.
- The entity ID becomes `binary_sensor.alert_<tag>` and must be unique.
- Then add `binary_sensor.alert_<tag>` to the `binary_sensor.automated_alert_sensors` group
  (Helpers → the group → edit members) and reload template entities.

## Step 11 – Test

1. Trigger a test alert (e.g. a toggle as `trigger_entity`) with severity `minor`.
2. Expect: sensor `on`, status helper → `notified`, a push on the phone, count = 1.
3. Turn the trigger off: sensor `off`, status → `idle`, phone notification removed.
4. Try `major` and tap **Acknowledge**: status → `acknowledged`, repeats stop.
5. With suppressor **on** and `binary_sensor.automated_alert_home` on, a suppressible alert should
   show status `suppressed` and send nothing; turn home off and it should notify.

If nothing happens, check in order: startup_complete is on; the sensor is in the group; the status
helper entity ID matches; the automation traces (Settings → Automations → trace).

## Step 12 – Dashboard card (optional)

Install `auto-entities` and `button-card` via HACS, then paste `alert_dashboard_card.yaml` into a
dashboard card (raw configuration editor). It lists the group members and shows severity colour,
title, message and status, with tap = mark done and hold = acknowledge/clear.

## Known pitfalls

- Blueprint changes need a **full restart**; notify group changes need a full restart.
- `on`/`off` values must be quoted (`'on'`).
- Changing a tag renames the entity – update group membership and remove the old sensor.
- Status helpers created earlier may lack `suppressed`/`actioned` options; add them.
- iOS: a notification sent at `critical` severity may not clear after resolve (under investigation).
