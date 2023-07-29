# Swept's* Notes on Hosting SS14
##### *I stole most of these.

## Important Notes
### Launcher
The launcher pulls a list of servers from the "Hub", similar to Byond. Servers are **manually** added to the **official** launcher by a member of SS14's head staff. You can still connect to non-hub servers through their IP with the official launcher by using the "Direct Connect to Server..." button in the bottom right of the launcher's "Home" tab. You can also use the "Add favorite..." button to add a direct-connect server to the hub list.

It's still unsure if SS14's maintainers are going to allow alternate hubs to be added to the official launcher, see [this](https://github.com/space-wizards/SS14.Launcher/pull/87) pull request for further reading. There's talk of an [alternate launcher](https://xkcd.com/927/) being developed to account for the SS14 staff's reportedly tight control over the current hub, but I can't speak on that.

### Port Forwarding / Firewalls
RobustToolbox now contains UPnP to automatically forward ports, but this will not solve absolutely 100% of cases. Your router may not support UPnP. In this case, you need to port forward both TCP and UDP ports for whatever the server’s net.port config setting is set to (usually 1212 ), or it won’t work.

SS14.Watchdog will need to be publically accessible if and only if using it in the Git update provider mode with HybridACZ off. Note that turning HybridACZ off is not a default. The default Watchdog port is 5000 , and it’s HTTP, so forward TCP port 5000.

If you have any other firewall-style mechanisms, review the above information **carefully**.

### Server Config
The server config, server_config.toml, contains various settings you can adjust regardless of which method you use. Note however, that the location of this file is different per-method.

## Hosting Methods
There are several different ways to run a Space Station 14 server. Which method is best depends on how you intend to customize the server, how often you want to update it, and your technical knowledge. These methods have been sorted from easiest to hardest.
<details>
  <summary>Official Builds w/ Modification</summary>

  |
**Pros/Cons**: Almost as simple as using an official build, but you can modify YAML files and textures for custom content.

**How-To**
- Download a server build from https://central.spacestation14.io/builds/wizards/ and extract it somewhere.
- In build.json, there is a URL to a client zip file. Download this “SS14.Client.zip” and rename it “Content.Client.zip”, placing it in the directory containing the server executable (where you find build.json).
- Delete build.json.
- Any time you modify a file, modify the equivalent file in the client zip.

**How It Works**: Hybrid ACZ, by design, is almost identical to production builds. As such, it is possible to turn the latter into the former, which is what is described above in steps 1-3. From that point forward, you’re modifying a built Hybrid ACZ server. As it is, there’s nothing ensuring that server and client files remain in sync, so you have to make sure to do so manually. Obviously ACZ or regular Hybrid ACZ is better when possible as this is not a problem.
</details>
<details>
  <summary>ACZ: Automatic Client Zip</summary>
  
|
**Pros/Cons**: Requires the knowledge of a developer, but if you have that, super simple and insanely flexible. Manual updates.

**How-To**
- Have a developer environment. (Properly cloned repository, correct .NET SDK - no IDE is required, but the build must succeed.)
- Do `dotnet build -c Release`
- Run the server (try --data-dir and --config-file options to keep the data outside of the build directories).
- Connect to your server from the launcher.

*server_config.toml location*: Either in bin/Content.Server/ (if you did not pass --config-file to the server executable) or wherever you want (if you did.).

**How It Works**: ACZ assumes a developer environment (i.e. you run your server via `./bin /Content.Server/Content.Server` or go into that directory and run ./Content.Server ). ACZ also assumes you have made a release build, but not a “full release” build - i.e. the output of dotnet build -c Release. ACZ automatically finds the client assemblies in bin/Content.Client and the Resources directory, and creates and hosts a client zip on the status host. The goal of ACZ is that despite whatever modifications you may have made to your content fork, it just works.
</details>

<details>
  <summary>Hybrid-ACZ</summary>

|
**Pros/Cons**: Like ACZ, requires developer knowledge. Less simple than ACZ, but the resulting server executable works outside a developer environment, which when relevant makes it a lot better than ACZ for that usecase. As flexible as ACZ but only really shines when the computer you’re writing the code on isn’t the computer running the server. Manual updates.

**How-To**
- Have a developer environment. (See regular ACZ.)
- Package a server build for the intended target, i.e:
  - Retrieve the resulting zip, i.e. `release/SS14.Server_linux-x64.zip`
  - Send to target server (various methods).
  - Run on target server as you would an official build.
- `python3 Tools/package_server_build.py --hybrid-acz --platform linux-x64`

*Note*: The target server can be the same server you build the Hybrid-ACZ package on. This is relevant when the Watchdog is involved.

**How It Works**: The ACZ code is still enabled in full server builds, and it looks for a file called Content.Client.zip - if it finds it, rather than attempting to construct an ACZ zip, it simply uses this file. This effectively replaces the build.json at the cost of some RAM usage (potentially removable at an IO cost) and other potential ACZ inefficiencies from using the status host as the client zip server. (It’s not like you can’t build your own complex infrastructure if you really care that much. The point of all this is that it’s simple and Just Works.).
</details>
<details>
  <summary>Official Builds + Watchdog (Auto-Updating)</summary>

|
**Pros/Cons**: All options with Watchdog are the most complicated options, as Watchdog does not get any form of public builds, and isn’t often visited by developers. You need to at least have the correct .NET SDK installed for whatever the Watchdog is using at the time, and this may include installing older versions of dependencies such as the ASP .NET Runtime. Developers only. The good news is that using Watchdog brings automatic restarts and at least semi-automatic updates. With this specific usage of it, however, it is about as flexible as using the official server builds directly.

**How-To**
**You are fully expected to figure out your own personal Watchdog publish-and-run workflow.** 
Some things to consider:
- .NET publishing and builds like to copy over appsettings.yaml which will erase your existing appsettings.yaml .
- Start with something like `dotnet publish SS14.Watchdog -c Release --no-self-contained -o doggy`, then erase files it copied over that you don’t want overwriting your main files, then move from the published output into the main “this is where we are running things” directory.
- Might be an idea to keep backups.
- [SS14.Watchdog/appsettings.Development.yaml](https://github.com/space-wizards/SS14.Watchdog/blob/master/SS14.Watchdog/appsettings.Development.yml) contains examples of a few of the update methods.
- For this server hosting method, you want UpdateProviderManifest .
- Poke the watchdog when you want it to check for updates, example: `curl -X POST -d "" -u syndicate_mothership:Honk "http://localhost:5000/instances/syndicate_mothership/update"`.

*server_config.toml location*: Pretty sure it’s `instances/INSTANCE_NAME/config.toml` but not 100% sure.

**How It Works**: This is pretty much how the official servers work. They run Watchdog(s) that get poked when updates happen, and the Watchdogs read the one true official manifest, and restart servers as appropriate. 
</details>
<details>
  <summary>Watchdog ‘Git’ Update Provider (Integrated CI)</summary>
  
|
**Pros/Cons**: As usual, this is a Watchdog-based option so it’s pain. The advantage of this option is that it lets you run what is essentially your own personal CI, all within Watchdog.

**How-To**
- Have Watchdog in a state where you can deploy it reliably.
- Use the Git update type, set the repository to something a command-line Git client running as the user the Watchdog will run as can access without password prompts.
- This means if the repository is private, you may need to be giving SSH keys to the Watchdog.
- Consider giving the watchdog a dedicated user, giving the watchdog user a bare clone of the repository, and pushing to that repository.
- Before you continue, Something to consider here: “HybridACZ: true” or “HybridACZ: false”? HybridACZ is enabled by default, and hosts the client zip on the game server’s status host. This means that UPnP will work perfectly.
- Disabling HybridACZ causes the client zip to be hosted on the Watchdog. This means that the built-in UPnP forwarding will not work because the Watchdog still has to be forwarded. In addition, if disabling HybridACZ, the Watchdog’s configuration must contain it’s own publically accessible URL, since, again, it’s hosting the client zip - and it needs to be able to give the client zip’s URL to the status host so it can give it to the launcher.

*server_config.toml location*: Same as with official-build Watchdog.
</details>

## Examples

This section is for examples of hosting scripts/setups lifted from other servers.

**Nyano-trasen (Circa 7/15/2022)**
```py
#!/usr/bin/env python3

import json
import os
import hashlib
import argparse
import datetime
import string
from pathlib import Path

parser = argparse.ArgumentParser()
parser.add_argument("-c", "--codebase", dest="codebase", help="codebase")
parser.add_argument("-v", "--version", dest="version", help="version")
arguments = parser.parse_args()

cwd = "/var/www/builds.station14.space"

buildpath = cwd + "/{}/builds/{}/".format(arguments.codebase, arguments.version)
manifestpath = cwd + "/{}/builds/manifest.json".format(arguments.codebase)
manifestfile = open(manifestpath, "w")

SHA = 0
TIME = datetime.datetime.utcnow().strftime("%Y-%m-%dT%H:%M:%S.%f-04:00")


path = Path(buildpath)
manifest = Path(manifestpath)

linuxarm64 = ""
linuxx64 = ""
osxx64 = ""
winx64 = ""
client = ""

for file in path.glob('*.zip'):
    sha256_hash = hashlib.sha256()
    with open(file, "rb") as f:
        # Read and update hash string value in blocks of 4K
        for byte_block in iter(lambda: f.read(4096), b""):
            sha256_hash.update(byte_block)
        print(file.name + " " + sha256_hash.hexdigest())
        if file.name == "SS14.Server_linux-arm64.zip":
            linuxarm64 = sha256_hash.hexdigest().upper()
        elif file.name == "SS14.Server_linux-x64.zip":
            linuxx64 = sha256_hash.hexdigest().upper()
        elif file.name == "SS14.Server_osx-x64.zip":
            osxx64 = sha256_hash.hexdigest().upper()
        elif file.name == "SS14.Server_win-x64.zip":
            winx64 = sha256_hash.hexdigest().upper()
        elif file.name == "SS14.Client.zip":
            client = sha256_hash.hexdigest().upper()

data = """{{
  "builds": {{
    "{}": {{
      "server": {{
        "win-x64": {{
          "sha256": "{}",
          "url": "https://builds.station14.space/{}/builds/{}/SS14.Server_win-x64.zip"
        }},
        "linux-arm64": {{
          "sha256": "{}",
          "url": "https://builds.station14.space/{}/builds/{}/SS14.Server_linux-arm64.zip"
        }},
        "osx-x64": {{
          "sha256": "{}",
          "url": "https://builds.station14.space/{}/builds/{}/SS14.Server_osx-x64.zip"
        }},
        "linux-x64": {{
          "sha256": "{}",
          "url": "https://builds.station14.space/{}/builds/{}/SS14.Server_linux-x64.zip"
        }}
      }},
      "time": "{}",
      "client": {{
        "sha256": "{}",
        "url": "https://builds.station14.space/{}/builds/{}/SS14.Client.zip"
      }}
    }}
  }}
}}""".format(arguments.version, winx64, arguments.codebase, arguments.version, linuxarm64, arguments.codebase,
             arguments.version, osxx64, arguments.codebase, arguments.version, linuxx64, arguments.codebase,
             arguments.version, TIME, client, arguments.codebase, arguments.version)
print(data)
manifestfile.writelines(data)
manifestfile.close()

os.system(""" curl -X POST -d "" -H 'Authorization: Bearer KEY' "http://localhost:27690/control/update" """)
```











