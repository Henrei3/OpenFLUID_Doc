# Migration VM

[Documentation](./Doc/EN/README.md)

|        |         |
|--------|---------|
| Projet | migration VM |
| Version | 20240717 |
| dépôt GitHub | https://team.openfluid-project.org/gogs/OpenFLUID/migration-VM |
| doc GitHub | https://team.openfluid-project.org/gogs/OpenFLUID/migration-VM/Doc/ |
| doc ReadTheDocs | - |

## Quick Start

For simplicity this Quick Start and the rest of the documentation will be done in English. If you wish to learn more in detail about the structure of this application you can click the link above.

This application uses Ansible as it's main component. It's goal is to deploy the OpenFLUID architecture in an infrastructure-as-code manner. This means that if you read the deployment code, you will be able to understand it's architecture. I have documented the architecture and how it can be deployed. And explained it in detail, so that you don't have to do it.

### Install Ansible

Red Hat recommends to install it's software using pip. So you should have installed Python and pip first:

`apt-get install python3 python3-pip`

Then you can install Ansible with pip:

`python3 -m pip install --user ansible>=10.0`

Ansible version should be 10.0 or superior, since this script uses ansible-core 2.17 functionnality's.

### Install and configure Docker

As the official Docker Documentation says[¹]: « [...] Distro maintainers provide unofficial distribution of Docker packages in APT. You must uninstall these packages before you can install the official version of Docker Engine. » To uninstall them use the command :

`for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done`

If you wish to clean any existing data you should « [...] read the uninstall Docker Engine[²] section »

It is possible to install Docker in a easier way installing Docker Desktop, if you wish to do so you can do it, it poses no problem to the execution of the ansible script. I will be showing you how to install from Docker's apt repository.

[¹]

**Add Docker's official GPG key**

``` 
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

**Add the repository to Apt sources**

```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update 

