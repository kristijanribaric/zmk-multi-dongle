# zmk-multi-dongle

A ZMK module that lets the halves of a BLE split keyboard work with **more than one
dongle**. The halves connect to whichever paired dongle is powered on and in range.

Typical use: one dongle stays plugged into your home dock, a second one lives in your
laptop bag or office. Plug in either one and the halves connect to it. You don't need a key
press, a profile switch or a reflash.

## How it works

Stock ZMK split peripherals advertise _directed_ at the one central they bonded with
first, so they can only ever reconnect to that one dongle.

This module only changes the halves' advertising and connection policy:

- The halves always advertise _undirected_ with the ZMK split service UUID.
- A dongle that has already paired with the halves sees them and reconnects using its
  stored bond.
- A new dongle with free peripheral slots pairs with them, the same way as a first-time
  ZMK dongle pairing.
- The halves remember up to `CONFIG_BT_MAX_PAIRED` dongles. Pairing one more replaces
  the oldest (`BT_KEYS_OVERWRITE_OLDEST`).
- Advertising is fast for `CONFIG_ZMK_SPLIT_BLE_ANY_CENTRAL_FAST_ADV_S` seconds after
  a disconnect or boot, then drops to a slow interval (about 1 s) to save battery.

The module registers its own split peripheral transport and reuses ZMK's split GATT
service, so the wire protocol is unchanged and **dongles run stock ZMK**. The module
is only needed on the peripherals (the halves). ZMK picks the module's transport over
its stock BLE peripheral transport because the module's transport has a better
priority. The stock transport is compiled but never enabled.

## Requirements

- ZMK `main` with the pluggable split transport API (Zephyr 4.1 based ZMK).
- A BLE split with a dongle as the central (see the
  [ZMK dongle docs](https://zmk.dev/docs/hardware-integration/dongle)).

## Installation

Add the module to your `config/west.yml`:

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: kristijanribaric
      url-base: https://github.com/kristijanribaric
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml
    - name: zmk-multi-dongle
      remote: kristijanribaric
      revision: main
  self:
    path: config
```

Enable it in the **peripheral (half) `.conf` files only**:

```conf
CONFIG_ZMK_SPLIT_BLE_ANY_CENTRAL=y
# Number of dongles the halves remember
CONFIG_BT_MAX_PAIRED=2
```

Each dongle needs nothing beyond a normal ZMK dongle build. Build the same dongle
firmware for every dongle. They can be different boards, for example one XIAO
nRF52840 and one nice!nano.

## Pairing a second dongle

1. Flash the module-enabled firmware to both halves. They keep their existing bond,
   so no settings reset is needed.
2. If the new dongle was used before, flash `settings_reset` to it first so its
   peripheral slots are empty. Then flash the dongle firmware.
3. **Unplug the first dongle.** Power on the new dongle and the halves. They pair
   with the new dongle.
4. From now on, whichever dongle is powered gets the halves.

## Options

| Kconfig                                       | Default | Description                                                               |
| --------------------------------------------- | ------- | ------------------------------------------------------------------------- |
| `CONFIG_ZMK_SPLIT_BLE_ANY_CENTRAL`            | n       | Enable the module on a split peripheral.                                  |
| `CONFIG_ZMK_SPLIT_BLE_ANY_CENTRAL_FAST_ADV_S` | 30      | Seconds of fast advertising before falling back to slow advertising.      |
| `CONFIG_ZMK_SPLIT_BLE_ANY_CENTRAL_PRIORITY`   | 0       | Transport priority. Must sort before `CONFIG_ZMK_SPLIT_BLE_PRIORITY` (1). |

## Caveats

- **Two dongles powered at once:** whichever connects first wins. Switching between
  live dongles is not supported, so unplug the one you're not using.
- **Unknown dongles:** while the halves are not connected, any _unpaired_ ZMK dongle
  in range with free peripheral slots could pair with them. That would also push out
  the oldest bond. This is unlikely in practice, but worth knowing.
- **ZMK internals:** the module calls `zmk_split_transport_peripheral_bt_report_event()`
  from ZMK's split BLE service, which is not a public API. If ZMK changes it, the build
  fails at link time instead of misbehaving at runtime.
- **Connection widgets:** the stock `zmk_split_bt_peripheral_is_connected()` and the
  `zmk_split_peripheral_status_changed` event keep working, so peripheral display
  widgets such as nice!view show the connection status correctly.

## License

MIT
