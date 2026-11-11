# Reliable Minecraft crossplay egg

One Paper world shared by Java, Xbox and iOS/Android players, at home and over the internet.
Java and mobile players save **mc.nrgserver.me**; Xbox players join the bot's **Friends** session.

## How players reach the server

- **Java:** the DNS-only wildcard record, kept current by ddclient, resolves `mc.nrgserver.me` to the
  home IP. The router forwards TCP 25565 to Paper on the Calagopus/Wings VM at `192.168.0.11`.
- **iOS/Android Bedrock:** save `mc.nrgserver.me` with the default Bedrock port, 19132. The router
  forwards TCP and UDP 19132 to Geyser in the same game container. TCP handles NetherNet signaling;
  UDP carries gameplay. Geyser and Floodgate connect the player to the same Paper world.
- **Xbox:** MCXboxBroadcast advertises a joinable session through the bot's Xbox account. When the
  player joins from **Play → Friends**, the bot relays the request to Geyser inside the container.
  Geyser supplies the gameplay connection information; the bot does not relay gameplay. The same
  Friends session can be used from home or another network when that network permits the connection.

**Bedrock Advertise Address = auto** enables Geyser's STUN discovery instead of pinning the VM's LAN
address. Discovery supplies public UDP mappings alongside the candidates gathered by ICE and is
refreshed by the networking library. ddclient and discovery have different jobs: ddclient keeps the
initial hostname reachable, while discovery keeps the gameplay endpoint current. An ISP address
change can interrupt play until DNS and discovery catch up; it is not a seamless connection handover.

Using the public hostname from home may require the router's NAT loopback for both TCP and UDP.
Java working at home proves only the TCP path. Test all clients inside and outside before relying
on the setup; Docker/NAT mappings and a particular Xbox or mobile network can still affect Bedrock.

Consoles cannot add custom servers. MCXboxBroadcast keeps that restriction out of the connection
steps; phones can add the server directly. Java already defaults to 25565 and Bedrock to 19132, so
no SRV record is needed. Bedrock does not support SRV records.

## What every start does

1. Checks the settings and stops with a message that names any setting it cannot use.
2. With **Auto Update** on, refreshes Paper (stable builds of exactly **Minecraft Version**), Geyser,
   Floodgate, ViaVersion and MCXboxBroadcast from their official download APIs. Each jar must match
   the publisher's SHA-256 and pass a zip test before it atomically replaces the current one, and the
   replaced jar is kept in `.crossplay/previous/`. Any failure keeps the current jar. The only fatal
   case is a first start that cannot download Paper.
3. Refuses to start if either **Minecraft Version** or the Paper jar on disk is older than the newest
   version this world has run on, which it records in `.crossplay/minecraft-version`.
4. Starts Paper with a 1 GiB initial heap (capped at the maximum for smaller memory limits) and a
   maximum heap of 80% of the memory limit. Pre-touching is disabled so an idle server does not
   allocate its maximum heap upfront. Geyser uses built-in NetherNet signaling over TCP and gameplay
   over UDP, both on **Bedrock Port**. Auto mode enables STUN discovery; an explicit IP disables
   discovery and supplies only that address as the advertised endpoint.

Calagopus also rewrites these settings before every start, so edits to them do not stick:

| File | Setting | Why |
| --- | --- | --- |
| `server.properties` | `online-mode=true`, `white-list=true`, `enforce-whitelist=true` | The Java port is on the internet. |
| `server.properties` | `enforce-secure-profile=false` | Bedrock players cannot sign chat messages. |
| MCXboxBroadcast `config.yml` | `friend-sync.expiry.enabled: false` | The default unfriends anyone who has not joined for 15 days. |
| Geyser `config.yml` | `bedrock.signaling.advertise-addresses: []` | A saved explicit list would disable STUN even when auto mode enables it. Explicit IP mode uses a JVM override. |

Leave MCXboxBroadcast's `session.remote-address` and `session.remote-port` on `auto`: the extension
then relays joins to Geyser inside the container, while any other address sends them out through
Docker and back in.

