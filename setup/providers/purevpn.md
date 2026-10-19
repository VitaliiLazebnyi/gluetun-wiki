# PureVPN

## TLDR

```sh
# OpenVPN
docker run -it --rm --cap-add=NET_ADMIN --device /dev/net/tun \
-e VPN_SERVICE_PROVIDER=purevpn \
-e OPENVPN_USER=abc -e OPENVPN_PASSWORD=abc \
-e SERVER_COUNTRIES=Netherlands qmcgaw/gluetun
```

```yml
version: "3"
services:
  gluetun:
    image: qmcgaw/gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - VPN_SERVICE_PROVIDER=purevpn
      - OPENVPN_USER=abc
      - OPENVPN_PASSWORD=abc
      - SERVER_COUNTRIES=Netherlands
```

⚠️ Wireguard requires an image built after this feature was added (`:latest`).

```sh
# Wireguard
docker run -it --rm --cap-add=NET_ADMIN --device /dev/net/tun \
-e VPN_SERVICE_PROVIDER=purevpn \
-e VPN_TYPE=wireguard \
-e PUREVPN_USERNAME=you@example.com -e PUREVPN_PASSWORD=abc \
-e SERVER_COUNTRIES=Netherlands \
-v /yourpath:/gluetun qmcgaw/gluetun
```

```yml
version: "3"
services:
  gluetun:
    image: qmcgaw/gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - VPN_SERVICE_PROVIDER=purevpn
      - VPN_TYPE=wireguard
      - PUREVPN_USERNAME=you@example.com
      - PUREVPN_PASSWORD=abc
      # Only if two-factor authentication is enabled:
      # - PUREVPN_TOTP_SECRET=JBSWY3DPEHPK3PXP
      - SERVER_COUNTRIES=Netherlands
      # Optional, see the city IDs section below:
      # - PUREVPN_CITY_ID=2902
    volumes:
      - ./gluetun:/gluetun
```

