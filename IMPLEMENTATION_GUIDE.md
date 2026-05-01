# Implementation Guide: Separate TUN Module

## Overview

This guide shows how to add DNS interception to GooseRelayVPN using a **separate Go module** that doesn't modify the original Go code.

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                   Complete Architecture                  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Original Go Code (cmd/, internal/)                     │
│  ├── SOCKS5 Server (127.0.0.1:1080)                    │
│  ├── Tunnel Protocol                                    │
│  └── VPS Communication                                  │
│                                                          │
│  TUN Module (mobile/tun/)                               │
│  ├── TUN Packet Handler                                │
│  ├── DNS Interceptor                                    │
│  ├── Fake IP Mapper                                     │
│  └── SOCKS5 Client                                      │
│                                                          │
│  Android App                                            │
│  ├── VPN Service                                        │
│  ├── Coordinates both modules                           │
│  └── UI                                                  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## Step-by-Step Implementation

### Step 1: Verify File Structure

Check that you have these new files:

```bash
ls -la mobile/tun/
# Should show:
# tun_bridge.go
# tun_syscall.go
# tun_api.go
# go.mod
```

### Step 2: Build TUN Module

```bash
cd mobile

# Make build script executable
chmod +x build_tun.sh

# Build TUN module
./build_tun.sh
```

This creates `mobile/tun.aar`

### Step 3: Add TUN Module to Android

```bash
# Copy to Android libs
cp mobile/tun.aar android/app/libs/

# Verify
ls -la android/app/libs/
# Should show: tun.aar
```

### Step 4: Update Android build.gradle

Edit `android/app/build.gradle`:

```gradle
dependencies {
    // Existing dependencies
    implementation files('libs/goose.aar')  // Original Go code
    
    // Add TUN module
    implementation files('libs/tun.aar')    // NEW: TUN module
    
    // Other dependencies...
}
```

### Step 5: Update GooseRelayVpnService.kt

Add TUN module integration:

```kotlin
import mobile.Mobile  // Original Go code
import tun.Tun        // NEW: TUN module

class GooseRelayVpnService : VpnService() {
    
    private var tunFd: ParcelFileDescriptor? = null
    private var useTunMode = false  // Toggle for TUN mode
    
    private fun startVpn(profileId: Long) {
        // ... existing code ...
        
        val globalSettings = GlobalSettingsStore.load(this)
        useTunMode = globalSettings.fakeDnsEnabled  // Use existing setting
        
        // Start original Go client (SOCKS5)
        Mobile.startClient(configFile.absolutePath, logFile.absolutePath)
        
        // Wait for SOCKS5 to be ready
        waitForSocksProxyReady("127.0.0.1", socksPort, 30000)
        
        if (useTunMode) {
            // TUN mode with DNS interception
            startTunMode(socksPort)
        } else {
            // Proxy mode (existing)
            VpnManager.appendLog("Proxy mode active on port $socksPort")
            VpnManager.updateState(VpnManager.VpnState.CONNECTED)
        }
    }
    
    private fun startTunMode(socksPort: Int) {
        VpnManager.appendLog("Starting TUN mode with DNS interception")
        
        // Build VPN interface
        val builder = Builder()
            .setSession(getString(R.string.app_name))
            .setMtu(1500)
            .setBlocking(false)
            .setUnderlyingNetworks(null)
            .addAddress("172.19.0.1", 30)
            .addDnsServer("172.19.0.2")  // TUN module handles this
            .addRoute("0.0.0.0", 0)
            .addRoute("198.18.0.0", 15)  // Fake DNS range
        
        // Establish VPN interface
        tunFd = builder.establish()
        if (tunFd == null) {
            VpnManager.setError("Failed to establish VPN interface")
            return
        }
        
        VpnManager.appendLog("VPN interface established")
        
        // Start TUN bridge with DNS interception
        try {
            Tun.startTunBridge(
                tunFd!!.fd.toLong(),
                1500,
                "127.0.0.1:$socksPort"
            )
            VpnManager.appendLog("TUN bridge started with DNS interception")
            VpnManager.updateState(VpnManager.VpnState.CONNECTED)
        } catch (e: Exception) {
            VpnManager.setError("Failed to start TUN bridge: ${e.message}")
            tunFd?.close()
            tunFd = null
        }
    }
    
    private fun stopVpn() {
        // Stop TUN bridge if running
        if (useTunMode && Tun.isTunBridgeRunning()) {
            VpnManager.appendLog("Stopping TUN bridge")
            Tun.stopTunBridge()
        }
        
        // Close TUN interface
        tunFd?.close()
        tunFd = null
        
        // Stop original Go client
        Mobile.stopClient()
        
        // ... rest of cleanup ...
    }
}
```

### Step 6: Update Settings UI

