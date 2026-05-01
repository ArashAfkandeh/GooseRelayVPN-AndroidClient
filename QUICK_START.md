# Quick Start - Go TUN Bridge Integration

## What Changed?

The app now uses a **Go-based TUN bridge** for DNS interception instead of the Kotlin fake DNS implementation. This allows DNS resolution to happen on your VPS (remote) instead of locally, bypassing DNS filtering.

## Files Modified

1. **`android/build_go_mobile.sh`** - Builds both `gooserelayvpn.aar` and `tun.aar`
2. **`.github/workflows/android-ci.yml`** - Uploads both AARs as artifacts
3. **`android/app/src/main/java/com/gooserelay/gooserelayvpn/service/GooseRelayVpnService.kt`** - Integrated Go TUN bridge

## How to Build

### On GitHub Actions (Recommended)
```bash
git add .
git commit -m "Integrate Go TUN bridge for remote DNS resolution"
git push
```

GitHub Actions will automatically:
- Build `gooserelayvpn.aar` (original Go client)
- Build `tun.aar` (TUN bridge with DNS interception)
- Build Android APK
- Upload all artifacts

### Locally (Optional)
```bash
# Build both Go modules
cd android
bash build_go_mobile.sh

# Build Android APK
cd android
./gradlew assembleDebug
```

## How to Use

1. **Install the APK** on your Android device
2. **Open the app** and go to Settings
3. **Enable "Fake DNS"** toggle
4. **Connect VPN**
5. **Check logs** - You should see:
   - `Starting Go TUN bridge with DNS interception...`
   - `Go TUN bridge started (DNS will be resolved remotely)`

## Architecture

```
Android App
    ├── gooserelayvpn.aar (Original Go client - NEVER MODIFIED)
    └── tun.aar (Separate TUN bridge module)
```

**When Fake DNS is OFF:**
- Uses standard tun2socks (from original Go code)
- DNS resolved locally (may be filtered in Iran)

**When Fake DNS is ON:**
- Uses Go TUN bridge (from separate module)
- DNS queries intercepted and return fake IPs (198.18.x.x)
- TCP connections to fake IPs are intercepted
- Real hostname is sent to SOCKS5 proxy
- VPS resolves DNS remotely (bypasses filtering)

## Benefits

✅ **No modification to original Go code** - Easy to update  
✅ **Remote DNS resolution** - Bypasses local DNS filtering  
✅ **Single app solution** - No need for multiple apps  
✅ **Builds on GitHub Actions** - Fully automated  
✅ **TCP forwarding included** - Complete solution  

## Troubleshooting

### Build fails on GitHub Actions
- Check that `mobile/tun/` directory exists
- Verify `mobile/tun/go.mod` is present
- Check GitHub Actions logs for specific errors

### App crashes when enabling Fake DNS
- Check logcat: `adb logcat | grep -E "TUN-|GooseRelayVPN"`
- Verify both AARs are in `android/app/libs/`
- Make sure `tun.aar` was built successfully

### DNS still filtered
- Verify "Fake DNS" is enabled in settings
- Check logs for `Go TUN bridge started`
- Test with: `nslookup google.com` (should work through VPN)

## Log Messages to Look For

**Success:**
```
[TUN-API] Starting TUN bridge: fd=X mtu=1500 socks=127.0.0.1:1080
[TUN-BRIDGE] Starting bridge
[TUN-DNS] Mapped example.com -> 198.18.0.1
[TCP] New connection: 198.18.0.1:443
[TCP] Resolved 198.18.0.1 -> example.com
[TCP] SOCKS5 connecting to example.com:443
```

**Failure:**
```
Failed to start Go TUN bridge: ClassNotFoundException
```
→ `tun.aar` not included in build

## Updating Original Go Code

When you want to update the original Go client:

```bash
# 1. Update original Go code (cmd/, internal/, etc.)
cd /path/to/original/go/project
git pull

# 2. Copy to your project (don't touch mobile/tun/)
cp -r cmd/ internal/ go.mod /path/to/GooseRelayVPN/

# 3. Rebuild
cd /path/to/GooseRelayVPN/android
bash build_go_mobile.sh
```

The TUN module (`mobile/tun/`) is **completely separate** and won't be affected by updates to the original Go code.

## Next Steps

1. Push to GitHub and let CI build
2. Download APK from Actions artifacts
3. Test on your device with Fake DNS enabled
4. Monitor logs to verify remote DNS resolution
5. Enjoy bypassing DNS filtering! 🎉