💁 Mount `/gluetun` as a volume with Wireguard, see [the Wireguard section](#wireguard) for why.

## Required environment variables

- `VPN_SERVICE_PROVIDER=purevpn`

### OpenVPN only

- `OPENVPN_USER`
- `OPENVPN_PASSWORD`

### Wireguard only

- `VPN_TYPE=wireguard`
- `PUREVPN_USERNAME` and `PUREVPN_PASSWORD`: your PureVPN account email and password, the ones you use on the PureVPN website. This is **not** the `purevpn0s...` VPN username. They can be omitted if `PUREVPN_SESSION_TOKEN` is set.
- `SERVER_COUNTRIES`: **exactly one** country, for example `United States`. Any known country name works, see [the city IDs](#city-ids) for the countries PureVPN offers with Wireguard.

## Optional environment variables

- `VPN_TYPE`: `openvpn` or `wireguard`, defaults to `openvpn`
- OpenVPN only
  - `SERVER_COUNTRIES`: Comma separated list of countries
  - `SERVER_REGIONS`: Comma separated list of regions
  - `SERVER_CITIES`: Comma separated list of cities
  - `SERVER_HOSTNAMES`: Comma separated list of server hostnames. Beware this is the narrowest filter, so if you set this to a single hostname and this hostname disappears from the Gluetun servers data due to an update, your container will no longer work until this filter is changed. I would suggest avoiding it unless you know this reliability risk.
  - `OPENVPN_PROTOCOL`: `udp` or `tcp`, defaults to `udp`
- Wireguard only
  - `PUREVPN_TOTP_SECRET`: base32 secret of the authenticator app, only needed if two-factor authentication is enabled, see [Login and two-factor authentication](#login-and-two-factor-authentication)
  - `PUREVPN_SESSION_TOKEN`: session token copied from a browser, to use instead of or in addition to the username and password, see [Session token](#session-token)
  - `PUREVPN_CITY_ID`: PureVPN city ID in the country set, for example `8778` for New York, see [City IDs](#city-ids). If empty, PureVPN picks a city.
  - `PUREVPN_SUBSCRIPTION_ID`: subscription ID, only needed if the account has more than one subscription. If empty, the first active subscription is used.
  - `PUREVPN_VPN_USERNAME`: VPN username (`purevpn0s...`), only needed if the subscription has more than one. If empty, the first one is used.
  - `WIREGUARD_PRIVATE_KEY`: if empty, Gluetun generates a key pair and keeps it in `/gluetun/purevpn-wireguard.json`
  - `WIREGUARD_PERSISTENT_KEEPALIVE_INTERVAL`: defaults to `0`, which uses the interval given by PureVPN. Any other value, for example `25s`, overrides it.

💁 The username, password, TOTP secret and session token can also be read from [Docker secret files](../advanced/docker-secrets.md): `/run/secrets/purevpn_username`, `/run/secrets/purevpn_password`, `/run/secrets/purevpn_totp_secret` and `/run/secrets/purevpn_session_token`, or the paths set with `PUREVPN_USERNAME_SECRETFILE`, `PUREVPN_PASSWORD_SECRETFILE`, `PUREVPN_TOTP_SECRET_SECRETFILE` and `PUREVPN_SESSION_TOKEN_SECRETFILE`.

⚠️ PureVPN assigns the server endpoint, its public key and the interface addresses, and uses no pre-shared key, so these are rejected at startup with Wireguard: `WIREGUARD_ADDRESSES`, `WIREGUARD_ENDPOINT_IP`, `WIREGUARD_ENDPOINT_PORT`, `WIREGUARD_PUBLIC_KEY` and `WIREGUARD_PRESHARED_KEY`. If PureVPN assigns no IPv4 address and [IPv6](../advanced/ipv6.md) is not supported, the connection attempt fails.

⚠️ With Wireguard, the server location is only selected with `SERVER_COUNTRIES` and `PUREVPN_CITY_ID`. The other `SERVER_*` filters (regions, cities, ISPs, names, numbers, hostnames, categories) are ignored, with a warning at startup.

## Wireguard

Gluetun generates the Wireguard configuration from your PureVPN account, the same way the [PureVPN member area](https://my.purevpn.com/v2/dashboard/manual-config) does. PureVPN removes unused Wireguard peers, so a new configuration is requested on every connection attempt, including reconnections after a failed health check.

⚠️ PureVPN has no official API for this. Gluetun uses the member area endpoints, which PureVPN can change at any time without notice.

### State file

Gluetun keeps the Wireguard key pair and the PureVPN login session in `/gluetun/purevpn-wireguard.json`, with permissions `0600`. Mount `/gluetun` as a volume so a restart does not need to log in again, and keeps the same key pair.

### Login and two-factor authentication

Gluetun logs in on its own the first time, and again whenever the stored session stops working. It first tries the stored session, and only uses the username and password if that session is no longer valid.

If two-factor authentication is enabled on your account, set `PUREVPN_TOTP_SECRET`. Gluetun generates the 6-digit code from it, and asks PureVPN to trust the device, which may let later logins skip the code. Only authenticator app (TOTP) two-factor authentication is supported. The TOTP secret is the text version of the QR code shown when you enable two-factor authentication, often shown behind "Can't scan the code?".

### Session token

If you do not want to give Gluetun your password or TOTP secret, log in once in a browser instead:

1. Log in to [my.purevpn.com](https://my.purevpn.com), and check "keep me signed in" if it is offered.
1. Open the browser developer tools, then *Application* (Chrome) or *Storage* (Firefox), then *Cookies*, then `https://auth.purevpn.com`.
1. Copy the value of the `fusionauth.sso` cookie into `PUREVPN_SESSION_TOKEN`.

Gluetun stores and refreshes the session itself. When the session finally expires, Gluetun falls back to the username and password if they are set; otherwise it fails and you need to set a new token.

### City IDs

To find the current city IDs:

1. Log in to the PureVPN member area.
1. Open the browser developer tools, then the *Network* tab.
1. Open the [manual configuration page](https://my.purevpn.com/v2/dashboard/manual-config). It loads `GET /v2/api/manual/configuration`, whose response lists the countries (`iso_code`) and their cities, each with a numeric `id`.

<!-- markdownlint-disable MD033 -->
<details>
<summary>City IDs snapshot (August 2025, may be outdated)</summary>

PureVPN may add, remove or renumber cities at any time.

| Country (`SERVER_COUNTRIES`) | Cities (`PUREVPN_CITY_ID`) |
| --- | --- |
| United States | Miami=7040, Phoenix=7190, Los Angeles=7344, Chicago=7904, Newark=8643, New York=8778, Houston=9259, Atlanta=16158, Washington DC=25099, Ashburn=26001, San Francisco=26002, Seattle=26021, Salt Lake City=26024 |
| United Kingdom | London=3147, Manchester=4218 |
| Australia | Sydney=65, Perth=72 |
| Germany | Frankfurt=2762 |
| Canada | Toronto=22 |
| Netherlands | Amsterdam=2902 |
| Hong Kong | Hong Kong=4300 |
| Belgium | Brussels=173 |
| France | Paris=2753 |
| Sweden | Stockholm=3005 |
| Japan | Tokyo=993 |
| Singapore | Singapore=5459 |
| Italy | Milano=3124 |
| Ireland | Dublin=713 |
| Luxembourg | Luxembourg=2864 |
| India | Mumbai=2797 |
| Austria | Vienna=2387 |
| Denmark | Copenhagen=5197 |
| Norway | Oslo=2917 |
| Romania | Bucharest=1290 |
| Russian Federation | Moscow=96 |
| Poland | Warsaw=3431 |
| Czech Republic | Prague=370 |
| Finland | Helsinki=2748 |
| Spain | Madrid=530 |
| Switzerland | Zurich=3021 |
| Latvia | Riga=1044 |
| Turkey | Istanbul=1385 |
| Lithuania | Vilnius=2863 |
| Greece | Athens=666 |
| Estonia | Tallinn=2746 |
| Bulgaria | Sofia=189 |
| Slovakia | Bratislava=1539 |
| Brazil | Sao Paulo=44 |
| South Africa | Johannesburg=2946 |
| Hungary | Budapest=701 |
| Korea | Seoul=4865 |
| Nigeria | Lagos=1145 |
| Moldova | Chisinau=1047 |
| Albania | Tirana=2335 |
| China | Beijing=265 |
| Vietnam | Hanoi=3062 |
| Afghanistan | Kabul=2334 |
| Algeria | Algiers=2336 |
| Angola | Benguela=6228 |
| Argentina | Buenos Aires=4091 |
| Aruba | Oranjestad=2355 |
| Bahamas | Freeport=1506 |
| Bahrain | Manama=3196 |
| Bangladesh | Dhaka=2389 |
| Barbados | Bridgetown=3299 |
| Bermuda | Hamilton=2395 |
| Bolivia | Sucre=1574 |
| British Virgin Islands | Road Town=6463 |
| Brunei Darussalam | Bandar Seri Begawan=1548 |
| Cayman Islands | George Town=2420 |
| Chile | Santiago=256 |
| Egypt | Cairo=513 |
| Monaco | Monaco=4568 |
| Oman | Salalah=26007 |
| Panama | Panama City=1218 |
| Philippines | Manila=1243 |
| Portugal | Lisbon=2929 |
| Puerto Rico | San Juan=3439 |
| United Arab Emirates | Dubai=3049 |
| Serbia | Belgrade=26004 |

</details>
<!-- markdownlint-enable MD033 -->

### Kill switch

All traffic stays blocked until the VPN is up, including to the PureVPN website. To fetch the configuration, Gluetun resolves `auth.purevpn.com` and `my.purevpn.com` using DNS over HTTPS, and opens one narrow temporary firewall rule per HTTPS request, only to these hosts and over IPv4. Each rule is removed once its request is done.

### Retries

Every error is retried, with a delay starting at 15 seconds and doubling after each failure. The delay goes back to 15 seconds after a successful connection. To stop retries with wrong credentials, fix the settings.

A start that logs in to PureVPN again can take about 30 seconds, longer than the Docker health check start period of the image. If other containers wait for Gluetun to be healthy (`depends_on` with `condition: service_healthy`), give the health check more time:

```yml
    healthcheck:
      start_period: 60s
```

### Errors

| Error | Meaning |
| --- | --- |
| `PureVPN rejected the account username or password` | Wrong credentials, shown with PureVPN's own message |
| `two-factor authentication is enabled on the PureVPN account but no TOTP secret is set` | Set `PUREVPN_TOTP_SECRET` or `PUREVPN_SESSION_TOKEN` |
| `PureVPN rejected the two-factor code, check the TOTP secret and the system clock` | Wrong TOTP secret, or the system clock is off |
| `PureVPN requires a captcha to log in, log in once from a browser or set a session token` | Too many logins. Log in once from a browser, or use a [session token](#session-token) |
| `PureVPN account username and password are not set` | The session token expired or is not usable, and no username and password are set. After an expired session, it is prefixed with `PureVPN login session expired and` |
| `request blocked by the PureVPN website protection` | Cloudflare blocked the request, see [Known limitations](#known-limitations) |
| `no active PureVPN subscription found` | The account has no active subscription |
| `PureVPN subscription not found` | `PUREVPN_SUBSCRIPTION_ID` or `PUREVPN_VPN_USERNAME` matches no subscription; the error lists the ones available |
| `unexpected response from PureVPN` | PureVPN probably changed its website, please report it |

### Known limitations

The PureVPN member area is behind Cloudflare, which can challenge or block clients that are not browsers, regardless of the account. Gluetun then fails with `request blocked by the PureVPN website protection`. Retrying later can help, and there is no workaround in Gluetun.

## Servers

To see a list of OpenVPN servers available, [list the VPN servers with Gluetun](../servers.md#list-of-vpn-servers).
