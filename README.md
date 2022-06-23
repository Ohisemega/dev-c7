



The launcher script is specific to machines that have /dls and /dls_sw mounted 
so intended for running on DLS workstations only.

This is for getting DLS RHEL7 dev environment working in a container hosted on  
RHEL8.

This is a stopgap so that we can use effort to promote the kubernetes 
model https://github.com/dls-controls instead of using effort in 
rebuilding our toolchain for RHEL8.

How to use
==========

These instructions will work for a RHEL8 or RHEL7 DLS workstation (or
any linux workstation that has /dls_sw and /scratch mounted)

configure podman
----------------

For first time use only:

    /dls_sw/apps/setup-podman/setup.sh

start the container
-------------------

To start the devcontainer:

    ./run-dev.sh

work as usual
-------------

You will now have a prompt inside of the developer container. You will be
running as your own user ID if using run-dev.sh. If you are using the vscode
integration then you will run as root but file access to the host filesystem
will use your own user id. (this is due to an issue with one of the 
services that vscode starts in the container - to be investigated 
further)

These host filesystem folders are mounted.

    - Your home directory
    - /scratch
    - dls_sw/*  (or at least those dls_sw mounts needed by a controls dev)

You could add further mounts as required by editing a copy of run-dev.sh.

You are free to yum install anything and to modify any of the files inside
the container but these changes will only last for the lifetime of the
container (this restriction can be lifted by using container commits but
you would need to remove the --rm argument in run-dev.sh).

work with edm (optional)
------------------------

If you want to use edm then you will need to install the local fonts on your
host machine. Use ``sudo yum install`` on each of the rpms in this folder
https://github.com/dls-controls/k8s-epics-utils/tree/main/dls-images/edm-fonts

VSCode integration
==================

VSCode has beautiful integration for developing in containers. However, to make
it work with podman requires a little persuasion.

I believe this will only work on RHEL8. The earlier version of podman on RHEL7
does not have the correct API.

install podman-docker
---------------------

Execute these commands:

    sudo yum install podman-docker
    systemctl --user enable --now podman.socket
edit user settings
------------------

Add the following to  /home/[YOUR USER NAME]/.config/Code/User/settings.json

    "docker.dockerodeOptions": {
        "socketPath": "/run/user/[YOUR USER ID]/podman/podman.sock"
    },

(you can find your uid with the `id` command)

Adding these additional settings ensures that each terminal loads the bash (login) profile by default. 
This includes essential setup for DLS developer environment:
```
"terminal.integrated.defaultProfile.linux": "bash (login)",
"terminal.integrated.profiles.linux": {
    "bash (login)": {
        "args": [
            "-l"
        ],
        "path": "bash"
    }
}
```

install remote development plugin
---------------------------------

Run up vscode and install the remote development plugin:

    module load vscode
    code

Inside vscode:
    
    Ctrl+P
    ext install ms-vscode-remote.vscode-remote-extensionpack

add container config to local repository
----------------------------------------

Finally drop the file `.devcontainer.json` into the root folder of a project
and open that folder with VSCode. You will be prompted to reopen the project
in a container.

UNFORTUNATELY: run-dev.sh uses --userns=keep-id to give you your native user id
inside and outside of the container. With VSCode integration this starts the
container OK but fails when VSCode tries to exec a service in the container.
Therefore we drop this option in .devcontainer.json and you will run as root
inside the VSCode terminals. 

Running as root has minimal side affects, any interaction
with host filesystems will use your own ID (because podman runs in your user
ID). SSH keys will still work but to SSH to another machine you will need
to use the following:
```bash
ssh your_fed_id@machine_name
```

Known Issues
============

profile mismatch
----------------
When a devcontainer is launched, it will usually start a single terminal that is
NOT using the login profile. To work around this, close and reopen the 1st terminal
or type:
```
bash -l
```

insufficient UIDs/GIDs
----------------
When building the container for the first time, you may come across the following:
```
Error: writing blob: adding layer with blob 
"sha256:0b2dc63a68b9b80b6e261e0c71119894a739d353f8263d6b2f1394c66a45f5af": ApplyLayer exit status 1 stdout:  
stderr: potentially insufficient UIDs or GIDs available in user namespace (requested 0:54 for /run/lock/lockdev): 
Check /etc/subuid and /etc/subgid: lchown /run/lock/lockdev: invalid argument
```

Rootless Podman uses a pause process to preserve the unprivileged namespaces, which locks down the user files /etc/subuid and /etc/subgid.
The following command will stop the pause process and release the files for editing:

    podman system migrate