```

To install docker; run this command in your terminal :

``` sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin ```

Running this Ansible script as root can lead to unexpected errors. However Docker will need root permissions to execute. See [Manage Docker as a non-root user](https://docs.docker.com/engine/install/linux-postinstall/) to solve this problem

### Install KeePass2

If you don't have KeePass2 installed, you will have to install it. We use KeePass2 to store the encryption passwords that decrypt our sensitive files. All passwords are stored inside an encrypted [DataBase](/Database.kdbx). The key used to decrypt the Database should be stored in a second location. Such as an USB stick. If you want you could recreate a DataBase with a Master Password **and** a Keyfile. Even though this is enough measure to secure the sensitive files, it would be usefull to try and use a dedicated device for authentification, for instance a **YubiKey**. And the decryption key could be stored inside a KMS Service like AWS Key Management Service.

If you are on MacOS please install the KeePassXC app:

`brew install keepassxc`

### Set Up Ansible Vault

To execute the playbooks located inside this git repo, this includes the `dev` `test` and `prod` playbooks you should first fill the necessary variables that the scripts needs to function properly, see [Vars file specifications](/EN/README.md#vars-file-specifications). There is already a vars file called main.yml located inside each of the roles. To edit it and fill in your credentials you should ask the administrator for the *decryption key* containing the necessary information to unencrypt the file. If you are unsure about the vault-id of the file. You can just read the last readable word located at the end of the first line of the encrypted file.

> :warning: **The ansible-vault function (keepass-client) has been tested in MacOS and Ubuntu environnements**. Be very carefull when executing in other environnements. Such as Debian, since it handles program interruption differently.

### Deployment

#### Dev

We recommended to first deploy the services in a contained environnement, such as docker. This is done to ensure the compatibility with the actual debian version (bookworm in 2024). To deploy the docker containers, you should execute the script [init.sh](/docker/init.sh) file inside the `docker` folder

**gogs**

You should always deploy the gogs service first since it hosts multiple git repository needed for the deployment. In the case that the original gogs service is down, a backup should have been [done](#add-the-backup-as-a-cron-job) and the git repo url can be specified through a [variable](/EN/README.md#vars-file-specifications).

Since the prj VM hosts only one service, it isn't necessary to deploy a selection of services. You can however choose to only build the device, in which case you could set `deploy` value to `build` using the `-e` flag.

To deploy and configure the prj Virtual Machine. You should execute the next command if you want to deploy the services in a docker stack:

`ansible-playbook -i docker.yml --vault-id dev@scripts/keepass-client prj-services.yml`

**www**

To deploy only a selected service or both at once or even just build the VM and not install any service at all in the www service deployment. To execute all services you should just run the next command `ansible-playbook -i your-inventory.yml --vault-id yourID@script-client www-services.yml` all the services should then deploy. If you want to display only the vitrine service you should add the extra variable deploy equals to **vitrine**. This is done adding the flag `--extra-vars` or `-e` to the command shown above. To only deploy the community service, set the var `deploy` value to **community**. If you only want to build the VM, asign `deploy` value to **build**.

**all services**

To deploy all service in one command althought it is not recommended since it would be harder to trace an error however if you still wish to do it:

`ansible-playbook -i docker.yml --vault-id dev@scripts/keepass-client prj-services.yml www-service.yml hub-service.yml`

#### Prod

The production deployment is meant to be done by an experienced user that has already an understanding of the Ansible script functions. Since the deployment will be now done in distant machines with a static IP Address, you should first connect and copy their ssh id. This is really important since Ansible will need you to have their ID's, to connect himself through ssh, you can do that with the command: `ssh-copy-id host`.

"Host" is the IP Address to the device you want to connect.
This command will prompt you to write your ssh private key password (if you don't have one please create one [Generate a new SSH key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) ).And write the ssh password of the distant machine.

If for any reason you have the Man In the Middle Warning you can remove the host from the keyring : `ssh-keygen -R host`. You should then have to re-add it again.

Please be aware that maybe you are in fact being a victim of a MIM attack, although it is not very likely that you will receive a warning if you really are.

**gogs**

To deploy it in production using as managed node a VM, use the `prod` repertory and execute the next command.

`ansible-playbook -i inventory.yml --vault-id prod@scripts/keepass-client prj-services.yml`

As you can see the only aspects that change between the prod command and the dev one, are the inventory and vault-id values. This is logical since we use a different encryption password for the prod environnement, and a different inventory, since it's a static one, please note that the IP adresses of the VM are encrypted and located inside the `host_names.yml` file.

**all services**

`ansible-playbook -i inventory.yml --vault-id prod@scripts/keepass-client  prj-services.yml www-services.yml hub-services.yml`

None of the underlying services of OpenFLUID will be accessible if the www service is not deployed, since it hosts the reverse-proxy that handles connexions and redirections.

#### Mac Compatibility

If you are using macOs for the dev deployment, you should change the docker_host value to `unix:///var/run/docker.sock`. Additionnaly, ensure that the community.docker module is installed to his latest version. Use: `ansible-galaxy collection install community.docker --force`.

In mac, Docker container are not directly accessible through their IP Address, you will have to launch a browser inside a docker container and connect it to the docker container respective networks adding the `--network` flag. Run the next command:

```
docker run -d --name=firefox -p 5800:5800 -v /Users/jimenezalfonso/datas:/config/:rw --shm-size 20m jlesage/firefox
```

To execute the docker community docker inventory plugin you should have installed the `requests` and `paramiko` library's [³]. Install them with pip

If you have an error that states that ansible-galaxy can't read your certificates (Mac). You should install Python Certificates with this command:

``` 
cd /Applications/Python\X/
./Install\ Certificates.command
```

### Automatic changes in localhost

In order to deploy the entirety of the OpenFLUID structure it is needed to install some packages and run some tasks on localhost this is done automaticaly, we are going to list in here all the packages and changes that will be done/installed into your machine in order to deploy the OpenFLUID services.

- expect

Expect is an executable command that let's you create scripts that receive and send input into a certain command, this is usefull in order to enable an ssh connexion before sending big chunks of data to our nodes.
Expects takes 303 Ko of space. (Original-Maintainer: Sergei Golovan <sgolovan@debian.org>)

- yq/jq

The community.docker.docker_containers invetory plugin doesn't create a docker connexion via it's IP, it does so by connecting to the docker socket. However, to create a network of devices, we need our docker container to be accessible through multiple IP Adresses. To identify this IP adresses we use a script that reads the docker configuration parsing and creating the necessary docker networks. In order to automate network creation and ip assignement we use the (yq) and jq tools, that parse respectively yaml and json files.

yq takes 136 Ko of space (Author: Andrey Kislyuk) (yq has been deprecated by some undiscovered jq functionalities)

jq takes 102 Kb of space (Original-Maintainer: ChangZhuo Chen (陳昌倬) <czchen@debian.org>)

