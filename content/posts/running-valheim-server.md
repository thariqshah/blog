---
author: ["Thariq Shah"]
title: "Running Valheim Server on FriendlyELEC CM3588"
date: 2025-05-04T18:58:31+05:30
draft: false
ShowToc: true
cover:
  image: /valheim-server-friendlyelec/cover/valheim-server-cover.png
categories: ["how-to", "Valheim", "Gaming"]
tags: ["valheim", "friendlyelec", "docker", "arm64", "gaming-server"]
series: ["Game Server Deployments"]
searchHidden: true
---

## 🚀 Running the Valheim Server Using Docker Compose

The Docker Compose configuration we used to deploy a Valheim server on the FriendlyELEC CM3588 board/Arm SBC

```yaml
version: "3"

services:
  raspiheim:
    image: arokan/raspiheim:latest
    container_name: raspiheim
    environment:
      - SERVER_NAME=Raspiheim
      - SERVER_PASS=Raspipass
      - WORLD_NAME=Raspiworld
      - PUBLIC=disabled
      - CROSSPLAY=disabled
      - UPDATE=enabled
      - SAVE_INTERVAL=1800
      - BOX86=default
      - BOX64=default
      - NO_BUILD_COST=disabled
      - PLAYER_EVENTS=disabled
      - PASSIVE_MOBS=disabled
      - NO_MAP=disabled
    ports:
      - "2456:2456/udp"
      - "2457:2457/udp"
    volumes:
      - "/path/to/valheim/data:/data"
      - "/path/to/valheim/server:/valheim"
    restart: unless-stopped
```

> 💡 **Note**: Replace `/path/to/valheim/data` and `/path/to/valheim/server` with actual absolute paths.

---

## 💾 Copying Your Valheim World to the Server

Want to continue exploring your existing world instead of starting from scratch? Copy your Valheim world files to the server's volume.

### 📁 Locate World Files on Your PC

```text
C:\Users\<YourUsername>\AppData\LocalLow\IronGate\Valheim\worlds
```

look for these files (replace MyWorld with your world name)

- MyWorld.db
- MyWorld.fwl

Copy the files to container path `/path/to/valheim/data/worlds_local/`

---

## 🌐 Networking & Tunneling with playit.gg

Since most local servers are behind NAT or firewalls, I used [**playit.gg**](https://playit.gg) to tunnel the Valheim server, making it accessible to friends over the internet — no port forwarding required.

### Steps:

1. **Sign up at [playit.gg](https://playit.gg)** and download the client for Linux ARM64.
2. \*\*Gve it executable permissions:
   ```bash
   chmod +x playit
   ./playit
   ```
3. **Follow the on-screen instructions** to link the tunnel to your account.
4. **Create a UDP tunnel** for port `2456` (default Valheim port).
5. Share the generated tunnel address with your friends (it looks like `abc.playit.gg:2456`).

> ✅ **Tip**: Run playit.gg in the background using `tmux`, `screen`, or a systemd service so it survives disconnections.
