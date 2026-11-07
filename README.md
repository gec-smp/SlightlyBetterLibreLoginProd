# LibreLoginProd

A production-focused fork of [LibreLogin](https://github.com/kyngs/LibreLogin) — an authentication
plugin for Minecraft proxies. It keeps the original config, messages and command layout, but tracks
the newest Minecraft, Velocity and Paper releases.

<p>
  <img src="https://img.shields.io/badge/Java-21%2B-blue?style=for-the-badge&logo=openjdk&logoColor=white" alt="Java 21+">
  <a href="https://github.com/Navio1430/LibreLoginProd/blob/master/LICENSE">
    <img src="https://img.shields.io/badge/License-MPL%202.0-green?style=for-the-badge" alt="MPL 2.0">
  </a>
  <a href="https://github.com/Navio1430/LibreLoginProd/wiki">
    <img src="https://img.shields.io/badge/Docs-Wiki-orange?style=for-the-badge&logo=wikipedia" alt="Wiki">
  </a>
  <a href="https://discord.gg/eSTjEgtXBy">
    <img src="https://img.shields.io/badge/Discord-Support-purple?style=for-the-badge&logo=discord&logoColor=white" alt="Discord">
  </a>
</p>

## What this is

LibreLogin handles everything an online-mode network normally gets from Mojang, on an offline-mode
proxy: premium autologin, `/register` + `/login` for cracked players, sessions, 2FA, and Bedrock
support through Floodgate. This fork exists because the upstream plugin had drifted behind the
Minecraft versions used in production.

**Differences from upstream / original LibreLogin:**

- Support for current Paper and Velocity releases (protocol up to 26.x clients).
- BungeeCord support removed — Velocity and Paper only.
- NanoLimbo integration is deprecated (it is still detected optionally); any limbo server such as
  NanoLimbo or PicoLimbo works as a regular backend.
- Forced-host routing keeps working for premium autologin and Floodgate players, and respects
  `velocity.toml` forced hosts when LibreLogin has no entry for the host (see below).
- Same `plugins/librelogin` folder, `config.conf` and `messages.conf` as upstream — dropping this
  jar in place of the original is enough.

## Features

- Premium autologin (Mojang-checked, no password needed for premium players).
- Offline `/register` and `/login` with bcrypt hashing.
- Session system (IP + timeout based re-login).
- TOTP 2FA (Google Authenticator, Authy, …).
- Premium/offline identity handling: name validation, case-sensitivity checks, profile conflict
  strategies, automatic premium data migration.
- Bedrock support via Floodgate (UUID- and name-based detection).
- Per-hostname lobbies (forced hosts) and configurable limbo servers.
- SQLite (default), MySQL and PostgreSQL storage, plus migration from other auth plugins.
- Optional integrations: LuckPerms, Protocolize, RedisBungee.

## Platforms and requirements

| Platform | Status |
|---|---|
| Velocity 3.x / 4.x (incl. Velocity-CTD) | Supported |
| Paper 1.19+ | Supported |
| BungeeCord | Not supported |

- Runtime: Java 21 or newer (Velocity 4 based proxies need Java 25).
- The proxy must run with `online-mode = false`; LibreLogin performs the authentication.
- Libraries are downloaded on first start into `plugins/librelogin/lib`, so the proxy needs internet
  access once.

## Installation

1. Drop `LibreLoginProd-<version>.jar` into the proxy's `plugins/` folder (or the Paper server's
   `plugins/` for a single-server setup).
2. Start once to generate `plugins/librelogin/config.conf` and `messages.conf`.
3. Register your limbo and lobby servers in the proxy config, then list them in LibreLogin:

   ```hocon
   limbo = [ limbo ]

   lobby {
       root = [ lobby ]
       "play.example.com" = [ survival ]   # use § instead of dots: "play§example§com"
   }
   ```