- KPScript Plugin

The [keepass-linux.sh](/ansible/test/keepass-linux.sh) file is a vital part of the ansible-vault functionnality. This is the Linux version of the `keepass-client` script that takes a vault-id as parameter (dev as default) and will get the password associated to that ID, only if the USB device is found. In order to retrieve this password we need to install the KeePass KPScript Plugin.[⁴] Additionnaly to this, we will add `/usr/lib/keepass2/` to the Path.

KPScript takes 60K of space (Author: Dominik Reichl)

### Backup with Ansible

**How to Backup**

Inside the `ansible/prod/` folder you will have all the necessary files to create a backup of the OpenFLUID environment in your machine. This script will copy dynamically all the files specified in the target_vars variable (see [main.yml](/ansible/backup/roles/unencrypted_backup/tasks/main.yml) file). To execute the backup you should run the next command  :

` ansible-playbook -i inventory.yml --vault-id prod@scripts/keepass-client backup_local.yml `

The backup of a selection of files is done in the prod environnement, since it's the environment that contains all the necessary secrets that allow you to access the devices containing the data. In dev it is only possible to `send_files` since making a backup would't be possible with the proper credentials.

If you wish to send_files in a dev environment, this is done following the deployment linearity of the services. If you want to send the vitrine, dev and wareshub data you will have to execute the www-services.yml playbook. Once the www services have been deployed the data will be sent automatically. If you wish to only send the data in the case a task has failed, you can do so by setting the `deploy` variable value to `send_files`:

`ansible-playbook -i docker.yml --vault-id dev@scripts/keepass-client www-services.yml -e "deploy=send_files -vvv"`

I recomment you use a triple vervosity level, it will allow you to see the execution of the async task, and if it has been executed properly. It's important to execute this task once the www services have been deployed since rsync is unable to create an architecture by himself. 

**Do rutine Backups with Cron Jobs**

This Ansible script can make a backup of certain files used for the deployment of our services. In order to have an updated version of these files we add a Cron Job to enable this process.

A Cron Job is a script or command that will execute in a regular basis. You can send it every minute, or once per week etc... To do so you should just add as a Cron Job the next command.

`ansible-playbook -i full_path/backup/inventory.yml --vault-id dev@/full_path/backup/keepass-client backup_local.yml`

This command is executed in the localhost, however some adjustments are needed such as:  installing rsync in both parties, and making backups of certain services to then retrieve their zip file. All of this adjustments are done automaticaly. *Backup needs access to root permissions*

## Auteur

[Alfonso Jimenez](https://github.com/henrei3) made this project during an Internship at the LISAH lab.

The project was tutored and his documentation reviewed by [Armel Thöni](https://github.com/Arthoni).

## Licencing

-

### To do

- [x] SSH Connexion
- [x] Dynamic Inventary Docker
- [x] Conf files
- [x] Hugo deployment
- [x] apache redirection of both websites: openFluid 8100 and community 8200
- [x] dloadsproxy deployment
- [x] use scripts for deployment (community - site-vitrine)
- [x] checkdload in Ansible (test proxy working after deployment task)
- [x] check update community service
- [x] choose to deploy one or more services
- [x] handler "remove site vitrine" gives errors
- [x] configuration apache dloads proxy
- [x] fichiers rss API proxy
- [x] architecture VM-1 (/srv/www/etc...)
- [x] config apache VM1
- [x] Change to HTTPS
- [x] Documentation Apache VM1 configuration
- [x] Add docker network config to ./init
- [x] Installation plugin ansible-vault. Gestion des erreurs
- [x] Ip statique docker network docker.ip. Change ./init to handle multiple ip.
- [ ] Voir gestionnaire de clés secrètes pour connexion avec une VM. AWS KMS
- [x] FluidHUB (hub)
- [x] team
- [x] Wareshub
- [x] Fix init script
- [x] keepass-client script restructure
- [x] Transfer all the necessary data for deployment
- [x] Backup
- [x] Config Gogs to use https
- [x] VM test deployment
- [ ] :tada:

[¹]: https://docs.docker.com/engine/install/ubuntu/
[²]: https://docs.docker.com/engine/install/ubuntu/#uninstall-docker-engine
[³]: https://docs.ansible.com/ansible/latest/collections/community/docker/docker_containers_inventory.html
[⁴]: https://keepass.info/help/v2_dev/scr_sc_index.html