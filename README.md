# CoreWLAN

CoreWLAN (`/System/Library/Frameworks/CoreWLAN.framework`, public): Wi-Fi interfaces, scanning, joining, power. Plus its private companion `CoreWiFi` and the UI framework `CoreWLANKit` (linked by ControlCenter).

**Status: planned; today a shim lives in `SDK`** (`SDK/include/CoreWLAN`, `SDK/include/CoreWiFi`, `SDK/src/CoreWLAN/CoreWLAN_Generated.m`, `SDK/build-corewlan.sh`: generated headers and stubs). The daemon side exists in `d/bsd/airportd` (`WSAirportService`, `WSWiFiWpaBackend` = wpa_supplicant, `WSWiFiSimBackend`, `WSIEParser`). This repository will take over the framework from `SDK` when the work starts.

## Layer
Layer 2. Depends on: `Foundation`, `SystemConfiguration`, `libxpc`; talks over XPC to `airportd` (layer 4 daemon, never linked). Used by: ControlCenter (Wi-Fi tile, page, menu-bar item), Settings, `apps/topbar`.

## API the Control Center uses (`apps/controlcenter-cocoa/CCControls.m`, through the Objective-C runtime)
`+[CWWiFiClient sharedWiFiClient]`, `-interface` (the default `CWInterface`), `-powerOn`, `-setPower:error:`, `-ssid`, `-scanForNetworksWithName:error:` (an `NSSet` of `CWNetwork`, `-ssid`); joining needs `-associateToNetwork:password:error:`. **Not exercised: the Mac VM has no Wi-Fi hardware** (`CWWiFiClient.interface` is nil there, which is also why the real ControlCenter shows no Wi-Fi menu-bar item).
The real ControlCenter's XPC and entitlement surface for Wi-Fi (from its entitlements, `codesign -d --entitlements :-`): `com.apple.private.corewifi`, `com.apple.private.corewifi-xpc` (mach-lookup), `com.apple.wifi.manager-access`, `com.apple.wifi.autojoin`, `com.apple.wifi.user_agent_join`, `com.apple.private.wifi-ui.join-delegate`, `com.apple.wifip2pd` (mach-lookup). Our client must use the same service names for the daemon so that real Mac binaries linking CoreWLAN work unchanged.

## Plan
1. Move the generated headers/stubs from `SDK` here; keep `SDK` pointing at this repository.
2. Implement `CWWiFiClient`/`CWInterface`/`CWNetwork`/`CWConfiguration` over the XPC protocol of `airportd` (document the protocol in `docs/`), power on/off through `ifconfig wlanN up/down`, scan through `wlan(4)` (`ifconfig wlanN list scan`, `get80211`), join through wpa_supplicant (`WSWiFiWpaBackend`).
3. Event notifications (`CWEventDelegate`: power, SSID, link, scan cache) from devd (`system=IFNET`) and wpa_supplicant.
4. Verify against Apple: REA of CoreWLAN in the dyld cache, the real protocol of `com.apple.private.corewifi-xpc` observed on a Mac with Wi-Fi.

## Existing code to reuse
`SDK` (CoreWLAN, CoreWiFi), `d/bsd/airportd`, `apps/topbar/WifiPanel.m` (the earlier Wi-Fi panel), `apps/controlcenter-cocoa/CCPlatform.c` (`ccp_wifi_*` bindings with `ifconfig`).

## Rules for this repository
- Our code: BSD-2-Clause. Apple files are never copied here: reference material is read from the user's own Mac (`macdeps`), notes and measured values are written in our own words (`docs/`).
- No GPL/LGPL code. Upstream code, if any, only under `upstream/` with a `PROVENANCE.md`.
- Before writing a function: search the code index and `where-is.sh` (rule 3); do exactly what Apple does, with evidence recorded here (rule 4).
- Test on the FreeBSD VM; CI is the shared `hygiene` workflow (`pearOS-bsd/.github`).
