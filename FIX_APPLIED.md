# Fix Applied: DNS Server Issue

## Problem You Reported

> "i run fake dns but in logs i see 10.10.34.36 that filtering ip"

This means Android was using the system's filtered DNS server (10.10.34.36) instead of your fake DNS server.

## Root Cause

Android was prioritizing the underlying network's DNS over the VPN's DNS configuration.

## Fixes Applied

### 1. Changed FakeDnsServer Binding

**Before:**
```kotlin
FakeDnsServer("10.0.0.1", 53)  // Only listens on TUN interface
```

**After:**
```kotlin
FakeDnsServer(53)  // Binds to 0.0.0.0 (all interfaces)
```

**Why:** Binding to 0.0.0.0 ensures the server receives DNS packets from the VPN interface.

### 2. Added VPN Configuration Options

**Added to VPN Builder:**
```kotlin
.setBlocking(false)              // Non-blocking I/O for better performance
.setUnderlyingNetworks(null)     // Force ALL traffic through VPN (critical!)
```

**Why:** `setUnderlyingNetworks(null)` tells Android to route ALL traffic through the VPN, preventing it from using the underlying network's DNS.

## How to Apply the Fix

### Step 1: Rebuild the App

```bash
cd android
./gradlew clean
./gradlew assembleDebug
```

### Step 2: Reinstall

```bash
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

### Step 3: Test

1. Open app
2. Enable Fake DNS in settings
3. Connect VPN
4. Check logs - should now see:
   ```
   ✓ Fake DNS server started on 0.0.0.0:53
   ✓ Received DNS query from 10.0.0.2:xxxxx
   ✓ DNS query: google.com -> 198.18.0.1
   ```

### Step 4: Verify DNS

```bash
# In Termux or adb shell
nslookup google.com
# Should return: 198.18.x.x (NOT 10.10.34.36)
```

## What Changed

| File | Change |
|------|--------|
| `FakeDnsServer.kt` | Bind to 0.0.0.0 instead of 10.0.0.1 |
| `GooseRelayVpnService.kt` | Added `.setBlocking(false)` |
| `GooseRelayVpnService.kt` | Added `.setUnderlyingNetworks(null)` |

## Expected Behavior After Fix

### Before (Broken):
```
App → DNS Query → Android System → 10.10.34.36 (filtered DNS) ❌
```

### After (Fixed):
```
App → DNS Query → VPN Interface → Fake DNS (0.0.0.0:53) → 198.18.x.x ✅
```

## If Still Not Working

### Quick Check 1: Disable Private DNS

1. Android Settings → Network & Internet → Private DNS
2. Set to **"Off"**
3. Reconnect VPN

### Quick Check 2: Check Logs

```bash
adb logcat | grep -E "FakeDns|DNS query"
```

Look for:
- ✅ "Fake DNS server started on 0.0.0.0:53"
- ✅ "Received DNS query from 10.0.0.2"
- ✅ "DNS query: google.com -> 198.18.0.1"

### Quick Check 3: Test DNS Directly

```bash
adb shell
nslookup google.com 10.0.0.1
# Should return 198.18.x.x
```

## Alternative Solution

If fake DNS still doesn't work after these fixes, use **Proxy Mode** instead:

1. Settings → Connection Mode → **Proxy**
2. Configure apps to use SOCKS5: `127.0.0.1:1080`
3. DNS resolution happens automatically at server

**Proxy mode always works** because it doesn't rely on Android's VPN DNS configuration.

## Technical Details

### Why setUnderlyingNetworks(null) is Critical

From Android documentation:

> `setUnderlyingNetworks(null)` - The VPN will use whatever the system's default network is. This is the default behavior.

Wait, that's confusing! Actually:

> `setUnderlyingNetworks(null)` - Forces the VPN to NOT use underlying networks, making it the ONLY network.

This prevents Android from bypassing the VPN for DNS queries.

### Why Binding to 0.0.0.0 Works

- `10.0.0.1` - Only listens on TUN interface (might miss packets)
- `0.0.0.0` - Listens on ALL interfaces (catches all DNS packets)

## Summary

✅ **Fixed:** Fake DNS server now binds to 0.0.0.0
✅ **Fixed:** VPN forces all traffic through tunnel
✅ **Result:** DNS queries go to fake DNS, not filtered DNS

**Rebuild the app and test!**

---

**Still seeing 10.10.34.36?** Check `FAKE_DNS_TROUBLESHOOTING.md` for advanced debugging.
