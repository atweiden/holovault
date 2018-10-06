# Connecting To Captive Wifi Portal

## All Approaches: Partially connect to captive wifi portal

See also: [configure-wireless.md][configure-wireless.md]

This is needed no matter which approach you decide to take later on:

```sh
INTERFACE="wlan0"

# connect to captive wifi portal
wpa_supplicant -B -s -i "$INTERFACE" -c /etc/wpa_supplicant/wpa_supplicant.conf
wpa_cli
> add_network
> set_network 0 ssid "SSIDNAME"
> set_network 0 key_mgmt NONE
> enable_network 0
> save_config
> quit

# lease ip from captive wifi portal
dhcpcd "$INTERFACE"
```

Note: `key_mgmt NONE` is only necessary if the captive wifi portal can
be connected to without supplying a password. If a password is required,
supply it via `set_network 0 psk "PASSWORD"` as per usual.

## Approach A: Hijack your own GUI machine's active connection

The most universal approach.

Requires two internet-capable machines. One of them must have GUI support.

### Step 1. Fully connect to captive wifi portal via GUI machine

**Note the GUI machine's local IP address**

Later, you will set `ip_addr_spoof` to this value:

Operating System | How to obtain local IP address
---------------- | -----------------------------------------------
iOS              | Settings->General->About->Wi-fi Address
macOS            | `ipconfig getifaddr en0`
Linux            | `ip -o -4 route get 1 | awk '/src/ {print $7}'`

- {[mac][macfiles],[pac][pacfiles],[tty][ttyfiles],[void][voidfiles]}files
  - `localip`

**Note the GUI machine's MAC address**

Later, you will set `mac_addr_spoof` to this value:

Operating System | How to obtain MAC address
---------------- | -----------------------------------------------
iOS              | Settings->General->About->Wi-fi Address
macOS            | `ifconfig en0 ether | tail -n 1 | awk '{print $2}'`
Linux            | `ip -0 addr show dev $INTERFACE | awk '/link/ && /ether/ {print \$2}' | tr '[:upper:]' '[:lower:]'`

- {[mac][macfiles],[pac][pacfiles],[tty][ttyfiles],[void][voidfiles]}files
  - `macaddr`

### Step 2. Hijack GUI machine's active connection

```sh
# -----------------------------------------------------------------------------
# us
# -----------------------------------------------------------------------------
# e.g. wlan0
interface="$(ip -o -4 route show to default | awk '/dev/ {print $5}')"
# e.g. 192.168.10.1
gateway="$(ip -o -4 route show to default | awk '/via/ {print $3}')"
# e.g. 192.168.10.255
broadcast="$(ip -o -4 addr show dev "$interface" | awk '/brd/ {print $6}')"
# e.g. 192.168.10.151/24
ipmask="$(ip -o -4 addr show dev "$interface" | awk '/inet/ {print $4}')"
# e.g. 24
netmask="$(printf "%s\n" "$ipmask" | cut -d "/" -f 2)"
# e.g. 89:cd:a8:f3:b7:92
macaddress="$(ip -0 addr show dev "$interface" | awk '/link/ && /ether/ {print $2}' | tr '[:upper:]' '[:lower:]')"

# -----------------------------------------------------------------------------
# our spoof
# -----------------------------------------------------------------------------
# from GUI machine
ip_addr_spoof="192.168.10.115"
# from GUI machine
mac_addr_spoof="f3:c2:39:10:9e:b2"
ip link set "$interface" down
ip link set dev "$interface" address "$mac_addr_spoof"
ip link set "$interface" up
ip addr flush dev "$interface"
ip addr add "$ip_addr_spoof/$netmask" broadcast "$broadcast" dev "$interface"
ip route add default via "$gateway"
```

Credit: [@systematicat][@systematicat]

### Step 3: Disable GUI machine's wifi

Quickly disable GUI machine's wifi shortly after hijacking its connection.

Needed to prevent the router from undoing your hijacked connection.

Alternatively, attempt overpowering the GUI machine's wifi.

## Approach B: Submit captive wifi portal login form interactively

The wifi captive portal login page will likely need to work with
javascript disabled for this approach to succeed.

