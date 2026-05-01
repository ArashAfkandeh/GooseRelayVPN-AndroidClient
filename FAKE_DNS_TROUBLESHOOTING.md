# Fake DNS Troubleshooting Guide

## Issue: Seeing Filtered DNS IP (10.10.34.36) in Logs

### Problem
When Fake DNS is enabled, you're still seeing the system's default DNS server (like 10.10.34.36) being used instead of the fake DNS server.

### Root Cause
Android is using the underlying network's DNS servers instead of the VPN's DNS configuration. This happens because:

1. Android prioritizes system DNS over VPN DNS in some cases
2. Some apps bypass VPN DNS and use hardcoded DNS servers
3. The VPN interface DNS configuration isn't being applied correctly

### Solution Applied

**Changes Made:**

1. **FakeDnsServer.kt** - Changed to bind to `0.0.0.0` instead of `10.0.0.1`
   - Reason: Binding to 0.0.0.0 ensures the server receives packets from all interfaces

2. **GooseRelayVpnService.kt** - Added two critical settings:
   ```kotlin
   .setBlocking(false)              // Non-blocking I/O
   .setUnderlyingNetworks(null)     // Force all traffic through VPN
   ```

### How to Test the Fix

1. **Rebuild and reinstall the app:**
   ```bash
   cd android
   ./gradlew assembleDebug
   adb install -r app/build/outputs/apk/debug/app-debug.apk
   ```

2. **Enable Fake DNS and connect VPN**

3. **Check logs for:**
   ```
   ✓ Fake DNS server started on 0.0.0.0:53
   ✓ Received DNS query from 10.0.0.2:xxxxx
   ✓ DNS query: google.com -> 198.18.0.1
   ```

4. **Test DNS resolution:**
   ```bash
   # In Termux or adb shell
   nslookup google.com
   # Should return 198.18.x.x (not 10.10.34.36)
   ```

5. **Check with dig (if available):**
   ```bash
   dig @10.0.0.1 google.com
   # Should return 198.18.x.x
   ```

### If Still Not Working

#### Option 1: Add DNS Blocking Route

Add a route to block the filtered DNS server:

```kotlin
// In GooseRelayVpnService.kt, after addRoute("0.0.0.0", 0)
if (globalSettings.fakeDnsEnabled) {
    // Block common filtered DNS servers
    try {
        builder.addRoute("10.10.34.36", 32)  // Block this specific DNS
    } catch (e: Exception) {
        Log.w(TAG, "Could not add DNS block route: ${e.message}")
    }
}
```

#### Option 2: Use Private DNS Setting

1. Go to Android Settings → Network & Internet → Private DNS
2. Set to "Off" (disable Private DNS)
3. This prevents Android from using DNS over TLS which bypasses VPN

#### Option 3: Check Split Tunneling

If split tunneling is enabled, make sure the DNS server app isn't excluded:

1. Settings → Split Tunneling
2. Make sure system apps are included in VPN

#### Option 4: Use tun2socks Instead

If fake DNS still doesn't work, consider using tun2socks (like MasterDnsVPN) which requires Go code changes but is more reliable.

### Debugging Commands

**Check if fake DNS server is listening:**
```bash
adb shell netstat -tuln | grep :53
# Should show: udp 0.0.0.0:53
```

**Check VPN interface DNS:**
```bash
adb shell getprop | grep dns
# Should show: 10.0.0.1
```

**Monitor DNS queries:**
```bash
adb logcat | grep -E "FakeDns|DNS query"
```

**Check routing table:**
```bash
adb shell ip route
# Should show routes through tun0
```

**Test fake DNS directly:**
```bash
adb shell
su  # if rooted
nslookup google.com 10.0.0.1
# Should return 198.18.x.x
```

### Common Issues

#### Issue 1: DNS Server Not Receiving Packets

**Symptoms:**
- Logs show "Fake DNS server started"
- But no "Received DNS query" messages

**Solution:**
- Check if VPN interface is up: `ip addr show tun0`
- Verify DNS is set to 10.0.0.1: `getprop | grep dns`
- Make sure `setUnderlyingNetworks(null)` is set

#### Issue 2: Apps Using Hardcoded DNS

**Symptoms:**
- Some apps work, others don't
- Logs show DNS queries for some domains but not others

**Solution:**
- Some apps (like Chrome) use DNS over HTTPS (DoH)
- Disable DoH in app settings
- Or use split tunneling to force those apps through VPN

#### Issue 3: Android Using Private DNS

**Symptoms:**
- DNS queries go to 8.8.8.8 or 1.1.1.1 instead of fake DNS

**Solution:**
- Disable Private DNS in Android settings
- Or add routes to block those DNS servers

#### Issue 4: Permission Denied on Port 53

**Symptoms:**
- Error: "Permission denied" when starting fake DNS server

**Solution:**
- Port 53 requires root on some devices
- Change fake DNS port to 5353 and update VPN config
- Or use a different approach (tun2socks)

### Alternative: Use Proxy Mode

If fake DNS continues to have issues, the simplest solution is to use **Proxy Mode**:

1. Settings → Connection Mode → Proxy
2. Configure apps to use SOCKS5 proxy: `127.0.0.1:1080`
3. DNS resolution happens automatically at the server

**Advantages:**
- ✅ No DNS configuration needed
- ✅ Always works
- ✅ True remote DNS

**Disadvantages:**
- ❌ Requires per-app configuration
- ❌ Not all apps support SOCKS5

### Summary

The issue of seeing filtered DNS (10.10.34.36) is caused by Android using the underlying network's DNS instead of the VPN DNS. The fixes applied should resolve this:

1. ✅ Bind fake DNS to 0.0.0.0
2. ✅ Set `setBlocking(false)`
3. ✅ Set `setUnderlyingNetworks(null)`

If issues persist, try:
- Disable Private DNS in Android settings
- Add routes to block filtered DNS servers
- Use Proxy mode instead

### Need More Help?

Check logs with:
```bash
adb logcat | grep -E "FakeDns|Interceptor|VPN"
```

And share the output for further debugging.
