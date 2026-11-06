# Alert Framework: Design Reference

Back to the [README](../README.md). To install the framework, see [INSTALL.md](../INSTALL.md).

## Contents

- [Adding a new alert](#adding-a-new-alert)
- [Alert lifecycle](#alert-lifecycle)
- [Status helper states](#status-helper-states)
- [Alert binary sensor schema](#alert-binary-sensor-schema)
- [Presence-based severity](#presence-based-severity)
- [Silencer (suppressor)](#silencer-suppressor)
- [Complex alert triggers](#complex-alert-triggers)
- [Acknowledgement](#acknowledgement)
- [Custom action buttons](#custom-action-buttons)
- [Badge count](#badge-count)
- [Platform notes (iOS and Android)](#platform-notes-ios-and-android)
- [Framework constants](#framework-constants)
- [Example dashboard card](#example-dashboard-card)
- [Known issues and limitations](#known-issues-and-limitations)
- [Future enhancements](#future-enhancements)
- [File structure](#file-structure)

## Adding a new alert

The first-time setup is in [INSTALL.md](../INSTALL.md). Once the framework is installed, each new alert takes four steps.

**1. Create the status helper.** Every alert except `info` needs an `input_select` helper with these options, in this order: `idle, skipped, suppressed, notified, actioned, acknowledged`. The entity ID convention is `input_select.alert_status_<tag>`. Either create it in the UI (Settings > Helpers > Create helper > Dropdown), or add an entry to `input_selects_alerts.yaml` (this needs a full restart):

```yaml
alert_status_garage_door_open:
  name: Alert Status Garage Door Open
  options: [idle, skipped, suppressed, notified, actioned, acknowledged]
  icon: mdi:alert
```

**2. Add a `use_blueprint` entry** to your alerts file in `template-sensors/`. `unique_id` goes on the `use_blueprint` entry, not inside `input`, and follows the convention `alert_<tag>`:

```yaml
- use_blueprint:
    path: mark1foley/alert_binary_sensor_blueprint.yaml
    input:
      trigger_entity: binary_sensor.garage_door_open_away
      trigger_state: 'on'
      severity: minor
      not_home_severity: major
      title: Garage Door Open
      message: The garage door has been open and no one is home
      tag: garage_door_open
      status_helper: input_select.alert_status_garage_door_open
      dashboard_url: /lovelace/security
  unique_id: alert_garage_door_open
```

**3. Add the sensor to the group.** Add `binary_sensor.alert_<tag>` to `binary_sensor.automated_alert_sensors` (Settings > Helpers > the group > edit members).

**4. Reload template entities** (Developer tools > YAML > Template entities). A full restart is only needed if the blueprint file itself has changed.

Optionally, assign the new sensor and helper to the "Alerts Framework" category so they are easy to find.

## Alert lifecycle

| Event | What happens | Badge |
|---|---|---|
| Condition activates | The binary sensor turns on and the count sensor increments. The State Change automation fires and checks `startup_complete`, the suppressor and `skip_first`. It computes the effective severity from `not_home_severity`. It then sets the status to `suppressed` or `skipped`, or calls `script.alert_notify` (which sets the status to `notified`). | Increments |
| Acknowledgement (notification button or dashboard) | `script.alert_clear` is called with `acknowledged=true`. The mobile tray is cleared and the status is set to `acknowledged` (`idle` for security). The Repeat automation skips the alert. The persistent notification remains (not applicable to security). | No change |
| Suppressor turned on while an alert is active | The Repeat automation sets the status to `suppressed` on its next cycle. Notifications pause. | No change |
| Suppressor turned off while an alert is active | The Repeat automation sees the `suppressed` status with `is_suppressed=false`. It notifies on the next interval, or immediately for minor and security alerts via the `home_changed` trigger. | No change |
| Home state changes | The Repeat automation is triggered immediately. It re-evaluates the effective severity of all active alerts with `not_home_severity` set, and notifies with the new severity if it changed. | No change |
| Condition resolves | The binary sensor turns off and the count sensor decrements. The State Change automation calls `script.alert_clear` with `acknowledged=false`. The persistent notification is dismissed and the status is reset to `idle`. | Decrements |
| Home Assistant restart | The Startup automation turns off `startup_complete`, waits 2 minutes, re-notifies all active alerts and turns `startup_complete` back on. The badge syncs again after 30 seconds. | Resynced |

## Status helper states

Each alert with a `status_helper` (an `input_select`) tracks its current state across six values. A status helper is **required for every alert except `info`**. Without one, an alert can re-notify on every change in the active count (see [Known issues](#known-issues-and-limitations)).

| Status | Meaning | Set by |
|---|---|---|
| `idle` | The alert is not active or has been fully resolved. Ready to fire again if retriggered. | `script.alert_clear` on resolve. Default state after a restart. |
| `skipped` | The alert is active but the first notification was deliberately skipped (`skip_first: true`). Waiting for the first repeat interval. | State Change automation when `skip_first` is true and the severity repeats. |
| `suppressed` | The alert is active but notifications are suppressed (home, suppressor on, `is_suppressible: true`). | State Change automation on activate, or the Repeat automation if the suppressor is turned on mid-alert. |
| `notified` | A notification has been sent. The alert is active and repeating (if the severity repeats). | `script.alert_notify`. |
| `actioned` | The user tapped a custom action button and the service call fired. Repeats continue until the condition resolves. | Acknowledge automation on an `ACTION1_` or `ACTION2_` event. |
| `acknowledged` | The user acknowledged the alert. Repeats stop. The persistent notification remains. | `script.alert_clear` with `acknowledged=true`. Not used for security severity, which uses `idle`. |

Status helpers must be created with all six options (see [Adding a new alert](#adding-a-new-alert)). Naming convention: `input_select.alert_status_<tag>`.

## Alert binary sensor schema

Every alert binary sensor carries the following attributes, enforced by the blueprint:

```yaml
automated_alert: true          # Self-documenting flag
trigger_entity: binary_sensor  # Entity driving this alert
trigger_state: 'on'            # State that activates the alert
severity: major                # info | minor | action | major | security | critical
not_home_severity: ''          # Optional severity override when no one is home
is_suppressible: true          # Can be suppressed by the silencer (default true)
skip_first: false              # Skip first notification, notify on first repeat
title: Garage Door Open        # Short notification heading
message: The garage door...    # Full alert message body
tag: garage_door_open          # Unique snake_case identifier
status_helper: input_select... # Status tracking helper (required except for info)
image_path: ''                 # Optional /local/ image path
dashboard_url: ''              # Optional Lovelace URL for notification tap
action_1_title: ''             # Optional custom action button 1 title (replaces Acknowledge)
action_1_service: ''           # Service to call when action button 1 is tapped
action_1_target: ''            # Target entity for action_1_service
action_2_title: ''             # Optional second custom action button title
action_2_service: ''           # Service to call when action button 2 is tapped
action_2_target: ''            # Target entity for action_2_service
```

The entity ID and friendly name are derived automatically from the tag:

```
tag: garage_door_open  ->  binary_sensor.alert_garage_door_open
```

Only `trigger_entity` is used in the state template. All other inputs are static attributes, evaluated once at load time. A full restart is required after blueprint file changes.

## Presence-based severity

The `not_home_severity` attribute lets the same alert behave differently depending on whether someone is home. When `binary_sensor.automated_alert_home` is `off`, the effective severity is overridden to `not_home_severity`.

Examples:

- `severity: minor`, `not_home_severity: major`: a gentle reminder when home, urgent when away.
- `severity: info`, `not_home_severity: security`: silent when home (you can see it), a security alert when away.
- `severity: action`, `not_home_severity: minor`: a task reminder when home, downgraded (no repeat) when away.

The Repeat automation is triggered immediately when `binary_sensor.automated_alert_home` changes state, so severity changes take effect without waiting for the next 5-minute cycle.

Interaction with the suppressor: a severity upgrade when away bypasses suppression automatically, because suppression requires `is_home=true`. So `is_suppressible` and `not_home_severity` work together cleanly: suppress when home, notify at the upgraded severity when away.

## Silencer (suppressor)

Alerts with `is_suppressible: true` (the default) can be suppressed when someone is home and the suppressor is active. The logic is `is_home AND suppressor_on AND is_suppressible`.

- `binary_sensor.automated_alert_home` is the presence abstraction. Create it as a template helper wrapping your own presence logic (for example `group.grp_family`). It defaults to home (true) if unavailable.
- `input_boolean.automated_alert_suppressor` is a toggle helper. Turn it on to suppress suppressible alerts. It is the single switch for this; there is no separate visitor mode.

`is_suppressible` applies to all severities, including critical. It defaults to true. Set it to false only for alerts that must always fire regardless of presence (for example smoke or flood). A typical use is suppressing camera person-detection alerts when a trusted person is home, while still receiving water and smoke alerts (`is_suppressible: false`).

When suppressed, the status helper is set to `suppressed`. The Repeat automation handles both minor and security severities for suppression recovery: when the suppressor turns off, the next 5-minute cycle (or the immediate `home_changed` trigger) notifies any alerts left in the `suppressed` state.

## Complex alert triggers

For complex conditions (numeric thresholds, multiple entities, latching logic), create an intermediary template binary sensor or `input_boolean` first, then use it as the `trigger_entity`.

Example: garage door open when no one is home.

```yaml
- binary_sensor:
    - name: alert_trigger_garage_door_open
      state: >
        {{ is_state('cover.garage_door', 'open') and
           is_state('group.grp_family', 'not_home') }}
```

Security alerts always use an `input_boolean` as the `trigger_entity`. An upstream automation turns it on when the security event occurs. The Done mechanism (tapping the icon on the dashboard card) turns off the `input_boolean`, which resolves the alert. Acknowledging resets the status to `idle` so the alert is ready to retrigger.

## Acknowledgement

Acknowledging is available for action, major, security and critical alerts, from two places:

- Mobile push: the Acknowledge button on the notification.
- Dashboard: the hold action on the card, calling `script.alert_clear` with `acknowledged: true`.

Acknowledging stops repeats, clears the mobile tray and sets the status to `acknowledged` (`idle` for security, which allows immediate retriggering). The persistent notification remains until the condition resolves. If the condition clears and re-activates, the status resets to `idle` and notifications resume.

## Custom action buttons

An alert can include up to two custom action buttons on the mobile notification. When defined, they replace the standard Acknowledge button.

- Tapping a button calls the configured service on the configured target entity.
- The status is set to `actioned`. Repeats continue until the condition actually resolves.
- The mobile tray is cleared after the action fires.
- If the condition resolves (for example the door closes), the alert auto-clears as normal.
- If the condition persists (the door fails to close), repeats keep reminding the user.
- The user can still acknowledge from the dashboard card (hold action) to stop repeats.

Example: a garage door close button.

```yaml
- use_blueprint:
    path: mark1foley/alert_binary_sensor_blueprint.yaml
    input:
      trigger_entity: input_boolean.alert_trigger_garage_open_away
      trigger_state: 'on'
      severity: major
      title: Garage Door Open
      message: The garage door is open and no one is home
      tag: garage_door_open_away
      status_helper: input_select.alert_status_garage_door_open_away
      action_1_title: Close Door
      action_1_service: cover.close_cover
      action_1_target: cover.garage_door
  unique_id: alert_garage_door_open_away
```

Action buttons appear on every severity that sends a mobile push. The `actioned` status is treated like `notified` by the Repeat automation, so repeats continue on their normal schedule.

## Badge count

The app badge count represents active (unresolved) alert conditions. It is a system health indicator, not a message counter. It is driven by `sensor.automated_alert_active_count`, which counts the members of `binary_sensor.automated_alert_sensors` that are `on`, excluding security alerts.

The badge increments when an alert sensor turns on and decrements when it turns off. Acknowledging has no effect, because the condition is still active. A startup sync resets the badge 30 seconds after a restart, and a periodic sync runs every 10 minutes in case iOS clears the badge when the app is opened.

The badge is **iOS only**. The sync automations send to `notify.badge_audience`, which must list iOS devices only (see the platform notes below).

## Platform notes (iOS and Android)

These notes come from testing with one iPhone and one Android phone (a budget Samsung on Wi-Fi only). Behaviour on other devices and Android versions may differ.

**iOS only**

- **Badge count.** The sync automations set the app icon badge with a silent `delete_alert` message. Android does not treat it as silent and shows the text. This is why they send to `notify.badge_audience`, which should contain iOS devices only. If nobody has an iPhone, leave the badge automations disabled.
- **Interruption levels.** `time-sensitive` and `critical` interruption levels are iOS features. Android ignores them, so on Android every severity arrives with the same urgency unless `importance` is set (see below).

**Android**

- **Channels.** The framework sets an Android notification channel per severity (`minor`, `action`, `major`, `security`, `critical`). iOS ignores it.
- **Importance is not set.** The channels are created at default importance, so Android notifications are not more prominent for higher severities. The Companion app supports an `importance` setting used with a channel. Per its documentation, a channel's importance is fixed after first use and later only lowering takes effect, so an existing channel may need to be deleted in the phone's notification settings before a change applies. Not implemented.
- **Delivery.** Direct, group and script notifications all reached the Android phone in testing, with delays of up to about 15 seconds. On the test phone, notifications did not reliably wake the screen or play a tone while the screen was off.
- **No badge number.** Android launchers may show a notification dot while an alert is in the tray. This is an operating-system behaviour and not something the framework sends. The dot goes when the notification is cleared, including by acknowledging.

**Clearing notifications**

- Clearing by tag worked on a locked iPhone via a direct action, a single notify group and `script.alert_clear`. One earlier clear did not take effect until the Companion app was opened, and has not been reproduced. The Companion documentation notes that clearing on iOS may require the app to have been used recently.
- `script.alert_clear` sends a clear to four groups at once, so each phone in all four groups receives four clear messages.

## Framework constants

These names are used across several framework files. Change them with care.

```
binary_sensor.automated_alert_sensors          # Group registry of all alert binary sensors
sensor.automated_alert_active_count            # Active alert count (excludes security)
input_boolean.automated_alert_startup_complete # Gates automation processing on startup
binary_sensor.automated_alert_home             # Presence abstraction (create as a template sensor)
input_boolean.automated_alert_suppressor       # Silencer toggle
input_text.automated_alert_default_dashboard   # Global default Lovelace URL
notify.minor_audience                          # Mobile push for minor
notify.action_audience                         # Mobile push for action
notify.major_audience                          # Mobile push for major
notify.security_audience                       # Mobile push for security
notify.critical_audience                       # Mobile push for critical
notify.badge_audience                          # Badge sync (iOS devices only)
```

## Example dashboard card

[`alert_dashboard_card.yaml`](../alert_dashboard_card.yaml) is a standalone example using `custom:auto-entities` and `custom:button-card` (both available through HACS). It displays active alerts with severity colour coding, the message and the status.

<!-- Add a screenshot here: ![Example dashboard card](images/alert_dashboard_example.png) -->

Features:

- Expands the `binary_sensor.automated_alert_sensors` group, so new alerts appear automatically.
- Filters to active (`on`) alerts only.
- Severity colour coding: critical purple, security dark blue, major orange, action yellow, minor amber, info green.
- Shows the alert title and message.
- Shows the status (`notified`, `skipped`, `suppressed`, `actioned`, `acknowledged`) at the top right.
- Tap marks the alert Done (turns off the trigger entity). Intended for action and security severities.
- Hold acknowledges (calls `script.alert_clear` with `acknowledged: true`).

Sorting by severity and showing or hiding buttons per severity would need a custom button-card template. This example shows all alerts with tap-to-done and hold-to-acknowledge for every severity.

## Known issues and limitations

- **`status_helper` is required but not enforced.** Alerts other than `info` need one. An alert without a helper can re-notify on every change in the active count. A guard in the State Change automation and a clearer blueprint description are planned.
- **`action` alerts and clearing.** `script.alert_clear` and the Acknowledge automation clear the minor, major, security and critical groups but not `notify.action_audience`, so the phone notification for an `action` alert is not cleared by the framework.
- **iOS clearing.** See the platform notes. Opening the Companion app cleared stale notifications in testing.
- **Android prominence.** `importance` is not set, so Android notifications are not louder for higher severities.
- **Image path.** `image_path` is supported in the blueprint and script. An upstream process must capture and store the image before the alert fires.
- **Slow-activating alerts.** Alerts using `trigger_state: unavailable` activate after a delay. The Repeat automation acts as a backstop.
- **Startup false alerts.** Integrations that briefly report `unavailable` before settling can cause false activations. Add a `for:` delay or use an intermediary template sensor.
- **Repeat timing.** The gap between the first and second notification may be shorter than the full repeat interval, depending on where in the 5-minute cycle the alert activated.
- **TTS.** It fires only for security severity, or when `tts_target` is explicitly provided.
- **Restarts.** Changes to the notify groups file or the blueprint file need a full Home Assistant restart.
- **Dynamic messages.** The `message` attribute is static (evaluated once at blueprint load). For dynamic content, use an intermediary template binary sensor.
- **Dashboard card gap.** An occasional gap between cards on a tablet layout when only a few alerts are active, suspected to be a CSS grid `align-content` issue.

## Future enhancements

- Dynamic messages: template expressions in the `message` attribute, evaluated at notification time.
- Android `importance` per severity.
- Enforce `status_helper` for non-info alerts.
- A custom integration with its own entity type, replacing the YAML implementation.

## File structure

```
config/
  configuration.yaml                    # includes below
  alert_notify_groups.yaml              # notify: !include alert_notify_groups.yaml
  input_selects_alerts.yaml             # input_select: !include input_selects_alerts.yaml
  scripts.yaml                          # contains alert_notify and alert_clear
  automations.yaml                      # contains the six Alert Framework automations
  template-sensors/
    binary-sensor-blueprint_alerts.yaml # alert definitions (use_blueprint entries)
  blueprints/template/mark1foley/
    alert_binary_sensor_blueprint.yaml  # alert binary sensor template blueprint

UI managed (Settings > Helpers):
  binary_sensor.automated_alert_sensors            # alert sensor group
  binary_sensor.automated_alert_home               # template helper (presence)
  sensor.automated_alert_active_count              # template helper (counter)
  input_boolean.automated_alert_startup_complete
  input_boolean.automated_alert_suppressor
  input_text.automated_alert_default_dashboard
```

Repository files used as sources for the above: `alert_notify_script.yaml`, `alert_clear_script.yaml`, `alert_automations.yaml`, `alert_notify_groups.yaml`, `alert_counter_sensor.yaml` (optional YAML alternative to the counter helper), `alert_dashboard_card.yaml`, `input_selects_alerts.yaml` and `binary-sensor-blueprint_alerts.yaml`.
