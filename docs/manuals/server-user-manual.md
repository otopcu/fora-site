---
description: How to run, connect to, configure and operate the Fora RTI server (Fora.Rti.Server) — quick start, settings, security, Docker and troubleshooting.
---
# Fora.Rti.Server User Manual

`Fora.Rti.Server` is the Fora RTI: the server your HLA 4 federates connect to. It implements IEEE 1516-2025 (HLA 4) and
speaks the standard **Federate Protocol**, so federates built with `Fora.Client` — or with any other HLA 4 Federate
Protocol client — can use it. It runs without a user interface; to watch it live, use `fora-console`.

## Table of Contents
- [1. Quick Start](#1-quick-start)
- [2. Which Tool Do I Need?](#2-which-tool-do-i-need)
- [3. Connecting Federates](#3-connecting-federates)
- [4. Configuration](#4-configuration)
- [5. Security](#5-security)
- [6. Saving and Restoring](#6-saving-and-restoring)
- [7. Monitoring and Administration](#7-monitoring-and-administration)
- [8. Running in Docker](#8-running-in-docker)
- [9. Troubleshooting](#9-troubleshooting)
- [10. Compatibility](#10-compatibility)

---

## 1. Quick Start

**Step 1 — Start the server.** Pick one:

<pre><code class="language-powershell"># From the NuGet package (Fora.Rti), in the package's tools folder
dotnet Fora.Rti.Server.dll

# From the source repository
dotnet run --project src\Fora.Rti.Server\Fora.Rti.Server.csproj

# In Docker (build the image once, from the source repository root)
docker build -f src\Fora.Rti.Server\Dockerfile -t fora-rti:local .
docker run --rm -p 8080:8080 -p 15164:15164 fora-rti:local
</code></pre>

**Step 2 — Check that it runs.** The log shows the Federate Protocol listener:

<pre><code class="language-text">04:16:51 [INFO] [FpServer] FP listener running on 127.0.0.1:15164 (TLS: False).
</code></pre>

and the health endpoint answers (port `7112` locally, `8080` in Docker):

<pre><code class="language-powershell">curl http://localhost:7112/health
</code></pre>

**Step 3 — Connect a federate** to `localhost:15164`. With `Fora.Client`:

<pre><code class="language-csharp">await using var rti = new ForaClient();
await rti.ConnectAsync("localhost:15164", myFederateAmbassador);
await rti.CreateFederationExecutionAsync("MyFederation", "MyFom.xml");
await rti.JoinFederationExecutionAsync("Federate1", "MyFederateType", "MyFederation");
</code></pre>

That's it. The [Fora.Client Programmer's Manual](fora-client-programmers-manual.md) covers the client side in
full.

## 2. Which Tool Do I Need?

| I want to… | Use | Notes |
|---|---|---|
| Host the RTI (service, container, CI) | `Fora.Rti.Server` | This manual. Headless; logs to the console |
| Watch the RTI live while developing | `fora-console` (`Fora.Rti.Console`) | Starts the same server in-process, or attaches to a running one with `--connect`. See the [Console User Manual](console-user-manual.md) |
| Script health checks, snapshots, log tails | `fora-admin` (`Fora.Rti.Admin`) | See the [Admin Tool User Manual](admin-tool-user-manual.md) |
| Build my own admin tool | `Fora.Rti.Admin.Client` library | See its [Programmer's Manual](admin-client-programmers-manual.md) |

`fora-console` runs exactly the same server code as `Fora.Rti.Server`, with the same settings, so everything in this
manual applies to it as well.

## 3. Connecting Federates

### Addresses

| Transport | Address | Enabled by |
|---|---|---|
| TCP | `localhost:15164` | Always on (`FederateProtocolPort`) |
| TLS | `localhost:15165` | `Tls:Enabled=true` and a certificate — see [Security](#5-security) |
| WebSocket | `ws://localhost:7112` | `WebSocketEnabled=true` (upgrade at `/` on the HTTP port) |
| Secure WebSocket | `wss://<host>:<https-port>` | `WebSocketEnabled=true` and an HTTPS endpoint in `Kestrel` |

By default the server accepts connections from the local machine only. To accept federates from other machines, set
`FederateProtocolBindAddress` to `*`.

### FOM modules

- **Send the FOM with the federate** (recommended). `Fora.Client` and other Federate Protocol clients send the module
  content, so the file only has to exist on the federate's machine.
- A bare file name that the client does not send is looked up in the server's own folder (its content root).
- `http(s)://` module URLs are refused unless you allow them with `AllowRemoteFomUrls` (see
  [Security](#5-security)).
- Modules must be **IEEE 1516.2-2025** object models. Modules in the older 1516.2-2010 (HLA Evolved) format are
  refused with `InvalidFOM`.
- The standard MIM (`HLAstandardMIM`) is loaded automatically.

### Names

Class names are written with their superclasses, dot-separated; the root class may be left out, so `Vehicle.Car` and
`HLAobjectRoot.Vehicle.Car` are the same class. Names are case sensitive. A name the FOM does not define raises
`NameNotFound`.

### Federations

The server hosts any number of federations at the same time. Each has its own FOM modules, handles, time management
and saves; nothing crosses from one federation to another.

## 4. Configuration

Settings live in `appsettings.json` next to the server executable. Each one can also be given as an environment
variable or a command-line argument, which is handy in scripts and containers:

<pre><code class="language-powershell"># appsettings.json:   "ForaRtiServer": { "FederateProtocolPort": 15200 }
$env:ForaRtiServer__FederateProtocolPort = "15200"                         # environment variable
dotnet Fora.Rti.Server.dll --ForaRtiServer:FederateProtocolPort=15200     # command line
</code></pre>

### Server settings (`ForaRtiServer`)

| Setting | Default | What it does |
|---|---|---|
| `FederateProtocolPort` | `15164` | Federate Protocol TCP port. `0` turns the TCP listener off (for a WebSocket-only host). |
| `FederateProtocolBindAddress` | `localhost` | Where the Federate Protocol listeners accept connections. `*` accepts from any network interface. |
| `WebSocketEnabled` | `false` | Accept Federate Protocol connections over WebSocket at `/` on the HTTP/HTTPS port. |
| `Tls:Enabled`, `Tls:Port`, `Tls:CertificatePath`, `Tls:CertificatePassword` | off, `15165` | TLS listener — see [Security](#5-security). |
| `AllowRemoteFomUrls` | `false` | Allow FOM modules given as `http(s)://` URLs to be downloaded by the server. |
| `MaxFomModuleBytes` | 16 MiB | Largest FOM or MIM module accepted. |
| `MaxPayloadSize` | 10 MiB | Largest Federate Protocol message accepted. |
| `SavePath` | `saves` | Folder for saved federation states, relative to the server folder unless absolute. |
| `SnapshotStoreType` | `file` | `file` keeps saves on disk; `memory` keeps them only while the server runs. |
| `Authorization:*` | off | HLA authorization — see [Security](#5-security). |
| `Admin:ApiKey` | none | Protects the `/admin/*` endpoints — see [Security](#5-security). |
| `LazyNameResolution` | `false` | For tests without a FOM only: invent handles for unknown names instead of raising `NameNotFound`. Leave it off. |

### Session timeouts (`FpSessionTimeouts`)

| Setting | Default | What it does |
|---|---|---|
| `HeartTimeoutSeconds` | `180` | A connection that sends nothing for this long (not even a heartbeat) is closed. Federate Protocol clients send heartbeats after 60 s of silence. |
| `PurgeTimeoutSeconds` | `600` | How long a disconnected federate may reconnect and resume its session. After that it is resigned automatically. |
| `PollIntervalSeconds` | `5` | How often the timeouts are checked. |

### HTTP port (`Kestrel`)

The health and admin endpoints (and WebSocket connections) use `http://localhost:7112` by default, set in
`Kestrel:Endpoints:Http:Url`. The Docker image uses `http://+:8080`.

## 5. Security

All security features are off by default, which suits a developer machine. Turn on what your network needs.

**Encrypted federate connections (TLS)**

<pre><code class="language-json">"ForaRtiServer": {
  "Tls": { "Enabled": true, "Port": 15165, "CertificatePath": "certs/server.pfx", "CertificatePassword": "…" }
}
</code></pre>

The certificate is a PKCS#12 (`.pfx`) file with its private key. Federates then connect to port `15165` with TLS.
For secure WebSocket, configure an HTTPS endpoint and certificate in `Kestrel` instead.

**Who may create, join and destroy federations (HLA authorization)**

<pre><code class="language-json">"ForaRtiServer": { "Authorization": { "Enabled": true, "PlainTextPassword": "…" } }
</code></pre>

Federates then connect with `HLAplainTextPassword` credentials; wrong or missing credentials get `Unauthorized`.

**Admin API key**

Set `ForaRtiServer:Admin:ApiKey` to require an `Authorization: Bearer <key>` header on every `/admin/*` request.
`/`, `/health` and `/version` stay open. The tools send the key with `--token <key>`. Use HTTPS when you set a key, so it is not sent in clear text.

**Remote FOM URLs** stay refused unless `AllowRemoteFomUrls` is on, so a federate cannot make the server fetch
arbitrary URLs.

## 6. Saving and Restoring

- **Federation save/restore** is part of HLA: a federate requests a save with a label, every federate saves its own
  state, and a later restore brings the federation back to that point. Restoring one federation never affects
  another.
- **Admin snapshots** capture the whole RTI at once, without the federates' help: `POST /admin/snapshot/<label>` to
  save, `POST /admin/snapshot/<label>/restore` to restore (also available as `save`/`restore` in `fora-console` and
  `fora-admin`).

Saves go to `SavePath` on disk, or stay in memory with `SnapshotStoreType=memory` (useful in containers without
storage).

## 7. Monitoring and Administration

### Endpoints

| Endpoint | Method | What you get |
|---|---|---|
| `/` | GET | A short status line |
| `/version` | GET | Version, standard and capability information |
| `/health` | GET | Health for probes and orchestrators |
| `/admin/federations` | GET | Active federations |
| `/admin/federates` | GET | Joined federates with name, type and federation |
| `/admin/sessions` | GET | Federate Protocol sessions, online or waiting to resume |
| `/admin/logs` | GET | Recent log entries; `?since=<ISO 8601 time>` filters |
| `/admin/logs/stream` | GET | Live log stream (Server-Sent Events) |
| `/admin/snapshot/{label}` | POST | Save a snapshot |
| `/admin/snapshot/{label}/restore` | POST | Restore a snapshot (`404` if the label is unknown) |
| `/admin/shutdown` | POST | Shut the server down gracefully |

### Tools

<pre><code class="language-powershell">fora-console --connect http://localhost:7112      # live dashboard for a running server
</code></pre>

<pre><code class="language-bash">fora-admin --connect http://localhost:7112 health
fora-admin --connect http://localhost:7112 federations list
fora-admin --connect http://localhost:7112 --format json sessions list
fora-admin --connect http://localhost:7112 logs tail
</code></pre>

### Reading the log

| Level | Meaning |
|---|---|
| `SUCCESS` | A lifecycle step completed: federation created or destroyed, federate joined or resigned |
| `INFO` | Normal activity: connections, sessions going offline and resuming, FOM modules loaded |
| `WARN` | A federate's request was refused with an HLA exception (the federate receives it too), or a session timed out |
| `ERROR` | Something went wrong inside the RTI — worth reporting |

## 8. Running in Docker

The image is built from the source repository (it is not published to a registry yet):

<pre><code class="language-powershell"># Build the image, from the repository root
docker build -f src\Fora.Rti.Server\Dockerfile -t fora-rti:local .

# Keep saves in a named volume
docker run --rm -p 8080:8080 -p 15164:15164 -v fora-rti-saves:/var/lib/fora-rti/saves fora-rti:local

# No storage needed: keep saves in memory
docker run --rm -p 8080:8080 -p 15164:15164 -e ForaRtiServer__SnapshotStoreType=memory fora-rti:local
</code></pre>

Inside the container the server listens on all interfaces: HTTP on `8080`, Federate Protocol on `15164`.

## 9. Troubleshooting

| What you see | Likely cause | What to do |
|---|---|---|
| The federate cannot connect | Wrong port, or the server only accepts local connections | Check the `FP listener running on …` log line; set `FederateProtocolBindAddress=*` for remote federates and open the port in the firewall |
| `CouldNotOpenFOM` | The FOM file was not found | Let the client send the module content (the default for Fora.Client), or place the file in the server folder |
| `InvalidFOM` | The module is not well-formed XML, is too large, or is not an IEEE 1516.2-2025 object model (1516.2-2010 modules are not supported) | Validate the file against the 1516.2-2025 schema; check `MaxFomModuleBytes` |
| `InconsistentFOM` | The same module name was given twice, or the modules do not combine into a valid FDD (including modules added at join) | Give each module once; make the modules agree with each other and with the federation's FOM |
| `NameNotFound` | The class, attribute, parameter or dimension name is not in the FOM | Check spelling and case, and include every superclass (`Vehicle.Car`); the `HLAobjectRoot.` prefix is optional |
| `FederationExecutionAlreadyExists` on create | Another federate created it first | Normal — catch it and join |
| `InvalidLogicalTime` when sending | A time-stamped message sent in time-stamp order is earlier than the federate's time plus its lookahead | Use a later timestamp, or check the lookahead |
| A federate disconnects after 3 minutes of silence | No heartbeats reached the server within `HeartTimeoutSeconds` | Make sure the client sends heartbeats (Fora.Client does) or raise the timeout |
| A federate disappears some minutes after losing its connection | It did not resume within `PurgeTimeoutSeconds` and was resigned automatically | Reconnect sooner, or raise the timeout |
| `Unauthorized` on connect, create or join | HLA authorization is on and the credentials are missing or wrong | Connect with the configured `HLAplainTextPassword` |
| `401` from `/admin/*` | `Admin:ApiKey` is set | Send `Authorization: Bearer <key>`; with the tools use `--token <key>` (`fora-admin` also reads `FORA_RTI_TOKEN`) |
| A module URL is refused | Remote URLs are off | Send the content instead, or set `AllowRemoteFomUrls=true` on a trusted network |

## 10. Compatibility

- **Standard:** IEEE 1516-2025 (HLA 4) — the 1516.1 services, 1516.2-2025 FOM modules, and the Federate Protocol with
  protocol version 1. HLA Evolved (IEEE 1516-2010) federates are not supported.
- **Clients:** any HLA 4 Federate Protocol client. Besides `Fora.Client`, Pitch's open-source FedProClient has been
  verified in Java and C++ over TCP, TLS, WebSocket and WSS, across every HLA service group.

---

## Disclaimer

See the [Disclaimer](../disclaimer.md) for usage terms and research-only conditions.