4. Enable `auto-register` if premium players should be created on their first join.
5. Set `player-info-forwarding-mode = "modern"` in `velocity.toml` (or configure the equivalent
   Paper forwarding) and make the backend use the same forwarding secret.

The wiki has the long version: [Configuring Servers](https://github.com/Navio1430/LibreLoginProd/wiki/Configuring-Servers).

## Server routing and forced hosts

LibreLogin picks the first server that applies, in this order:

1. A forced-host entry in LibreLogin's `lobby` section matching the player's hostname.
2. The server Velocity already selected from `[forced-hosts]` in `velocity.toml` (only if that
   hostname is configured there and LibreLogin has no entry for it).
3. The `root` lobby.

Players that are not authenticated are always sent to a limbo server, regardless of forced hosts.

Forced-host matching is case-insensitive, and the key has to be the hostname the client actually
sends in the handshake:

- **Java**: the address typed in the server list. `play.example.com` is not the same key as
  `example.com`, and an SRV record means the client sends the SRV target.
- **Bedrock (Geyser)**: by default Geyser sends the address it connects to with, which is the same
  for every Bedrock player. To route Bedrock players by hostname, enable `forward-hostname` in
  Geyser's `config.yml`:

  ```yaml
  java:
    forward-hostname: true
  ```

  With that on, Velocity sees the hostname the Bedrock client used, so the same `lobby` keys apply
  to both editions.

Useful when debugging routing:

```hocon
debug = true
```

LibreLogin then logs the resolved host for every connection:

```
[DEBUG] Virtual host for player Notch: play.example.com
```

## Configuration

`plugins/librelogin/config.conf` is self-documenting; the most important options:

| Option | Default | Meaning |
|---|---|---|
| `limbo` | `[limbo0, limbo1]` | Servers for unauthenticated players. |
| `lobby` | `root = [lobby0, lobby1]` | Authenticated destinations, per forced host. |
| `auto-register` | `false` | Create premium users automatically on first join. |
| `session-timeout` | `0` | Seconds a session stays valid; `0` disables sessions. |
| `database.type` | `librelogin-sqlite` | `librelogin-sqlite`, `librelogin-mysql` or `librelogin-postgresql`. |
| `ping-servers` | `false` | Ping backends to filter full servers (irrelevant on Paper). |
| `remember-last-server` | `false` | Return players to the server they were last on. |
| `debug` | `false` | Verbose logging, including virtual hosts. |

## Building from source

```bash
./gradlew shadowJar
```

The jar is written to `target/LibreLoginProd-<version>.jar`. Java 21 is required to build.

> **Note:** the build applies the `xyz.kyngs.libby.plugin` Gradle plugin, which used to be published
> on `repo.kyngs.xyz`. If that artifact is unavailable, dependency resolution fails before
> compilation; a stub that only creates the `libby` configuration (plus a `libby.json` in the jar)
> is enough to produce a working build.

Run `./gradlew spotlessApply` before sending a pull request — the project is formatted with
google-java-format (AOSP style).

## Troubleshooting

- **Premium/Floodgate players land in the root lobby.** Check the `Virtual host for player …` debug
  line: the hostname must match a `lobby` key exactly (mind dots/`§`, SRV targets, and for Bedrock
  `forward-hostname` above).
- **`kick-no-lobby` / `kick-no-limbo`.** No matching server is online, or the server names in
  `config.conf` do not match the proxy config.
- **Players can move before logging in.** Make sure no other plugin bypasses the limbo block, and
  that `limbo` contains a real server (a limbo server, not a lobby).

## Credits and license

- [kyngs](https://github.com/kyngs) and the original LibreLogin contributors — the plugin this fork
  is based on.
- [vuxeim](https://github.com/vuxeim) — keeping the fork working with newer Minecraft versions.
- [Navio1430](https://github.com/Navio1430) — fork maintenance.

Licensed under the [Mozilla Public License 2.0](LICENSE).
