# Reserve an IP Address in the KPN Box

Bind a device to a fixed IP via DHCP reservation on the KPN Box 12, so it keeps
the same address across reboots without configuring a static IP on the device.

## Prerequisites

- Access to the KPN Box web UI: [http://192.168.2.254/](http://192.168.2.254/)
- Login credentials for the box (printed on the label, unless changed)
- The target address chosen against the [IP plan](../network/overview.md) to avoid
  collisions
- For a device not yet seen by the box: its MAC address

## Steps

1. Open [http://192.168.2.254/](http://192.168.2.254/) and log in.
2. Click **Thuisnetwerk** in the menu.
3. Click **IP-adres reserveren** in the submenu.
4. Select the device from the list. If it is not listed, choose **Nieuw apparaat
   niet in huidige lijst** and fill in the missing details (MAC address and the
   IP to reserve).
5. Confirm to save the reservation.

## Verify

- Reboot or reconnect the device and confirm it receives the reserved address.
- The reservation appears in the **IP-adres reserveren** list with the expected IP.
- Record the assignment in the [IP plan](../network/overview.md) so the address
  stays the single source of truth.