### Example: [lynx][lynx]

**For Linksys Smart Wi-Fi**

```sh
guest_login_page="192.168.3.1:10080/ui/dynamic/guest-login.html"
# e.g. https%3A%2F%2Fwww.apple.com%2Flibrary%2Ftest%2Fsuccess.html
url="$(echo "https://www.apple.com/library/test/success.html" | sed 's#:#%3A#g' | sed 's#/#%2F#g')"
# e.g. 68%3Aec%3Ac5%3Ac1%3Aa3%3A63
mac_addr="$(ip link show "$INTERFACE" | tail -n 1 | awk '{print $2}' | sed 's#:#%3A#g')"
# e.g. 192.168.3.144
ip_addr="$(ip -o -4 route get 1 | awk '/src/ {print $7}')"
lynx "${guest_login_page}?mac_addr=${mac_addr}&url=${url}&ip_addr=${ip_addr}"
```

### Example: [edbrowse][edbrowse]

*edbrowse* is a console-only web browser with javascript support. Launch
*edbrowse* in render mode like so:

```sh
edbrowse
```

```
:b 192.168.3.1:10080/ui/dynamic/guest-login.html?mac_addr=68%3Aec%3Ac5%3Ac1%3Aa3%3A63&url=https%3A%2F%2Fwww.apple.com%2Flibrary%2Ftest%2Fsuccess.html&ip_addr=192.168.3.144
```

Note: *edbrowse*'s JavaScript support wasn't detected by the Linksys
Smart Wi-Fi login page.

## Approach C: Submit captive wifi portal login form programmatically

Use a headless browser or programming language to submit captive wifi
portal login form.

### Example: [Nightmare][Nightmare]

**For Linksys Smart Wi-Fi**

```sh
mkdir linksys && cd linksys
npm install --save nightmare
vim linksys.js
```

Contents of `linksys.js`:

```js
const Nightmare = require('nightmare')
// do not render visible window
const nightmare = Nightmare({ show: false })

nightmare
  .goto('http://192.168.3.1:10080/ui/dynamic/guest-login.html?mac_addr=68%3Aec%3Ac5%3Ac1%3Aa3%3A63&url=https%3A%2F%2Fwww.apple.com%2Flibrary%2Ftest%2Fsuccess.html&ip_addr=192.168.3.144')
  .type('#guest-pass', 'ThePasswordForLinksysSmartWiFiCaptivePortalGoesHere')
  .click('#submit-login')
  .wait(7000)
  .evaluate(() => {
    return document.title;
  })
  .end()
  .then((title) => {
    console.log(title);
  })
  .catch(error => {
    console.error('Error:', error)
  });
```

```sh
node linksys.js
```

### Example: Python

```python
#!/usr/bin/python
import urllib
url = "http://login.nomadix.com:1111/usg/process?OS=http://bellevue.house.hyatt.com/en/hotel.home.html"
username = "{whatever}"
password = "{whatever}"
login_data = urllib.urlencode({'username': username, 'password' : password, 'submit':'loginform2'})
op = urllib.urlopen(url, login_data).read()
print op
```

credit: [/r/raspberry_pi][/r/raspberry_pi]

## Notes

Linux `ip` commands require pkg [iproute2][iproute2].

## See Also

- https://github.com/authq/captive-login
- https://github.com/systematicat/hack-captive-portals
- https://github.com/imwally/starbucksconnect


[configure-wireless.md]: configure-wireless.md
[edbrowse]: https://github.com/CMB/edbrowse
[iproute2]: https://wiki.linuxfoundation.org/networking/iproute2
[lynx]: https://invisible-island.net/lynx/
[macfiles]: https://github.com/atweiden/macfiles
[Nightmare]: https://www.nightmarejs.org/
[pacfiles]: https://github.com/atweiden/pacfiles
[/r/raspberry_pi]: https://www.reddit.com/r/raspberry_pi/comments/4li7za/connecting_to_an_open_hotel_wifi/d3nlfq2/
[ttyfiles]: https://github.com/atweiden/ttyfiles
[voidfiles]: https://github.com/atweiden/voidfiles
[@systematicat]: https://github.com/systematicat/hack-captive-portals