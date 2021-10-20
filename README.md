# raspberry-ansible
Setup a new Raspberry Pi 3 B+ with Ansible

I bought a new Raspberry Pi 3 B+ and as I love to automate things, I want to automate the setup as far as possible. This repo contains my notes, what I did - maybe it is of help to you.


### HowTo configure Raspberry headless to expose SSH

I want to use my Raspberry as a home server only - so no need for Desktops whatsoever. And it will only be connected via Ethernet - directly to my Router.

1. Install [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
2. Connect the SD card to your Mac
3. Install Raspberry PI OS LITE (no desktop needed)

4. Remove and reconnect SD card to your Mac

5. Enable SSH on Raspberry by creating `ssh` file on the SD card inside `/boot` directory (because by default Raspbian disables sshd, we have to enable it):

```
touch /Volumes/boot/ssh
```

6. Remove the SD card, plug it into your Raspberry and then connect your Pi to your Network via Ethernet (additional configuration needed for Wifi). 
7. Plug in your power adapter and wait for your Pi to boot up.
8. Try to connect to your Pi via SSH using `pi` user & password `raspberry`. Therefore you may need to find your Raspberry's IP at in your Router - or you try the second option:

![router-raspberry-ip](screenshots/router-raspberry-ip.png)
```
ssh pi@yourIpHere

# or without IP
ssh pi@raspberrypi.local
```

10. Initial configuration

Run `sudo raspi-config` to change the default password (__1 Change User Password__) and expand the file system, to make the entire space of SD card available (__7 Advanced Options / A1 Expand Filesystem__). Then go to __Finish__, which will issue a reboot.

11. Copy your public key to your Pi

Prereq: Have your public key available (e.g. by generating a new keypair with `ssh-keygen`)

```
ssh-copy-id -i ~/.ssh/id_rsa.pub pi@raspberrypi.local
```


### Install Docker with Ansible

Run the Ansible playbook with:

```
ansible-playbook rpi-nextcloud.yml -i hosts
```

# Molecule?

### Use Molecule with Ansible to develop a role for your Pi

Now we have SSH running, we should be able to use Ansible with our Raspi! As we are developing infrastructure code, we shouldn't miss to make use of [Test-driven development (TDD)](https://en.wikipedia.org/wiki/Test-driven_development) - therefore we also use Molecule & Testinfra. If you don't know them, read this post about [Test-driven infrastructure development with Ansible & Molecule](https://blog.codecentric.de/en/2018/12/test-driven-infrastructure-ansible-molecule/) and come back here.

Be sure to have Ansible & Molecule installed. As we need to have a testing infrastructure ready, we should install Docker too. Use the package manager of your choice for Ansible & Docker but always use `pip` for Molecule (and for `docker-py`, which is needed for Molecule + Docker integration)

```
brew install ansible
brew install docker
brew install python
pip3 install molecule
pip3 install docker-py
```

Now let's initiate a new Ansible role skeleton with Molecule:

```
molecule init role --driver-name docker --role-name rpi-nextcloud --verifier-name testinfra
```

As we need to emulate a PI on our Mac/Win/Linux machine to test what we're doing with Ansible, we need to keep in mind that a Raspberry uses ARM instead of x86/amd64 architecture! So we need a Docker image, that is based on ARM. Luckily there is [resin/rpi-raspbian](https://hub.docker.com/r/resin/rpi-raspbian/), which is maintained quite well. We could have also started with [HyperiotOS](https://blog.hypriot.com/getting-started-with-docker-on-your-arm-device/), which is also a great starting point. But then we also need to flash this as the Raspberry image already at the beginning - maybe I'll give it a try later, for now I'll stick to the standard raspbian image.

So let´s change the Docker image inside our [molecule.yml](docker/molecule/default/molecule.yml):

```
scenario:
  # default scenario is Docker
  name: default

driver:
  name: docker
platforms:
  - name: docker-rpi
    image: resin/rpi-raspbian
    privileged: true

provisioner:
  name: ansible
  lint:
    name: ansible-lint
    enabled: false
  playbooks:
    prepare: prepare-docker-in-docker.yml
    converge: ../playbook.yml

lint:
  name: yamllint
  enabled: false
verifier:
  name: testinfra
  directory: ../tests/
  env:
    # get rid of the DeprecationWarning messages of third-party libs,
    # see https://docs.pytest.org/en/latest/warnings.html#deprecationwarning-and-pendingdeprecationwarning
    PYTHONWARNINGS: "ignore:.*U.*mode is deprecated:DeprecationWarning"
  lint:
    name: flake8
  options:
    # show which tests where executed in test output
    v: 1

```

# Links

* https://medium.com/@viveks3th/how-to-bootstrap-a-headless-raspberry-pi-with-a-mac-6eba3be20b26
* https://github.com/yusekiya/raspi3_setup
* https://hackernoon.com/raspberry-pi-headless-install-462ccabd75d0