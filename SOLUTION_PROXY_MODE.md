# Solution: Use Proxy Mode Instead of Fake DNS

## Problem

Fake DNS implementation isn't working reliably because:
1. Android bypasses VPN DNS in some cases
2. System DNS takes priority over VPN DNS
3. Apps use hardcoded DNS servers
4. Private DNS interferes with VPN DNS

## Why NekoBox Works

NekoBox uses **sing-box** framework which:
- Handles TUN interface in Go layer
- Intercepts DNS packets in Go code
- Has complete VPN stack implementation
- **Requires extensive Go code changes**

## Your Constraint

> "I don't want change any go core code because this is for another project"

This means you **cannot** implement it like NekoBox without changing Go code.

## Best Solution: Proxy Mode

Since you said "i proxy it with this project and its worked perfect", the solution is to **use Proxy Mode** which already works perfectly.

### How to Configure Proxy Mode

#### 1. In GooseRelayVPN App

1. Open Settings
2. Connection Mode → **Proxy**
3. Save settings
4. Connect VPN

#### 2. Configure System to Use SOCKS5

Create a **system-wide proxy** using your GooseRelayVPN SOCKS5:

**Option A: Use ProxyDroid (Requires Root)**
```
1. Install ProxyDroid
2. Set proxy type: SOCKS5
3. Host: 127.0.0.1
4. Port: 1080
5. Enable proxy
```

**Option B: Use Postern (No Root)**
```
1. Install Postern
2. Add proxy: SOCKS5 127.0.0.1:1080
3. Enable VPN mode in Postern
4. All apps will use proxy
```

**Option C: Use NekoBox as Proxy Frontend**
```
1. GooseRelayVPN in Proxy mode (127.0.0.1:1080)
2. NekoBox configured to use upstream SOCKS5: 127.0.0.1:1080
3. NekoBox handles TUN interface
4. NekoBox forwards to GooseRelayVPN
5. GooseRelayVPN tunnels to your VPS
```

This is actually what you're already doing and it works!

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Your Current Setup                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Apps → NekoBox (TUN) → SOCKS5 (127.0.0.1:1080)       │
│                              ↓                          │
│                    GooseRelayVPN (Proxy Mode)          │
│                              ↓                          │
│                         VPN Tunnel                      │
│                              ↓                          │
│                         VPS Server                      │
│                              ↓                          │
│                    DNS Resolved Here ✅                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**This is the perfect solution!**

### Why This Works

✅ **NekoBox handles TUN interface** (with sing-box)
✅ **NekoBox intercepts DNS** (in Go layer)
✅ **GooseRelayVPN handles tunneling** (your existing code)
✅ **No Go changes needed** (in GooseRelayVPN)
✅ **DNS resolved at VPS** (remote resolution)
✅ **Works perfectly** (as you confirmed)

### Configuration

#### GooseRelayVPN Config

```json
{
  "connection_mode": "PROXY",
  "socks_host": "127.0.0.1",
  "socks_port": 1080
}
```

#### NekoBox Config

```json
{
  "outbounds": [
    {
      "type": "socks",
      "tag": "goose-relay",
      "server": "127.0.0.1",
      "server_port": 1080
    }
  ]
}
```

### Benefits

1. **No Go changes** - GooseRelayVPN stays as-is
2. **DNS works** - NekoBox handles it
3. **System-wide** - All apps work
4. **Proven** - You already tested it
5. **Maintainable** - Two separate projects

### Alternative: Implement tun2socks in Go

If you really want fake DNS in GooseRelayVPN itself, you MUST:

1. Add tun2socks library
2. Implement TUN packet handling
3. Intercept DNS in Go layer
4. **This requires changing Go code**

Example libraries:
- `github.com/sagernet/sing-tun` (used by NekoBox)
- `github.com/xjasonlyu/tun2socks` (standalone)

But this contradicts your requirement of "no Go changes".

## Recommendation

**Keep using NekoBox + GooseRelayVPN** as you are now:

1. NekoBox handles VPN/TUN/DNS (frontend)
2. GooseRelayVPN handles tunneling (backend)
3. Perfect separation of concerns
4. No Go changes needed
5. Already working perfectly

### Final Setup

```bash
# 1. Start GooseRelayVPN in Proxy mode
# Settings → Connection Mode → Proxy → Connect

# 2. Configure NekoBox
# Add SOCKS5 outbound: 127.0.0.1:1080

# 3. Connect NekoBox
# NekoBox will use GooseRelayVPN as upstream

# 4. Done!
# All apps → NekoBox (TUN+DNS) → GooseRelayVPN (Tunnel) → VPS
```

## Conclusion

You have two choices:

### Choice 1: Keep Current Setup (Recommended) ✅
- Use NekoBox + GooseRelayVPN together
- No changes needed
- Already working perfectly
- DNS resolved at VPS

### Choice 2: Implement tun2socks (Not Recommended) ❌
- Requires extensive Go code changes
- Violates your "no Go changes" requirement
- Months of development work
- Reinventing what NekoBox already does

**Recommendation: Keep using NekoBox + GooseRelayVPN!**

It's the perfect solution that meets all your requirements:
- ✅ No Go changes in GooseRelayVPN
- ✅ DNS resolved at VPS
- ✅ System-wide VPN
- ✅ Already working

Don't fix what isn't broken! 🎯