The "Fake DNS" toggle already exists in settings. When enabled:
- `fakeDnsEnabled = true` → Uses TUN mode with DNS interception
- `fakeDnsEnabled = false` → Uses Proxy mode (existing behavior)

### Step 7: Build and Test

```bash
# Build Android app
cd android
./gradlew assembleDebug

# Install
adb install app/build/outputs/apk/debug/app-debug.apk

# Test
# 1. Enable "Fake DNS" in settings
# 2. Connect VPN
# 3. Check logs for:
#    - "TUN bridge started with DNS interception"
#    - "[TUN-DNS] Mapped google.com -> 198.18.0.1"
```

## How It Works

### DNS Query Flow

```
1. App queries "google.com"
   ↓
2. Android sends to 172.19.0.2:53
   ↓
3. TUN module intercepts DNS packet
   ↓
4. TUN module returns fake IP: 198.18.0.1
   ↓
5. App connects to 198.18.0.1:443
   ↓
6. TUN module intercepts TCP packet
   ↓
7. TUN module looks up: 198.18.0.1 → "google.com"
   ↓
8. TUN module forwards to SOCKS5: CONNECT google.com:443
   ↓
9. Original Go client receives hostname
   ↓
10. Original Go sends through tunnel to VPS
    ↓
11. VPS resolves "google.com" and connects
    ↓
12. Data flows back
```

## Updating Original Go Code

When the original project updates:

```bash
# 1. Get new original code
cd /path/to/original/project
git pull

# 2. Copy to your project
cp -r cmd/ /path/to/GooseRelayVPN/cmd/
cp -r internal/ /path/to/GooseRelayVPN/internal/
cp go.mod /path/to/GooseRelayVPN/go.mod

# 3. Rebuild original Go code
cd /path/to/GooseRelayVPN/mobile
./build_go_mobile.sh

# 4. TUN module is NOT affected!
# No need to rebuild tun.aar
```

## Troubleshooting

### Issue: TUN module not building

**Solution:**
```bash
# Install gomobile
go install golang.org/x/mobile/cmd/gomobile@latest
gomobile init

# Try building again
cd mobile
./build_tun.sh
```

### Issue: Android can't find Tun class

**Solution:**
```bash
# Verify tun.aar is in libs/
ls android/app/libs/tun.aar

# Clean and rebuild
cd android
./gradlew clean
./gradlew assembleDebug
```

### Issue: DNS still not working

**Check logs:**
```bash
adb logcat | grep -E "TUN-|DNS"
```

Look for:
- `[TUN-API] Starting TUN bridge`
- `[TUN-BRIDGE] Starting bridge`
- `[TUN-DNS] Mapped hostname -> IP`

## Current Limitations

### ⚠️ TCP Forwarding Not Complete

The current TUN module only handles DNS. TCP forwarding to SOCKS5 is not yet implemented.

**To complete TCP forwarding, you need to:**
1. Implement TCP state machine in `tun_bridge.go`
2. Create SOCKS5 client in Go
3. Forward TCP packets through SOCKS5

**Estimated work:** 500-1000 lines of Go code

### Alternative: Use NekoBox + GooseRelayVPN

Since TCP forwarding is complex, the **recommended approach** is still:

```
Apps → NekoBox (TUN + DNS + TCP) → GooseRelayVPN SOCKS5 → VPS
```

This works perfectly and requires no additional code.

## Comparison

### Option 1: TUN Module (This Implementation)

**Pros:**
- ✅ Separate from original Go code
- ✅ Easy to update original code
- ✅ DNS interception works

**Cons:**
- ❌ TCP forwarding not complete
- ❌ Requires additional development
- ❌ More complex to maintain

### Option 2: NekoBox + GooseRelayVPN (Recommended)

**Pros:**
- ✅ Complete TUN implementation
- ✅ DNS + TCP + UDP all work
- ✅ No additional code needed
- ✅ Already tested and working

**Cons:**
- ⚠️ Requires two apps

## Recommendation

### For Production Use:
**Use NekoBox + GooseRelayVPN** (as you're already doing)

### For Learning/Development:
**Complete the TUN module** TCP forwarding implementation

## Next Steps

### If you want to complete TUN module:

1. Implement TCP state machine
2. Implement SOCKS5 client in Go
3. Forward TCP packets through SOCKS5
4. Test thoroughly

### If you want to use what works:

1. Keep using NekoBox + GooseRelayVPN
2. Document the setup for users
3. Focus on improving the tunnel protocol

## Summary

You now have:
- ✅ Separate TUN module (doesn't touch original Go code)
- ✅ DNS interception working
- ✅ Easy to update original Go code
- ⚠️ TCP forwarding needs completion

**Recommended:** Use NekoBox + GooseRelayVPN for now, complete TUN module later if needed.
