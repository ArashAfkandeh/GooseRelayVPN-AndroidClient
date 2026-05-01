# Changes Summary - Go TUN Bridge Integration

## Problem
- DNS resolution was happening locally in Iran, where most DNS servers are filtered
- Previous Kotlin fake DNS implementation didn't work reliably (Android bypassed it)
- User wanted solution like NekoBox but in a single app
- User cannot modify original Go code (needs to update it from upstream project)

## Solution
Created a **separate Go TUN module** that intercepts DNS and TCP packets at the TUN interface level, sending hostnames (not IPs) to the SOCKS5 proxy for remote DNS resolution.

## Files Changed

### 1. Build System
**`android/build_go_mobile.sh`**
- Added build step for TUN module
- Creates `tun.aar` separately from `gooserelayvpn.aar`

### 2. CI/CD
**`.github/workflows/android-ci.yml`**
- Added artifact upload for `tun.aar`

### 3. Android VPN Service
**`android/app/src/main/java/com/gooserelay/gooserelayvpn/service/GooseRelayVpnService.kt`**

**Removed:**
- Import statements for `FakeDnsServer` and `FakeDnsInterceptor`
- Kotlin fake DNS server initialization (lines ~213-237)
- Kotlin fake DNS cleanup code (lines ~393-398)
- Class variables `fakeDnsServer` and `fakeDnsInterceptor`

**Added:**
- Class variable `tunBridgeActive: Boolean`
- Go TUN bridge initialization when `fakeDnsEnabled = true`
- Uses reflection to call `tun.Tun.startTunBridge(fd, mtu, socksAddr)`
- Go TUN bridge cleanup in `stopVpn()`

**Modified:**
- VPN address: `172.19.0.1/30` (when fake DNS enabled)
- DNS server: `172.19.0.2` (when fake DNS enabled)
- Conditional logic: TUN bridge for fake DNS, standard tun2socks otherwise

## Go TUN Module (Already Existed)
Located in `mobile/tun/`:
- `tun_bridge.go` - Main TUN packet handler with DNS interception
- `tcp_handler.go` - TCP state machine and SOCKS5 client
- `tun_api.go` - Android-compatible API (gomobile)
- `tun_syscall.go` - System calls for file descriptors
- `go.mod` - Independent module definition

## Technical Details

### Network Configuration
**Standard Mode (Fake DNS OFF):**
- VPN Address: `10.0.0.2/32`
- DNS: Custom or default public DNS
- Bridge: Standard tun2socks

**Fake DNS Mode (Fake DNS ON):**
- VPN Address: `172.19.0.1/30`
- DNS: `172.19.0.2` (TUN bridge)
- Fake IP Range: `198.18.0.0/15`
- Bridge: Go TUN bridge with DNS interception

### Data Flow (Fake DNS Mode)
1. App queries DNS → TUN bridge intercepts
2. TUN bridge returns fake IP (198.18.x.x)
3. App connects to fake IP → TUN bridge intercepts TCP SYN
4. TUN bridge looks up real hostname for fake IP
5. TUN bridge connects to SOCKS5 with hostname (not IP)
6. Original Go client tunnels to VPS
7. VPS resolves DNS and connects to real server
8. Data flows bidirectionally through tunnel

### Why Reflection?
The VPN service uses reflection to call the TUN module:
```kotlin
val tunClass = Class.forName("tun.Tun")
val startMethod = tunClass.getMethod("startTunBridge", Long::class.java, Long::class.java, String::class.java)
startMethod.invoke(null, fd.toLong(), 1500L, "127.0.0.1:$socksPort")
```

This allows the TUN module to be optional. If `tun.aar` is not included, the app will log an error but won't crash.

## What Wasn't Changed

### Original Go Code (Preserved)
- `cmd/` - Command-line interface
- `internal/` - Core client logic
- `go.mod` - Main module dependencies
- `mobile/` - Mobile bindings (except `mobile/tun/`)

These files are **never modified** so you can update them from the upstream project without conflicts.

### Kotlin Fake DNS Files (Kept but Unused)
- `android/app/src/main/java/com/gooserelay/gooserelayvpn/dns/FakeDnsServer.kt`
- `android/app/src/main/java/com/gooserelay/gooserelayvpn/dns/FakeDnsInterceptor.kt`

These files still exist but are no longer imported or used. You can:
- Keep them as backup
- Delete them to clean up the codebase

## Testing

### Build Test
```bash
cd android
bash build_go_mobile.sh
# Should create both:
# - android/app/libs/gooserelayvpn.aar
# - android/app/libs/tun.aar
```

### Runtime Test
1. Enable "Fake DNS" in app settings
2. Connect VPN
3. Check logs for:
   - `Starting Go TUN bridge with DNS interception...`
   - `Go TUN bridge started (DNS will be resolved remotely)`
   - `[TUN-BRIDGE] Starting bridge`
   - `[TUN-DNS] Mapped hostname -> fake IP`
   - `[TCP] SOCKS5 connecting with hostname`

### DNS Test
```bash
# Before VPN: DNS may be filtered
nslookup twitter.com
# May return filtered/blocked IP

# After VPN with Fake DNS: DNS resolved remotely
nslookup twitter.com
# Should return real IP (resolved on VPS)
```

## Benefits

1. **No Go Code Modification** - Original Go code untouched
2. **Easy Updates** - Copy new Go code from upstream without conflicts
3. **Remote DNS** - Bypasses local DNS filtering in Iran
4. **Single App** - No need for NekoBox + GooseRelayVPN combo
5. **Automated Build** - GitHub Actions builds both AARs
6. **TCP Forwarding** - Complete solution with TCP state machine
7. **Reliable** - Operates at TUN packet level (can't be bypassed)

## Migration Path

If you want to update the original Go code in the future:

```bash
# 1. Backup TUN module
cp -r mobile/tun /tmp/tun-backup

# 2. Update original Go code
cd /path/to/upstream/project
git pull
cp -r cmd/ internal/ go.mod /path/to/GooseRelayVPN/

# 3. Restore TUN module (if accidentally overwritten)
cp -r /tmp/tun-backup /path/to/GooseRelayVPN/mobile/tun

# 4. Rebuild
cd /path/to/GooseRelayVPN/android
bash build_go_mobile.sh
```

## Commit Message Suggestion

```
feat: Integrate Go TUN bridge for remote DNS resolution

- Add separate TUN module build to android/build_go_mobile.sh
- Update GitHub Actions to build and upload tun.aar
- Replace Kotlin fake DNS with Go TUN bridge in VPN service
- Configure VPN with 172.19.0.1/30 when fake DNS enabled
- DNS queries intercepted and resolved remotely on VPS
- Bypasses local DNS filtering in Iran
- Original Go code remains unmodified for easy updates

Closes #[issue-number]
```

## Documentation Created

1. **`INTEGRATION_COMPLETE.md`** - Full architecture and implementation details
2. **`QUICK_START.md`** - Quick reference for building and using
3. **`CHANGES_SUMMARY.md`** - This file, detailed change log

## Next Steps

1. **Review changes** - Check the modified files
2. **Commit and push** - Let GitHub Actions build
3. **Download APK** - From Actions artifacts
4. **Test on device** - Enable Fake DNS and verify logs
5. **Verify DNS** - Test that DNS is resolved remotely
6. **Enjoy!** - Bypass DNS filtering in Iran 🎉
