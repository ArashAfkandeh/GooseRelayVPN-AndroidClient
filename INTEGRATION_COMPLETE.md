# Go TUN Bridge Integration - Complete

## Overview
This document describes the integration of the Go TUN bridge module for DNS interception in GooseRelayVPN. The solution allows DNS resolution to happen on the VPS (remote) instead of locally, bypassing DNS filtering in Iran.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Android App                                                 │
│                                                             │
│  ┌──────────────┐         ┌─────────────────┐             │
│  │ VPN Service  │────────▶│ Go TUN Bridge   │             │
│  │ (Kotlin)     │  TUN FD │ (Separate .aar) │             │
│  └──────────────┘         └─────────────────┘             │
│                                   │                         │
│                                   │ DNS Query               │
│                                   ▼                         │
│                           Fake IP (198.18.x.x)             │
│                                   │                         │
│                                   │ TCP to Fake IP          │
│                                   ▼                         │
│                           Resolve Hostname                  │
│                                   │                         │
│                                   ▼                         │
│  ┌──────────────┐         ┌─────────────────┐             │
│  │ Original Go  │◀────────│ SOCKS5 CONNECT  │             │
│  │ Client       │         │ with Hostname   │             │
│  │ (goose.aar)  │         └─────────────────┘             │
│  └──────────────┘                                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    │ Tunnel to VPS
                    ▼
            ┌───────────────┐
            │ VPS Server    │
            │ Resolves DNS  │
            └───────────────┘
```

## Key Changes

### 1. Build System (`android/build_go_mobile.sh`)
- Added TUN module build step
- Creates `tun.aar` separately from `gooserelayvpn.aar`
- Both AARs are copied to `android/app/libs/`

### 2. GitHub Actions (`.github/workflows/android-ci.yml`)
- Added artifact upload for `tun.aar`
- Both AARs are now built and uploaded in CI

### 3. VPN Service (`GooseRelayVpnService.kt`)

**Removed:**
- Kotlin `FakeDnsServer` and `FakeDnsInterceptor` imports
- Kotlin fake DNS initialization code
- Kotlin fake DNS cleanup code

**Added:**
- Go TUN bridge initialization when `fakeDnsEnabled = true`
- Uses reflection to call `tun.Tun.startTunBridge()`
- Proper cleanup in `stopVpn()`

**Modified:**
- VPN address: `172.19.0.1/30` (was `10.0.0.2/32`)
- DNS server: `172.19.0.2` (was `10.0.0.1`)
- Conditional logic: uses TUN bridge for fake DNS, standard tun2socks otherwise

## How It Works

1. **User enables "Fake DNS" in settings**
2. **VPN connects with special configuration:**
   - Client IP: `172.19.0.1/30`
   - DNS server: `172.19.0.2` (TUN bridge)
3. **App makes DNS query → TUN bridge intercepts it**
4. **TUN bridge returns fake IP (198.18.x.x)**
5. **App connects to fake IP → TUN bridge intercepts TCP**
6. **TUN bridge looks up real hostname for fake IP**
7. **TUN bridge connects to SOCKS5 with hostname (not IP)**
8. **Original Go client tunnels to VPS**
9. **VPS resolves DNS and connects to real server**

## Benefits

- **No modification to original Go code** - TUN module is separate
- **Remote DNS resolution** - Bypasses local DNS filtering
- **Single app solution** - No need for NekoBox + GooseRelayVPN
- **Easy updates** - Original Go code can be updated without conflicts
- **Builds on GitHub Actions** - Fully automated CI/CD

## Testing on GitHub Actions

Push your changes to GitHub and the CI will:
1. Build `gooserelayvpn.aar` (original Go code)
2. Build `tun.aar` (TUN bridge module)
3. Build the Android APK with both AARs
4. Upload artifacts

## What About the Kotlin Fake DNS Files?

The Kotlin fake DNS files (`FakeDnsServer.kt`, `FakeDnsInterceptor.kt`) are still in the repository but are no longer used. They can be:
- **Kept as backup** - In case you want to revert
- **Deleted** - To clean up the codebase

The Go TUN bridge is more reliable because it operates at the packet level inside the TUN interface, whereas the Kotlin implementation tried to run a separate DNS server that Android could bypass.

## Logs to Watch

When fake DNS is enabled, look for these log messages:
- `[TUN-API] Starting TUN bridge`
- `[TUN-BRIDGE] Starting bridge`
- `[TUN-DNS] Mapped hostname -> fake IP`
- `[TCP] New connection to fake IP`
- `[TCP] Resolved fake IP to hostname`
- `[TCP] SOCKS5 connecting with hostname`

## Next Steps

1. **Push to GitHub** - Let CI build both AARs
2. **Download APK** - Test on your device
3. **Enable Fake DNS** - In app settings
4. **Connect VPN** - Check logs for TUN bridge messages
5. **Test browsing** - DNS should be resolved remotely
