# Alert Framework – Manual Installation Guide

Manual install for a fresh Home Assistant instance. Allow ~30 minutes plus two restarts.
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
| `alert_counter_sensor.yaml` | merge into your `template:` config |
| `notify_groups.yaml` | `config/notify_groups.yaml` |
| `alert_notify.yaml`, `alert_clear.yaml` | merge into `scripts.yaml` |
| `alert_automations.yaml` | merge into `automations.yaml` |
| `input_selects_alerts.yaml` | example status helpers (`input_select:`) |
| `low_battery_alerts.yaml`, `templates_example.yaml` | examples of alert entries |
| `alert_dashboard_card.yaml` | dashboard card (step 12) |

The blueprint folder name `mark1foley` must match the `path:` used in every `use_blueprint` entry.

## Step 1 – Install the blueprint

Copy `alert_binary_sensor_blueprint.yaml` to `config/blueprints/template/mark1foley/`.
A full HA restart is needed whenever this file changes (reloading templates is not enough).

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

1. Copy `notify_groups.yaml` into `config/` and edit the device names
   (`mobile_app_<your_device>`; find yours under Developer tools → Actions → search "notify.mobile_app").
2. In `configuration.yaml`: `notify: !include notify_groups.yaml`
   (or merge into your existing `notify:` section).

This creates `notify.minor_audience`, `notify.major_audience`, `notify.security_audience` and
`notify.critical_audience`. Each severity goes only to its own group, so list a device in every
group it should hear. Group notify platforms need a **full restart**.

## Step 4 – "Someone is home" sensor

Create a template binary sensor `binary_sensor.automated_alert_home` that is **on** when someone is home:

```yaml
template:
  - binary_sensor:
      - name: Automated Alert Home
        unique_id: automated_alert_home
        state: "{{ is_state('group.family', 'home') }}"   # replace with your own logic
```

Use whatever presence logic you already trust (person entities, a group, etc.). If this entity is
missing or unavailable the framework assumes **home**.

## Step 5 – Active alert counter

Merge `alert_counter_sensor.yaml` into your `template:` config. It creates
`sensor.automated_alert_active_count` (counts active alerts, excluding security), which triggers the
State Change automation and drives the app badge.

## Step 6 – Scripts

Add both scripts from `alert_notify.yaml` and `alert_clear.yaml` to `scripts.yaml`
(the top-level `script:` key in those files is for packages/`configuration.yaml`; in `scripts.yaml`
paste only the `alert_notify:` and `alert_clear:` entries). Resulting entities:
`script.alert_notify`, `script.alert_clear`.

## Step 7 – Automations

Paste the six automations from `alert_automations.yaml` into `automations.yaml`:
State Change, Repeat, Acknowledge, Badge Sync, Startup, Periodic Badge Sync (ids start `alert_framework_`).

## Step 8 – Restart

Full restart (Developer tools → YAML → Restart), not just a reload. Then turn
`input_boolean.automated_alert_startup_complete` **on** once; afterwards the Startup automation manages
it (it turns it off on boot and back on after a 2-minute delay, so alerts don't flood on restart).

**Check:** the four `notify.*_audience` services exist; `script.alert_notify` and `script.alert_clear` exist;
the six automations are enabled; `sensor.automated_alert_active_count` reads 0.

## Step 9 – Status helpers (one per alert)

Every alert except `info` needs a Dropdown helper with these options **in this order**:

`idle, skipped, suppressed, notified, actioned, acknowledged`

Entity ID convention: `input_select.alert_status_<tag>`. For many alerts, define them in YAML:
`input_select: !include input_selects_alerts.yaml` (see the file for the format; full restart needed).

> The status helper is effectively **mandatory for all non-info alerts**. Without one, the alert
> can re-notify on every change in the active count. A guard for this is planned but not yet implemented.

## Step 10 – Add each alert

For each alert add a `use_blueprint` entry to your `templates.yaml` (see `templates_example.yaml`):

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
