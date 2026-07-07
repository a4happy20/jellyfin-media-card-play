# Jellyfin Media Card — Play

The Home Assistant playback layer for the
[**Jellyfin Media Card**](https://github.com/a4happy20/jellyfin-media-card). It provides
the script the card calls when you tap an item, plus the helpers that choose a target
player and volume, and an orchestrator that makes sure the player is ready before playing.

> **This is a Home Assistant configuration *package*, not an integration or a HACS
> add-on.** You copy the YAML into your own configuration and enable Home Assistant's
> packages feature. See the official docs:
> **[Configuration packages — Home Assistant](https://www.home-assistant.io/docs/configuration/packages/)**.

<br>

Companion repos:
- [jellyfin-media-card](https://github.com/a4happy20/jellyfin-media-card) — the Lovelace card
- [jellyfin-media-card-sensors](https://github.com/a4happy20/jellyfin-media-card-sensors) — the data backend

<br>

## Requirements:

[Jellyfin integration](https://www.home-assistant.io/integrations/jellyfin)

The integration will provide the media player/s that we can use to send the media to.

(Optional)
[HASS.agent](https://www.hass-agent.io/2.2/getting-started/installation/)

Can be used to control a windows pc.

<br>

## The `play_script`

jellyfin-media-card `play_script` option should point at:

```
script.jellyfin_play_episode_custom_card
```

When you tap an item, the card calls that script with the item's ID as `episode_id`.

<br>

## What it creates

| Entity / script | Type | Purpose |
|-----------------|------|---------|
| `input_select.jellyfin_media_player` | helper | Which media player to play on |
| `input_number.jellyfin_volume` | helper | Default playback volume (0–100%) |
| `script.jellyfin_play_episode_custom_card` | script | **Card entry point** — ensures the player is ready, then plays |
| `script.jellyfin_ensure_player` | script | Powers on / launches the selected player |
| `script.jellyfin_play_episode` | script | Plays an episode by ID, sets volume, warns if playback didn't start |

## Setup

<br>

### Simple setup
High level overview:

    • add the package to home assistant
    • adapt the script to your setup

<br>

## 0. Enable packages (Optional)
<details>
  <summary>Package Setup</summary>

<br>

  > See the official docs: **[Configuration packages — Home Assistant](https://www.home-assistant.io/docs/configuration/packages/)**.

<br>

In your `configuration.yaml`, tell Home Assistant to load a `packages` folder:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Be sure to create the packages folder `config/packages/jellyfin_play_episode_custom_card.yaml`
(If you already have a `homeassistant:` block, just add the `packages:` line under it.)
Alternatively, include this one file directly:

```yaml
homeassistant:
  packages:
    jellyfin_play_episode_custom_card: !include jellyfin_play_episode_custom_card.yaml
```

</details>

<br>

## 1. Add the package file

Copy `jellyfin_play_episode_custom_card.yaml` into your `config/packages/jellyfin_play_episode_custom_card.yaml` folder.

> You don't need to setup packages if you don't want to.
> Instead you would just put the REST sensors and Template sensors
> where they belong in your setup.

load and play out of the box, also copy `device_readiness_stubs.yaml` (see below).

<br>

### 3. Adapt it to your setup

A few values are specific to your hardware and must be changed:

- **Media players** — edit the options in `input_select.jellyfin_media_player`
  (`media_player.jellyfin_brave`, `media_player.jellyfin_samsung_smart_tv` are examples).

```yaml
input_select:
  jellyfin_media_player:
    name: Jellyfin Media Player
    options:
      - "media_player.jellyfin_brave"        # your jellyfin media_player entity
```

- **Notification target** — replace `notify.mobile_app_YOUR_PHONE` (two places) with your
  own mobile app notify service, or remove those notify steps.

```yaml
script:
  jellyfin_play_episode_custom_card:
    alias: "Jellyfin Play Episode (custom card)"
    fields:
      episode_id:
        required: true
    sequence:
      - action: script.jellyfin_ensure_player
        response_variable: rdy
      - if: "{{ not (rdy.ready | default(false)) }}"
        then:
          - action: notify.mobile_app_a4happy20_s_iphone_7        # your mobile device to send a notification
            data:
              message: "Jellyfin: player not ready."
          - stop: "player not ready"
      - action: script.jellyfin_play_episode
        data:
          episode_id: "{{ episode_id }}"
```
```yaml
script:
  jellyfin_open_browser:
    sequence:
      - variables:
          player: "{{ states('input_select.jellyfin_media_player') }}"
      - repeat:
          while:
            - condition: template
              value_template: "{{ states(player) in ['unavailable','unknown'] }}"
            - condition: template
              value_template: "{{ repeat.index <= 3 }}"
          sequence:
            - action: mqtt.publish
              data:
                topic: "homeassistant/button/A4HAPPY20/jellyfin_start_brave/action"
                payload: 'cmd /c start "" "http://YOUR_JELLYFIN_HOST:8096/web/"'
            - wait_template: "{{ states(player) not in ['unavailable','unknown'] }}"
              timeout: "00:00:30"
              continue_on_timeout: true
      - variables:
          ok: "{{ states(player) not in ['unavailable','unknown'] }}"
      - if:
          - condition: template
            value_template: "{{ not ok }}"
        then:
          - action: notify.mobile_app_a4happy20_s_iphone_7        # your mobile device to send a notification
            data:
              message: "Jellyfin: browser session never came online."
      - delay: "00:00:03"
      - variables:
          result:
            ready: "{{ ok }}"
      - stop: "done"
        response_variable: result
```

<br>

### 4. Handle the device-readiness scripts (important)

`jellyfin_ensure_player` calls scripts that turn on and launch *your* specific devices:
`ensure_pc_is_on`, `ensure_speakers_are_on`, `jellyfin_open_browser`,
`ensure_pc_player_is_on`, `turn_on_samsung_tv`, `samsung_tv_launch_jellyfin`. These are
**not** part of the main package because they depend entirely on your hardware.

You have three options:

1. **Use the stubs** — copy `device_readiness_stubs.yaml` into `config/packages/`. It
   defines all six as no-op scripts that report "ready", so playback works when your
   device is already on. Replace each stub with real logic (Wake-on-LAN, turning on a TV,
   launching a browser, etc.) as you go.
2. **Write your own** scripts with those names.
3. **Skip readiness** — remove the `script.jellyfin_ensure_player` call from
   `jellyfin_play_episode_custom_card` if your player is always on.

### 5. Check config and restart

Developer Tools → YAML → **Check Configuration** (or `ha core check`), then restart Home
Assistant.

## How the play flow works

1. The card calls `jellyfin_play_episode_custom_card` with `episode_id`.
2. That runs `jellyfin_ensure_player`, which — based on
   `input_select.jellyfin_media_player` — powers on and launches the chosen player and
   returns whether everything is ready.
3. If ready, `jellyfin_play_episode` runs `media_player.play_media` for the episode, waits,
   sets the volume from `input_number.jellyfin_volume`, and — if playback still hasn't
   started after 30s — sends a fallback notification.


<br>

## Using it with the jellyfin-media-card

Once `script.jellyfin_play_episode_custom_card` exists, add the sensor entity to the card in the dashboard:

```yaml
type: custom:jellyfin-media-card
play_script: script.jellyfin_play_episode_custom_card
```

<br>

Full card options are documented in the
[Jellyfin Media Card README](https://github.com/a4happy20/jellyfin-media-card).

<br>

## License

Licensed under the [GNU General Public License v3.0](LICENSE) — the same license as the
rest of the Jellyfin Media Card project.