## Before merging

This runbook uses the Calagopus/Wings VM at **192.168.0.11** and the Java hostname
**mc.nrgserver.me**. This README is copied to the public egg mirror when merged.

Already configured:

- The VM has a DHCP reservation at **192.168.0.11**.
- The wildcard DNS record for **\*.nrgserver.me** points to the public home IP. It covers
  **mc.nrgserver.me** without a separate A record.

1. Confirm the wildcard is **DNS only** and ddclient keeps it current. If it is proxied for other
   services, leave it as-is and create a separate DNS-only `mc` A record instead; only that case
   requires adding `mc.nrgserver.me` to ddclient as well.
2. Create a dedicated Microsoft account for MCXboxBroadcast, then sign in at xbox.com once to
   create its Xbox profile and gamertag. The bot does not need to own Minecraft. An alternate account
   is recommended because MCXboxBroadcast's client emulation may risk an account ban.
3. Record the Java player's Minecraft Java username and the Xbox player's gamertag. These are for
   the whitelist; neither player's password or email is needed.

## Merge and publish

1. Merge the Minecraft crossplay PR into `main`.
2. In GitHub Actions, wait for **Publish Calagopus Eggs** on that merge to succeed.
3. Confirm `minecraft-crossplay/egg-minecraft-crossplay.json` exists in
   [the public mirror](https://github.com/nrgbistro/calagopus-eggs/tree/main/minecraft-crossplay).

Merging publishes the egg; the following steps create and configure the server in Calagopus.

## After merging

### 1. Import the egg

Merging the generated `eggs/minecraft-crossplay/dist/` files to `main` publishes them to `https://github.com/nrgbistro/calagopus-eggs.git`. In
**Admin → Egg Repositories**, add that URL if it is not there yet, click **Sync**, open the repository's
**Eggs** tab, and install **Minecraft Crossplay (Reliable)** into a nest. A later change to this egg
reaches the panel the same way: sync again, then use **Update from Repository** on the installed egg.
You can also import `egg-minecraft-crossplay.json` by hand through **Admin → Nests**.

### 2. Create the server

- **Image:** Java 25 (the only one offered; Paper 26.3 needs it).
- **Memory:** keep the 6144 MiB limit for this server's maximum of three players. It starts with a
  1024 MiB heap and can grow to 4915 MiB as needed, with additional memory used outside the heap.
  The JVM may retain expanded heap memory after players leave. The start refuses limits below
  1024 MiB.
- **Allocations:** the primary allocation is the Java port (25565). Add a second allocation for
  Bedrock (19132). Calagopus publishes every allocation on both TCP and UDP, which NetherNet needs.
- **Minecraft Version:** `26.3`.
- **Bedrock Port:** `19132` (the second allocation's port).
- **Bedrock Advertise Address:** `auto`. A literal IPv4/IPv6 address is an explicit fallback and
  disables discovery. Setting `192.168.0.11` permits only clients that can reach that LAN IP (home
  or a routed VPN); it does not enable public Bedrock play. Hostnames and address lists are not
  accepted by this egg's fallback setting.
- **Auto Update:** on (`1`).
- **Xbox Friends Broadcast:** on (`1`).

### 3. First start

1. Accept the Minecraft EULA when the console asks.
2. MCXboxBroadcast prints a code to enter at `https://www.microsoft.com/link`. Sign in with the bot's
   Microsoft account: the project recommends an alternate account because its client emulation may
   break Xbox terms. The sign-in is cached in `plugins/Geyser-Spigot/extensions/mcxboxbroadcast/`.
3. Allow the two players from the console:

   ```text
   whitelist add <JavaName>
   fwhitelist add <XboxGamertag>
   ```

   Replace the placeholders with the actual names; omit a leading `/` in the panel console. Quote
   an Xbox gamertag containing spaces, for example `fwhitelist add "Xbox Gamer Tag"`. Floodgate shows
   Bedrock players with a `.` prefix in game; use the Xbox gamertag in `fwhitelist`.

### 4. Router and DNS for the Java player

1. Add this router port-forward rule:

   | Setting | Value |
   | --- | --- |
   | Protocol | TCP |
   | External port | 25565 |
   | Destination LAN IP | 192.168.0.11 |
   | Internal port | 25565 |

   Add a second rule for **both TCP and UDP**, external port **19132** to
   **192.168.0.11**, internal port **19132**. The operator has already configured this forwarding;
   verify both protocols are included. If a VM firewall is enabled, allow TCP 25565 and TCP/UDP 19132.
2. Use the existing wildcard DNS record. With a DNS-only wildcard already maintained by ddclient,
   no new DNS record, ddclient edit or configuration PR is needed. Confirm the hostname resolves to
   your current public home IPv4 address:

   ```sh
   dig +short mc.nrgserver.me A
   ```

   If it resolves to Cloudflare proxy addresses, the wildcard is proxied: use the separate DNS-only
   `mc` record described above. If it resolves to an old home IP, check the existing ddclient updates.
3. Have the Java player test from outside your home network: in Minecraft Java, add
   **mc.nrgserver.me** as the server address and join. Port 25565 is the default, so no port suffix
   is needed. Confirm they can enter the world.

### 5. Bedrock players

For iOS/Android, sign into Minecraft with the player's Microsoft account. Add their Xbox gamertag
with `fwhitelist add <XboxGamertag>` in the panel console, then use **Play → Servers → Add Server**:

| Field | Value |
| --- | --- |
| Server name | NRG Server |
| Server address | mc.nrgserver.me |
| Port | 19132 (leave the default) |

Save the entry once and use it on home Wi-Fi or away. Accept a trust prompt if shown.

For Xbox:

1. On the Xbox, add the bot account's gamertag as a friend. The bot accepts within about a minute.
2. In Minecraft, open **Play → Friends**. The server is listed under the joinable friends.
3. Join. If the console asks whether to trust the server, accept it.
4. Confirm the Xbox, phone and Java players enter the same world. Repeat the phone test on mobile
   data with Wi-Fi off and the Xbox test on an outside network when available.

Keep `plugins/Geyser-Spigot/nethernet/identity.pem`: it is the server's NetherNet identity, and a new
one makes a console that trusted the old one ask again. The Xbox account's privacy and online-safety
settings must allow joining multiplayer and cross-network games, which child accounts often block.

### 6. Schedules

Add these in **Schedules** so the server keeps itself current and recoverable. Turn on **Ignore
Failure** for every step: by default a failed step ends the schedule, which could leave world saving
switched off.

- **Daily backup** at a quiet hour, such as 04:00 in the panel's timezone, keeping several days:
  1. Send command `save-off`.
  2. Send command `save-all flush`.
  3. Sleep 15 seconds. A console-line wait can miss a save that finishes before the wait starts,
     and its default timeout is 5 seconds.
  4. Create a backup with **Run in Foreground** on. Otherwise the next step runs while the archive
     is still being written.
  5. Send command `save-on`.
- **Daily restart** after the backup has time to finish, such as 05:00 in the panel's timezone:
  send command `say Restarting in 1 minute`, sleep
  60 seconds, send the restart power action. Each start installs the newest verified Geyser and
  MCXboxBroadcast, so the Xbox can join again within a day of both supporting a new Bedrock
  version, without anyone touching the server. Because the first step ignores failure, this also
  starts a server that was stopped, so pause the schedule during planned maintenance.

Run the backup schedule manually once and confirm the backup completes and `save-on` runs afterward.
Then run the restart schedule and confirm the server returns and both players can reconnect. Adjust
its time if the backup would overlap it. Leave crash detection on so Calagopus restarts the server
after a crash.

## Updating an existing LAN-only server

1. After merging, wait for **Publish Calagopus Eggs** to succeed, then sync the public egg repository.
2. In **Admin → Nests**, open the nest, its **Eggs** tab, then **Minecraft Crossplay (Reliable)**.
   Scroll to **Update → from Repository** and select this egg. On newer panels the installed egg
   may also have an **Update** button under **Admin → Egg Library → Installed**.
3. Stop the game server and **Reinstall**, leaving **delete all files unchecked**. This replaces the
   launcher without deleting the world, bot sign-in or NetherNet identity.
4. In the server's startup variables, change **Bedrock Advertise Address** from its saved
   `192.168.0.11` to **auto**. Updating the egg does not replace an existing server's saved value.
5. Confirm TCP and UDP 19132 are forwarded, then start. The console should show
   `STUN discovery set from system property: true` and should not show
   `Advertised addresses set from system property: 192.168.0.11`.
6. If upgrading a Geyser configuration with `config-version` 10 or older and custom address lists,
   restart once more after Geyser migrates it. That migration can copy legacy
   `bedrock.signaling.builtin.advertise-addresses` or `bedrock.signaling.nxs.advertise-addresses`
   back into the new list; the next start clears it again. Current configurations do not need this.
7. Test the following matrix. The live router, iOS and Xbox paths require real clients; a successful
   server boot alone does not prove they work.

| Client | Home | Outside |
| --- | --- | --- |
| Java | mc.nrgserver.me, default 25565 | Same saved server |
| iOS/Android | mc.nrgserver.me, default 19132 | Same saved server, Wi-Fi off for mobile-data test |
| Xbox | Bot's Friends session | Same bot's Friends session |

If home Bedrock fails but outside works, investigate UDP NAT loopback. A local DNS override for only
`mc.nrgserver.me` to `192.168.0.11` can avoid public routing for the initial connection, but does not
itself fix an unreachable gameplay candidate. Do not change the public wildcard or other services.
If discovery produces unusable endpoints in this Docker/router combination, collect diagnostics
before changing mode. The explicit LAN IP fallback restores the previous LAN-only behavior; it does
not satisfy simultaneous public access.

Verify recovery after a public-IP change when one next occurs: ddclient must update DNS, STUN must
refresh the public mappings, and outside clients must be able to reconnect. A daily restart is not
the intended IP-update mechanism in auto mode.

## Operating it

### After a Bedrock update

Consoles update Minecraft automatically, and each MCXboxBroadcast build only lets in consoles on the
Bedrock version it was built for. After an update, the Xbox player cannot join until Geyser and
MCXboxBroadcast have both published builds for the new version, which the daily restart installs.
The Java player is not affected.

### Upgrading Minecraft

1. Take a backup.
2. Change **Minecraft Version** after Paper marks the new version stable, then restart. Until then,
   the start keeps the current version and logs a warning.

Lowering the version is refused, because a world cannot be downgraded. Restore a backup from the
older version instead. ViaVersion lets an updated Java client join before the server is upgraded.
Never delete `.crossplay/`: the version recorded there is what the downgrade check compares against.

### Rolling back a jar

Turn **Auto Update** off, copy the jar from `.crossplay/previous/` over its managed path, and start.
Turn Auto Update back on once upstream has fixed the problem.

Paper can only be rolled back to a build of the same Minecraft version. Right after an upgrade, the
saved `server.jar` is still the old version's Paper, and the start refuses it rather than open the
upgraded world with it. To go back across versions, restore a backup taken before the upgrade.

| Component | Managed path |
| --- | --- |
| Paper | `server.jar` |
| Geyser | `plugins/Geyser-Spigot.jar` |
| Floodgate | `plugins/floodgate-spigot.jar` |
| ViaVersion | `plugins/ViaVersion.jar` |
| MCXboxBroadcast | `plugins/Geyser-Spigot/extensions/MCXboxBroadcastExtension.jar` |

### Updating the launcher

The installer writes `start-minecraft.sh`, so a changed egg reaches an existing server through
**Update from Repository** on the egg, then **Reinstall** on the server. Leave the option to delete all
files unchecked: the installer only rewrites the launcher and creates missing folders, and the world
and configs stay untouched.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| The server is missing from the Xbox Friends tab | The bot is not a friend, needs a new sign-in, or **Xbox Friends Broadcast** is off | Look for MCXboxBroadcast errors or a new sign-in code in the console, and confirm the friendship in the Xbox app. |
| Bedrock lists the server but cannot join | Missing UDP forward, unusable discovered mapping, or an explicit LAN-only address | Use **Bedrock Advertise Address = auto** for public play; confirm the STUN log and both TCP/UDP 19132 forwarding. MCXboxBroadcast logs `Relayed bedrock client` for a relayed Xbox join, or `Failed to relay offer` with the reason. |
| Bedrock works away but not at home | UDP NAT loopback or local routing | Test the LAN path and the router's loopback support. A local DNS override changes only initial lookup; verify gameplay candidates too. |
| Auto mode still advertises only the old LAN IP | Existing server variable or old launcher | Set the saved startup variable to auto and reinstall the launcher after updating the egg. |
| The Xbox reports an outdated server | The console updated before Geyser or MCXboxBroadcast supported its version | Restart to install the newest builds. If either project has no build for the version yet, wait for one. |
| Every join asks to trust the server again | `identity.pem` was deleted | Restore it from a backup, or accept the prompt once more. |
| Only the Java player cannot connect | Stale DNS record or missing port forward | `mc.nrgserver.me` must resolve to the current home IP, and TCP 25565 must be forwarded to `192.168.0.11:25565`. |
| The start stops with `[crossplay] ERROR` | A setting is invalid, or the first start had no internet access | The message names the setting to fix. |
| ViaVersion warns that it has no compatible versions | The server already runs the newest Minecraft release, so there is nothing newer to translate | Nothing to fix. It starts translating when a newer Java release ships. |

## Developing this egg

In the private nrgserver repository, edit these committed sources:

| File | Purpose |
| --- | --- |
| `eggs/minecraft-crossplay/src/launcher.sh` | Downloads, version checks and Java startup |
| `eggs/minecraft-crossplay/src/install.sh` | Installer template that writes the launcher |
| `eggs/minecraft-crossplay/src/egg-minecraft-crossplay.json` | Egg metadata, variables and panel-managed config |
| `eggs/minecraft-crossplay/README.md` | This runbook |

From the repository root, regenerate and verify:

```sh
python3 scripts/build-eggs.py
python3 scripts/build-eggs.py --check
python3 -m unittest discover -s scripts/tests -p 'test_*egg*.py'
```

Commit the sources and regenerated `dist/` files together. CI rejects a distribution that does not
match its sources. Only `dist/` is published, under the existing `minecraft-crossplay/` path; the
sources and build tools stay in the private repository. Do not edit generated files or the public
mirror directly. Apply launcher changes with the sync, update and reinstall procedure above.

## Upstream references

- [Preparing your Geyser server for NetherNet](https://geysermc.org/blog/preparing-for-nethernet/)
- [Geyser system properties](https://geysermc.org/wiki/geyser/geyser-command-line-arguments-and-system-properties/)
- [Geyser and secure chat](https://geysermc.org/wiki/geyser/secure-chat/)
- [MCXboxBroadcast](https://github.com/MCXboxBroadcast/Broadcaster)
- [Geyser address and STUN configuration](https://github.com/GeyserMC/Geyser/blob/master/core/src/main/java/org/geysermc/geyser/configuration/GeyserConfig.java)
- [NetherNet discovery and refresh implementation](https://github.com/CloudburstMC/Network/blob/nethernet/transport-nethernet/src/main/java/org/cloudburstmc/netty/channel/nethernet/signaling/NetherNetHTTPServerSignaling.java)
- [Paper downloads service](https://docs.papermc.io/misc/downloads-service/)
- [Calagopus egg repositories](https://calagopus.com/docs/panel/next-steps/egg-repos)
