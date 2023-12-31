# Cloudy Sunshine 

[Sunshine](https://github.com/LizardByte/Sunshine) in the Cloud !

> Sunshine is a self-hosted game stream host for Moonlight. Offering low latency, cloud gaming server capabilities

---

- [Requirements](#requirements)
- [Usage](#usage)
  - [Quickstart initial setup](#quickstart-initial-setup)
  - [Advanced initial setup](#advanced-initial-setup)
  - [Everyday usage](#everyday-usage)
  - [Customizing installation](#customizing-installation)
- [How much will I pay? 🫰](#how-much-will-i-pay-)
- [Performance tweakings](#performance-tweakings)
- [Maintenance](#maintenance)
  - [Grow disk after resize](#grow-disk-after-resize)
- [Development](#development)
- [License](#license)

## Requirements

- [Nix](https://nixos.org/download)
- Access to an AWS account with read/write permissions on EC2 & Route53

That's all 😎 Nix will provide every tools and binaries: AWS CLI, NodeJS, Pulumi, etc. 

## Usage

Every commands should be run under Nix shell, start it with:

```sh
nix develop
```

### Quickstart initial setup

Follow the guide 🧙

```sh
task quickstart
```

You'll be able to choose interactively your instance config and follow setup. 

### Advanced initial setup

Cloudy Sunshine deploys a Pulumi stack with a Cloud server and use NixOS to configure Sunshine on deployed Cloud server. Nix shell provides required binaries to run deployment tasks (Node, Pulumi, etc.), you don't have to install anything appart Nix as listed above. 

Create Pulumi stack config file from template. Example below use a stack named `sunshine`

```sh
cp infra/Pulumi.template.yaml cp infra/Pulumi.sunshine.yaml
```

Update `Pulumi.sunshine.yaml` as needed and deploy infra. See comments in template for details and usage. 

Once done, deploy infrastructure with:

```sh
task infra
```

Wait for instance to start (you can use `task wait-ssh`). Once started and ready, run NixOS configuration. **First run might take a while as lots of things will be downloaded and installed.** Subsequent runs (for update/upgrade) will run much faster. 

```sh
task nix-config
```

Once NixOS is configured, reboot

```sh
task reboot
```

Sunshine server should now start with Sunshine service.

Access Sunshine via broswer on port 47990 using either task command, IP address or defined FQDN to setup password. Remote access is disabled by default, you'll need to open an SSH tunnel.

```sh
# Open SSH tunnel
task ssh-tunnel

# Open Sunshine on default browser
task sunshine-browser

# Show stack outputs and use IP address or FQDN directly in your browser
task output
# {
#   ...
#   "fqdn": "sunshine.devops.crafteo.io", # <== use IP or FQDN
#   "ipAddress": "3.79.72.13"
# }
```

Congrats, your Cloudy Sunshine is ready ! 🥳

Run Moonlight and connect to your instance. Usage is standard from now on: you'll need to validate PIN via Moonlight and maybe tweak Sunshine configs. 

```sh
moonlight
```

**Important:** remember to `task stop` your instance to avoid unnecessary costs 💸 Even stopped, you may still be billed for disk usage and EIP usage. 

### Everyday usage

Start instance

```sh
task start
```

Run Moonlight and connect to your instance

```sh
moonlight
```

Once finished, **remember to stop your instance to avoid unnecessary costs 💸**

```sh
task stop
```

### Customizing installation

You can leverage Nix modules to customize Sunshine installation by creating a Nix module at `provision/modules/custom-config.nix`. 

Example: install Steam and change default keyboard layout

```nix
{ lib, pkgs, config, ... }: {
    
    # Enable Steam
    programs.steam.enable = true;

    # French layout
    services.xserver = {
        layout = "fr";
        xkbVariant = "azerty";
    };
    console.keyMap = "fr";
    time.timeZone = "Europe/Paris";
}
```

See `provision/modules/custom-config.example.nix` for more examples.

Apply your custom configs with:

```sh
task nix-config 
```

## How much will I pay? 🫰

Depends on your usage and how long your instance will run. **You are responsible for your usage**, make sure to check services costs and have proper usage (eg. stop instance when not in use) to avoid unnecessary costs !


Billable resources on AWS and related pricing page:

- [EC2 instance](https://aws.amazon.com/ec2/pricing/on-demand/#On-Demand_Pricing) - The Sunshine server with GPU, free when stopped.
- [Volume (disk)](https://aws.amazon.com/ebs/pricing/) - Disk storage for instance, billed even when EC2 instance is stopped.
- [EIP address](https://aws.amazon.com/ec2/pricing/on-demand/#Elastic_IP_Addresses) - free while in use, [will cost 0.005$ / h starting 02/2024](https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/)
- [Route53 record](https://aws.amazon.com/route53/pricing/) - mostly free, but you must bring your own Hosted Zone

Optimizing usage and cost:

- Always stop EC2 instance after usage (`task stop`) to avoid unnecessary costs
- Only provision what you use. Eg. if your games are taking ~60 Go of disk, no need for a 500 Go disk.  
- Avoid using a static IP (~3.72$ / month), but Sunshine IP will change every start/stop cycle (not on reboot)-
- When not using your instance for a long time, maybe it's worth destroying it (`task destroy`) and re-install it later - **though you'll lose any content, make sure to make a backup** or use tools that will sync your saves (eg. Steam Cloud sync)

Example setups and pricing estimation*:

|                    | g5.xlarge instance 100 Go GP3 disk 15h / month | g5.xlarge instance 100 Go GP3 disk 20h / month | g5.2xlarge instance 100 Go GP3 disk 20h / month | g5.2xlarge instance 100 Go GP3 disk 30h / month | g5.2xlarge instance 100 Go GP3 disk 30h / month |
|--------------------|------------------------------------------------|------------------------------------------------|-------------------------------------------------|-------------------------------------------------|-------------------------------------------------|
| EC2 instance       |                                         $18.87 |                                         $25.16 |                                          $30.31 |                                          $45.47 |                                          $45.47 |
| Route53 record     |                                          $0.00 |                                          $0.00 |                                           $0.00 |                                           $0.00 |                                           $0.00 |
| EC2 volume (disk)  |                                          $9.52 |                                          $9.52 |                                           $9.52 |                                           $9.52 |                                           $9.52 |
| EIP address        |                                          $3.53 |                                          $3.50 |                                           $3.50 |                                           $3.45 |                                           $3.45 |
| Est. TOTAL / month |                                         $31.92 |                                         $38.18 |                                          $43.33 |                                          $58.44 |                                          $58.44 |

_*Estimation based on eu-central-1 (Frankfurt) pricing in December 2023. Exact prices vary with time and regions._

## Performance tweakings

- Reduce in-game quality
- Reduce screen size
- Sunshine: set lower quality encoder settings

## Maintenance

### Grow disk after resize

See [AWS doc for details](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/recognize-expanded-volume-linux.html)

SSH into instance and:

```
# Check disk size to resize
df -h
lsblk -hT

# Grow partition
growpart /dev/nvme0n1 2

# Resize filesystem
resize2fs /dev/nvme0n1p2
```

## Development

Get Sunshine service status and logs 
 
```sh
# Show sunshine user journal (e.g. sunshine server logs and other useful debug info)
journalctl _UID=1000 -b

# Get user systemd service status and logs 
su sunshine
journalctl --user -u sunshine
systemctl --user status sunshine
```

Run Sunshine interactively via SSH

```sh
# Access display
export DISPLAY=:0

# Retrieve X authority file
ps -ef | grep Xauthority
cp /run/user/1000/gdm/Xauthority ~/.Xauthority

# Get command used to start sunshine (wrapper script)
cat /etc/systemd/user/sunshine.service 
# ...
# ExecStart=/run/wrappers/bin/sunshine /nix/store/pqyi3jwp47zsis33asq8hn2i01zdygcd-sunshine.conf/config/sunshine.conf
# ...

# Run sunshine with or without wrapper and conf
/run/wrappers/bin/sunshine /nix/store/90vsycwg87c82jldkg0ssl0a19a884lp-sunshine.conf/config/sunshine.conf
```

## License

[GNU General Public License v3.0](LICENSE.txt)