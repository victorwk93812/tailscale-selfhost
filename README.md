# Self-Hosting

## Hosting on Laptop with Tailscale

Tailscale provides a personal private network, consisting of node machines each with its own tailscale network interface, a tailscale IP, and each with identity authentified in advance. 
Nodes could have nearly full access to each other's services, (https, SSH, etc.), via a secured Wireshark P2P session. 
Tailscale provides routing, (magic)DNS, authentication services, while keeping the actual data traffic node-to-node. 
Here's [how tailscale works](https://tailscale.com/blog/how-tailscale-works) for more.  

For our production self-hosting https services today, we rely on tailscale by:  

1. All clients have tailscale installed, authentified themselves, and are added into the private tailnet.  
2. The Linux server with running services is accessible via https by tailscale's built-in `serve` command, which acts as a https reverse proxy routing traffic into the tailscale interface to specified ports.  
3. Clients on the network could access the https services by either `<tailscaleIP>:<tailscale_interface_service_port>` or `<magicDNS_domain_name>:<tailscale_interface_service_port>`.  

Useful information:  

| Title | Context |
|:-:|:-:|
| Podman Compose File Location | `~/selfhost/docker-compose.yml` |
| Tailscale Admin Page | <https://login.tailscale.com/admin/machines> |
| Tailscale -- ArchWiki | <https://wiki.archlinux.org/title/Tailscale> |
| Podman -- ArchWiki | <https://wiki.archlinux.org/title/Podman> |
| `podman-restart` reboot `enable-linger` GitHub thread | <https://github.com/containers/podman-compose/issues/587> |
| Anki -- ArchWiki | <https://wiki.archlinux.org/title/Anki> |
| Sync Server -- Anki Manual | <https://docs.ankiweb.net/sync-server.html> | 
| Anki Sync Server Dockerhub Image | <https://hub.docker.com/r/jeankhawand/anki-sync-server/tags> |
| Silverbullet Dockerhub Image | <https://hub.docker.com/r/zefhemel/silverbullet> |
| Installation with Docker - Forgejo | <https://forgejo.org/docs/latest/admin/installation/docker/> |
| Forgejo Image | <https://codeberg.org/forgejo/forgejo> |

### Selfhosted Servers in a Podman Container

**Step-by-Step Deployment:**

1. **Create a master directory** for your self-hosted stack somewhere in your home folder:

```bash
mkdir ~/selfhost
cd ~/selfhost
```

2. Create the compose file and directories for server data as follows:  

```
.
├── anki-data
├── forgejo-data
├── docker-compose.yml
└── silverbullet-data
```

3. **Start the podman stack:**

```bash
podman compose up -d
podman compose ps # check running processes
```

*(The `-d` flag runs it in the background so you can close your terminal).*

4. **Retain the services across reboots:** 
We have to both enable the systemd user `podman-restart` service for the user running the container, and enable lingering for the user:  

```bash
systemctl --user enable --now podman-restart
loginctl enable-linger $USER
```

Now if 3. yields proper running services, then services marked with `restart: always` should be fire up on all further system startups.  

### Tailscale Setup

#### Server

On the server we route running services on conventional ports to ports on the tailscale interface by the tailscale serve command. 

First we at least must have tailscale installed, the systemd daemon `tailscaled` activated  

```bash
sudo systemctl enable tailscale
```

and then run the tailscale service (which spins up the tailscale interface for incoming connections, etc.) by  

```bash
sudo tailscale up
```

Next we're about to let tailscale handle service routing and accessibility via the tailnet (the tailscale interface).  
For example according to the compose file, we now have the following real network interface port to internal image port of individual services mapping:

1. `localhost:3000` <-> `<forgejo_service_image>:3000`  
1. `localhost:3001` <-> `<silverbullet_service_image>:3000`  
1. `localhost:8080` <-> `<anki_service_image>:8080`  

We now further wanna route the services on 3000, 3001 and 8080 ports to tailscale interface ports so that clients could access them via tailnet, so we do  

```bash
sudo tailscale serve --bg --https=443 localhost:3000
sudo tailscale serve --bg --https=8443 localhost:3001
sudo tailscale serve --bg --https=9443 localhost:8080
```

Now the tailscale built-in reverse proxy is run, and our services should be accessible across the entire private tailnet. 
More useful tailscale commands  

```bash
sudo tailscale serve status # check proxy rules, shows service access point  
sudo tailscals serve reset # flushes all proxy rules  
tailscale ip # check current machine tailscale IP  
tailscale netcheck # debug tool, including information about NAT, latency, etc.
```

You simply run your three `sudo tailscale serve` commands to proxy them out to your Tailnet, and your self-hosting infrastructure is fully operational.

#### Clients

You simply install tailscale and login + authenticate yourself. 
On PC this could be done on browser, and on mobile you have to install the tailscale app.  
If the server is properly setup, now the tailscale IP ping should work, as well as the tailscale magicDNS domain names, and eventually all the services should be accessible.  

### Migrating from an existing anki sync server

This is actually optional, since you could always start a new profile and sync all current anki progress into a new sync server account's database.  

Anyways the following step helped me migrated the database from the AUR `anki-sync-server` to the new selfhost location `~/selfhost/anki-data/` managed by the container.  

1. **Migrate your Anki data:**

Copy your existing AUR database into the new local folder Docker will use.  

```bash
mkdir anki-data
sudo cp -r /var/lib/anki-sync-server/.syncserver/. ~/selfhost/anki-data/
```

2. **Find a way to findout the anki-sync-server's UID and GID from running image:** 
For me, this is  
    1. Create a new account.  
    2. Connect Anki to the sync server and login the new account.  
    3. Examine the database directory `~/selfhost/anki-data/` and checkout the UID and GID.  

3. **Change permissions recursively for the database directory:**  

```bash
sudo chown -R <UID_from_2.>:<GID_from_2.> ~/selfhost/anki-data/
```

4. **Disable the old AUR service** so it doesn't conflict with your new container:  

```bash
sudo systemctl disable --now anki-sync-server
```

5. **Restart the container:**  
Now the sync server should have the old account loginable, with all its data kept.  
