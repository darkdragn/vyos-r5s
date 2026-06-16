## TOC

1. [Set up Armbian](#set-up-armbian)
    1. Image used
    1. Package installs / Updates
    1. (Maybe) udev fix info (Might not be needed)
1. [Compose setup](#compose-setup)
    1. (Doc)Variables
1. [Stop talking just give me the steps!!!](#just-the-steps)


## Set up Armbian

- Download image: Head over [to the armbian R5S page](https://armbian.com/boards/nanopi-r5s) and download your preferred image. I used the debian bookworm image, primarily since the vyos rolling is bookworm based. It shouldn't matter, but for the sake of the kernel, I tried keeping it close.
- Write to an sdcard: 
    - Current filename as of writing this is Armbian_25.5.1_Nanopi-r5s_bookworm_current_6.12.28_minimal.img.xz. This will be our if (input file) line.
    - Use lsblk to determine the block device id for your sdcard. Mine was sda. This is our of (output file) line.
    - Use dd to write it, or if you're on windows or mac research what works for that. `unxz -dc Armbian_25.5.1_Nanopi-r5s_bookworm_current_6.12.28_minimal.img.xz | dd of=/dev/sda` 
    - Plug it in and get ready to rock. The first run will prompt for a username and password for your sudo user. ALSO!!! Big warning, the first run set display params for HDMI properly for my device, but subsequent runs set HDMI at a forced 4K which has text so distorted I can't read it on my 1080p monitor. Be careful, record your mac and keep an eye on what IP it gets.
- (Optional) Run updates: `apt update && apt upgrade -y` after frist boot.
    - __Warning__ Following updates the board reported the NICs differently after reboot. This messed with the udev rules. I'll provide further details below and put in a locked ansible task as a fix. I'll come back to pop a link in here once I get to that part.
- (Optional) Install to eMMC if you have it: use armbian-install to knock that out. I'm still running on the sdcard for now, I might move to emmc in the future.
- Install docker: 
    - [According to the docs](https://docs.armbian.com/User-Guide_Advanced-Configuration/#install-docker) you have two options: CON001 for docker lite and CON002 for full docker.
    - In the iteration of armbian-config that came with the Bookworm image, CON002 isn't valid, only CON001
    - You can just run `armbian-config` and navigate the UI via Software -> Containers -> Docker to install it.
    - Or if you want it to be more automated, `armbian-config --CON001` will install it for you.
    - Heads up, if you haven't rebooted since the `apt upgrade`, docker didn't want to start for me. It tried loading br_netfilter, but the modules for the running kernel wern't there anymore. Don't panic! Once you reboot you'll be running the kernel that has it's modules in /lib/modules and it'll start up fine.
- Update docker daemon
    - I'm going to add an ansible task for this, but if you'd rather perform things manually: fill in /etc/docker/daemon.json with '{"ipv6": true,  "fixed-cidr-v6": "2001:db8::/64"}' You don't need the single quotes, just using it to separate the json from the other text.
    - By default, docker.service is triggered by docker.socket. This means docker.socket is on at startup, but the daemon doesn't actually start until someone runs a docker command. This means even if the service is `restart: always` the container won't start until someone logs in and runs `docker ps` or any other docker cli command. We don't want that. This is going to be a router/firewall... Do router and firewall things without manual intervention. To fix this just run `systemctl enable --now docker.service`

## Compose setup

I was not brave enough to give it full host networking, so I went with the macvlan driver. This has a couple of drawbacks, though. The first is that each macvlan interface has to have an IP assigned, meaning the interface addresses will be managed by docker not by your Vyos config. At least for untagged vlan. This has no impact on sub interfaces.

- Render your compose:
    - You can take compose.yml.example and rename it to compose.yml, then update all of the spots called out by comments.
    - Alternatively, you can fill in the vars at the top of the vyos.yml playbook and run `ansible-playbook -i localhost, vyos.yml --tags compose` and it'll render it for you.


## Just the steps

- Download Debian Bookworm: https://armbian.com/boards/nanopi-r5s
- Write to sd card, imageName=Armbian_25.5.1_Nanopi-r5s_bookworm_current_6.12.28_minimal.img.xz sdcardBlock=/dev/sda; adjust for your system: 
    - `unxz -dc Armbian_25.5.1_Nanopi-r5s_bookworm_current_6.12.28_minimal.img.xz | dd of=/dev/sda status=progress`
- Install docker: `armbian-config --CON001`
- Install ansible and git: `apt update && apt install -y ansible git`
- Clone my repo: `git clone https://github.com/darkdragn/vyos-r5s`
- Pick: Ansible or manual config
    - Let ansible do everything else for you:
        - Populate your network stats in vyos.yml
        - Run: `ansible-playbook vyos.yml`
        - This will: Update docker service, update docker daemon.json, render your compose
    - Do things manually:
        - Update daemon.json: `echo '{ "ipv6": true,  "fixed-cidr-v6": "2001:db8::/64" }' > /etc/docker/daemon.json`
        - Update docker daemon: `sudo systemctl enable docker && sudo systemctl restart docker`
        - Make a copy of the example compose: `cp compose.yml.example compose.yml`
        - Update commented sections of compose.yml; use your own editor, I prefer vim.
- Start it up: `docker compose up -d`
- It'll take a second to generate some stuff, like an initial boot config. You can monitor with `docker compose exec -it vyos journalctl -ef`
- When it's done, run as vyos user: `docker compose exec -it su - vyos`
- You can start the ssh daemon and use it for all future access / configs. Using the standard method:
```
config
set service ssh
```

- It'll be accessible IAW with whatever you configured in the compose.yml. In the compose the var is services.vyos.networks.lan1.ipv4_address or services.vyos.lan2.ipv4_address. It depends on how you have things wired up. **Warning** Unless you want to mess with rp_filter, I don't recommend having both lan1 and lan2 on the same subnet. In my env, they're two completely different network segments, but if you have them on the same segment with DHCP on, the host armbian os might get an IP on each and have rp_filter problems. Just tweak netplan to disable DHCP on one of them.
- **Caution** After an update of armbian, my system reported a different PCI path/ID than the udev rules from the base image accounted for. This means wan and lan1 got their aliases, but lan2 was enP1p17s0. If this happens to you, you can just adjust the parent name in compose.yml or you can update /etc/udev/rules.d/70-persistent-net.rules. I also have a manual task with the tags [manual, udev]. You can use it with `ansible-playbook vyos.yml --tags udev`. No guarantee you get the same PCI id I did, though.
    
