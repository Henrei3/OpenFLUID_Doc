
# Automatization of OpenFLUID services

The goal of this project is to deploy in automatic manner, the entirety of OpenFLUID services.

These service are deployed in 3 different VM:
-  `openfluid-www`
-  `openfluid-prj`
-  `openfluid-hub` 

They are distributed in this form:

```
                          / --- team --- (proxypass http)[openfluid-prj]
                         /
                        /
                       / --- community --- (localhost)[openfluid-www]
                      /
                     /
WEB --- (https)[openfluid-www] --- www --- (localhost)[openfluid-www]
                     \
                      \
                       \ --- dev --- (localhost)[openfluid-www]              
                        \
                         \
                          \ --- hub --- (proxypass http)[openfluid-hub]

```
[¹]


## Deployment architecture Overview

### WWW (16-04-2024)

WWW would be deployed having cloned two git repos: being the community repository and the openfluid-web repository. Their data will then be reorganized and the git repository's eliminated.
The site-vitrine service (openfluid-web repo) and the community service will deploy their services inside the **/srv/www/openfluid-project.org/** folder. This is the actual architecture of original Virtual Machines.

### TEAMS

There is a single service deployed inside the prj Virtual Machine. This service being Gogs, the local git server. This service is started with the help of systemd through a file that we introduced in the systemd service files. The service is first deployed in localhost to then be redirected with apache to an actual secure URL that is redirected by the WWW reverse-proxy, this ensures the Integrity of the service since this many redirections act as security layers. 

### VM-Migration repository architecture

In the git repo of this automation project we have the option to execute the dev or prod playbooks. This project uses roles[⁸] to organize their tasks in a comprehensible and modular manner. Ensuring that it can be reused and refactored in the future by other programmers.

This is the full arborescence of the VM-Migration repository: 

```

   Database.kdbx
   ansible/
        backup/
            roles/
            deployment files
            Link to Prod Inventory
            Link to host_names.yml
            Symbolical Link to Original Scripts
        dev/
          roles/
          group_vars/
              docker/
                  IP_Adresses.json
                  Secret_Common_files.yml

          deployment files
          inventory (dynamic or not)
          Symbolical Link to Original Scripts
        prod/
          roles/
          deployment files
          inventory
          host_names.yml
          Symbolical Link to Original Scripts
        test/
          roles/
          group_vars/
          deployment files
          Link To Prod Inventory
          Link to host_names.yml
          Symbolical Link to Original Scripts
          
        Original Scripts
   docker/
      docker_compose.yml
      docker-entrypoint.sh
      Dockerfile
      init.sh

```

The Database.kdbx is an encrypted database that needs a symetrical key to be opened. It contains all passwords used to manipulate the encrypted vars with Ansible Vault. They are identified by their vault-id, which can be dev or prod.

All the diferent folders have different deployment files unique to their needs, their roles have a symilar structure however they are different in every folder, and need to be treated as separate entities. The only files that are identical in every folder (dev, test, prod, backup) are the "Original Scripts". These scripts are:

- keepass-client
- keepass-linux.sh
- keepass-mac.sh
- scp-send.exp

They are linked to the same folder as the deployment files for simplicity, the alternative would be to write their full path every time we execute the deploy command. Which would be more time consuming than just making a symbolical link. 

host_names.yml is the file that stores the encrypted IP Adresses of the real Virtual Machines where the deployment must be done, the encryption algorithm used is AES254, approved for Federal Government Standards as of March 2020[¹⁴].

The docker folder stores a docker_compose file that will a Dockerfile to build the container images and configure them with a docker-entrypoint (ssh user and connexion). The init.sh file executes the succession of commands to make that possible, it also creates the needed networks to do so, and stocks the IP adresses of the docker containers inside the IP_Adresses.json file in the `dev/group_vars/docker` folder.


## Recompilation

### Service "site-vitrine"


***keep the container open***

Before deploying the service on the actual VM (`openfluid-www`), it was necessary to deploy the services in a test environnement. The OpenFLUID team chose to do it with the help of docker. The first stage was to launch a debian12 (bookworm) docker image. Then to make the necessary configurations. However this docker container has the particularity of ending his process once all commands are done. It is thus essential to use a commands that keeps the container open, such as: 

` tail -f /dev/null `

This commands let's you keep the container alive until it's next utilization. A command like `sleep infinity` would do the same.

_It's quite obvious but it's important to know that apache2 package doesn't automatically install php_

***ssh configuration***

Angular is a tool that need to make an ssh connexion between the control node and the managed node. Thus, needing to have the python3 and ssh packages installed in the managed node too. Plus, our docker container doesn't let us by default have a root ssh connexion. We need to manually change the config files. These changes are written with help of [docker-entrypoint.sh](./docker/docker-entrypoint.sh)

***git clone with ansible***

Once the connexion ssh is done we can exec our playbook, we start by intall apache, php, hugo and finally git. We want to clone the project that hosts the static OpenFLUID website. But, there is a problem, how do you specify the password and username in a secured way ?

You can create a seperate file where you will stock all your varibles, Ansible has a lot of functionnalities related to variable sharing and stocking. We use **Ansible-Vault** to encrypt the sensitive data that is stored inside our variables, it is important to know that if you have access to certain variable values. You could have also access to the gogs server, understand in detail the architecture of the Virtual Machines, and you could have direct ssh access to the Virtual Machines. 

For instance the id and password variables that allow access to a certain repo will be stored inside an encrypted file using AES256.

`git clone https://{{identifiant}}:{{mdp}}@lien-du-repo.git`

***docker ip identification***

Our final deployment steps are to adapt the static website config files. But to do that you need to know the ip of node, in this case our node's ip is dynamic. To get around this obstacle we used the dynamic inventory plugin, docker_containers [²]. However this plugin doesn't use an ip for connexion, it uses the docker container name or id. So, it was necessary to create a script  that let's us get the container ip and send it as a variable. The commands were incorporated at [init.sh](/docker/init.sh).

***scripts folder in git repo***

There are certain deployment scripts located inside each service repository that are no longer useful since the deployment is done with Ansible. Although, there are in fact other deployment script that are very valuable to the deployment of their respective service, this Ansible script reuses them and then deletes them once the deployment is done.

This is the architecture of the scripts folder in the openfluid-web (vitrine) repo:

<pre><code>scripts/
       Dockerfile
       Helpers.py
       build
       checkdloads
       deploy
       relnotes-from-raw
       update-yaml
</code></pre>

- Dockerfile

The Dockerfile will install all the tools necessary for the deployment of the site-vitrine service in a docker ubuntu:16.04 environnement. It will install c tools, python, uswgi and ruby.
It will install a specified version of Hugo, the Saas Ruby Gem globally and flask with the help of pip. 

_It is no longer necessary_

- Helpers.py

this python script will set different Directory Path such as www, src, proxies. As well as functions like makedirs, execCommand and versionToNumeric. this will be usefull to our next python scripts.

_It is necesary_

- build

build is a python script and executes the `command hugo -d www_build_dir` with the Helpers function **execCommand**

- checkdloads

dloads is a proxy that let's the user download a file et notify the admin of it's download. the file checkdloads uses the ArgsParser Python library. It finds the newest openfluid version and checks that all files indexed by the yaml file located in ` www/data/downloads ` are available. To check it has to make a proxy call with the `dloadsproxy/check` url. we want to do the same kind of test with Ansible, we will probably reuse this file for this purpose.

_It is necessary_

- deploy

This file uses as well the ArgsParser library. It gives the option of deploying all OpenFluid websites ( site-vitrine and community ). Only the site-vitrine website or only the community website, the proxy services (api and dloads) or only build the service.

_It is no longer necessary_

- relnotes-from-raw

This file treats text, I assume it was previously used to automaticcaly create release notes in `src/www/data/releases` but the question is what are the raw files ? And are we going to need this script in the future. Either way relnotes-from-raw is deprecated since it's a Python2 script. I suppose it is not useful rigth now.

_Could be necessary to Change to Python3_

- update-yaml

This is a very usefull file that we will be reusing for the deployment of site-vitrine. This Python3 script uses a config template to generate whatever config file whe need. In our case that would be a prod config file and a config file used for local deployment. 

_This one sparks joy_

### Service Community

***python mkdocs dependency's***

The community service is a documentation website deployed with the help of Mkdocs, a very usefull and popular tool used for generating documentation from .md files (like this one). To install Mkdocs in our debian12 device we would have to install python an pip module first since mkdocs works with python modules to deploy websites. On top of needing mkdocs installed we need some extra modules to enable the `build` fonctionnality of mkdocs these modules are :

- mkdocs-markdownextradata-plugin
- pymdown-extensions
- markdown-include
- jinja2
- bibtexparser

From this modules the most importants are : `markdownextradata` that let's us use custom templates for our site. And `pymdown-extensions` that are used all along the deployment of the site.

**configure and deploy mkdocs**

To configure the website there is actually just one variable we have to configurate and that is `site_url` we use the module ansible.builtin.lineinfile to configure this variable in an automated form. To deploy the website we just need to execute the command:

`mkdocs --build` and a new folder named `site` will deploy. Having all of the necessary files to index the website with help of apache2.

***realocation of "site-vitrine"***

Once the website is deployed we  asked ourselves if the current architecture would continue being logical and optimal for our deployment. Since we already had a website in the root folder of apache. We would have to create another folder inside the root folder of apache to host our community website. Which would complicate a lot of things in the next steps of our deployment and is very disorganized architecture. So the other alternative was to deploy our site-vitrine website in a separate folder inside apache's root. That way we would have an architecture like this :

```
                         [ / ]
                           |
                 _____________________
                |                     |
         [ site-vitrine ]       [ community]
``` 

We had to change the automation script of both services but it was quite straightforward and dind't pose any major complication.

***configure apache2 redirection***

Finally, once the deployment had been succesfully automated we needed to make a redirection that in a production environnement would be to the domain site of community and site-vitrine respectively. In our case since we are testing it's deployment. We would like to start a redirection only by changing the port. The port 8100 would redirect to the site-vitrine website and the port 8200 would redirect to the community website. The solution was quite simple we would just set the `DocumentRoot` of a VirtualHost with the respective port to the location of our websites. then we should : `service apache2 restart` or just start the service when automating it, to enable our configuration.  

***Proxy Deployment***

For deploying the proxy we had to intricately comprehend how the proxy acts and it's apache configuration. We learned that there multiple proxys for the vitrine service (API Proxy and dloads Proxy). Dloads proxy sends an Email to the administrator to notify a user download and saves the download in a sqlite3 database as well. Both proxys use the apache WSGI module for deployment.

The dloads proxy uses a .ini configuration file to save sensible information such as SMTP passwords or the resources and sqlite DB path. Inspired by the original deployment structure. I will create a Jinja2 template to complete a local config file and copy it to the managed node.

Give the right permission to the folder where you want to put the database.

I will then deploy all my websites inside the root folder including the proxy and then delete all git repos because they're not necessary. Ansible handlers are used for doing so.

***scripts folder in git repo***

Certain git repo scripts have been reused in our Ansible Script, they are then removed from the deployment once they have served their purpose.

The reused scripts are:

- update-bib
- update-compat-matrix
- update-faq
- update-yaml

This is the architecture of the scripts folder in the openfluid-community repo:

<pre><code class="language-text hljs plaintext">scripts/
       Dockerfile
       Helpers.py
       build
       update-bib
       update-charter-pdf
       update-compat-matrix
       update-faq
       update-yaml
</code></pre> 

- Dockerfile

This Dockerfile creates a ubuntu 16.04 container, it installs python and wsgi as well as mkdocs markdown-include bibtexparser and Jinja2 with pip.

- Helpers.py 

this file is similar as the Helpers.py we find in the openfluid-web repo. Although it has some additional functions: 
getGeneratedWarning, this function is used in multiple scripts in the scripts folder, it is used to create a warning a certain file has been generated automatically. 

Slugify is a function that was cerated to reaplace a library since the library is to big ans we are going to use only this function of the library. 

generateTableOfContent as his name states it, it generates a table of content of a markdown file, this function is then used to automatically generate a faq.

- build

a simple python script that build the community site in the `_build` folder created automatically. 

- update-bib

This file will update the bibliography of the community site, in our use case it will be executed everytime we deploy the site. Here is a graph explaining how it works:

<pre><code class="language-text hljs plaintext">
       references.md       <-- update-bib <-- Helpers.py (getGeneratedWarning)
                                   |
                                   |
        __________________________________________________________
       |                           |                              |
OpenFluidCiting.bib         OpenFluidRefs.bib           References.tpl.md
</code></pre> 

- update-charter-pdf

This is a a python scrpt that uses the chromium command part of Chrome Developer Tools to convert to pdf an html site. In our case this html files are located at `/site-vitrine-root-site/charter/` This file makes a call to site-vitrine to retrieve the html files and them transforms them into pdf. 

- update-compat-matrix

This python scripts updates the system-comptability.md file using a template and a yaml file. Here is a graph explaining how it works:

<pre><code class="language-text hljs plaintext">
system-compatibility.md  <-- update-compat-matrix  <-- Helpers.py (getGeneratedWarning)
                                          |
                                          |
               __________________________________________________________
              |                                                          |
       compatibility-matrix.yml                                compatibility.tpl.md
</code></pre> 

- update-faq

This python script will update the FAQ (Frequent asked questions) thus creating or updating the faq.md file in `docs/general/`. Here is a grpah explainig how it works:

<pre><code class="language-text hljs plaintext">
       faq.md       <-- update.bib <-- Helpers.py (getGeneratedWarning
                                                   slugify
                                                   generateTableOfContent)
                            |
                            |
                            |         
                     faq-source.md
</code></pre> 

- update-yaml 

Similar to the python script in site-vitrine git repo it will create as many config files as added in it's list with help of a template file. And save them inside the `src/` folder. The script will create two config files:  one for production and another one for testing in it's actual state.

### Apache VM1 Configuration

**error handling**

Ansible let's you handle errors with [Blocks](/EN/README.md#essential-functionnalitys) similar to a `try` clause they let execute a task if another one has failed. In our case we had to remove all the different repos that were created. Even if the task fails those git repositories will be removed. this allows us to re-execute the task without any inconveniences in a renewed environnement. In the case all task have been executed properly, this means that the deployement has been done, all important files have been allocated into their respective place and the git repos can be safely removed.  

**configuration files**

In order to exactly replicate the apache configuration that the `openfluid-www` has. We use the sites-enabled/sites-available architecture. Having a very readable list of configuration files, each dedicated to one of the OpenFLUID services. As explained in the internal documentation[¹] the `openfluid-www` VM redirects to the services that are hosted on another VM through a proxy. Allowing them to be accessed via a domain name. The services that are hosted inside the `openfluid-www` VM have each a different domain.name or alias asigned to them. Our ansible script pastes the config files templates and enables all of the available sites in order to deploy the services. The only difference being the log file names for each site and the domain names that are replaced by IP values in the dev ansible script. 

**docker network**

The prod VM1 configuration uses domain names to redirect the user to the different sites. (vitrine, community, team, dev, hub). However, it isn't as straightforward to replicate this in a Docker environnement. After doing some research. The `openfluid-www` VM needs to host 3 different domain names, or in our case three different IP's. I ended up figuring out that it wasn't as difficult as I though, we just needed to create, or use three exisiting networks having different subnets[¹¹]. This would assign three different IP's to your docker container, letting effectively host docker multiple sites. The other sites such as team and hub are deployed in separate VM's this means that in a docker environnement, we would have to deploy other containers with separate IP adresses.

**docker.ip deprecated**

In the past, our ansible deployment script used a generic variable called docker.ip in order to replicate the apache configuration. But this configuration was done with aliases and not with domain names. Since using different domain names would mean that we would have to use different IP adresses. However it is possible to assign multiple IP adresses to a docker container using docker networks[¹¹]. This does in fact render the variable docker.ip useless. Well... not quite, we won't be needing one variable, but multiple since we use multiple docker networks, one for each domain name. For this reason we have now variables such as docker.ip1, docker.ip2 in our group_vars/docker/IP_Adresses.json file.

Note: This variable is only useful for the dev environnement. Since domain names aren't used for the docker deployment.

**https certificates**

In order to deploy our websites in https we need a certificate. In the prod environnement this won't be a problem since they are generated automatically by let's encrypt. However in the dev environnement we need to create our own and copy it into the managed node. To generate the certificate we used openssl, the certificate name is www_openfluid_org.crt. We also generated a self-signed Certificate Authority (CA).

### Gogs service deployment

**apache configuration**

This service and the PRJ VM is only accessible in the LAN sector. However, thanks to the WWW VM, we can do a reverse proxy redirection so that it is accessible through https and a domain name. The PRJ VM, also includes a reverse proxy redirection to locahost:3000, the url where gogs is hosted inside the local machine.
The gogs service is treated as a systemd service. This is done by connecting a systemd prebuilt gogs script that is part of the [binary installation](https://gogs.io/docs/installation/install_from_binary). To start the service we have to use systemctl, this is done automaticaly by Ansible prj role.

**deployment**

It is strongly recommended to deploy the services via a docker instance first, to test that the deployment has been executed properly in a certain debian environnement. The actual environment is BookWorm (debian12). Also to understand how it will work in a real environment.

**configuration**

Gogs is a very resilient, fast and optimal service, however the only thing it lacks is password management via the configuration files. This makes impossible to automatizate the admin user creation. Once the services have been deployed by `prj-service.yml`. You will have to connect to the service through his domain name, or docker IP address and state your admin credentials and password manually. You should also change the Application URL to the Domain name / Docker IP address.  It is very important to change the Application URL to the domain name, since this will affect the way that gogs sends data. Either via HTTPS or via HTTP.

**Configuration files**

This deployment script has 3 main configuration files that enable a proper gogs deployment:

- openfluid.conf

This file hosts all the apache configuration, including the redirection to localhost gogs server and the definition of a root directory.

- app.ini.tpl

This file is the configuration file for gogs, it set up a file structure for gogs data and all the information that the service might need. 

- gogs.service.tpl

This file is the systemd service file, it is stored as default in the gogs github repository. However, some changes needed to be added in order for the service to work in our environment. 

**How to backup Gogs**

Gogs offers the possibility to backup their system automatically. you simply have to use the gogs executable and write the command: 

`./gogs backup`

This command may difer depending in your use-case, please refer to this github discussion on the subject: [How to backup, restore and migrate](https://github.com/gogs/gogs/discussions/6876).


### FluidHUB service Deployment

The FluidHUB service can be deployed in two modes: `Production` and `Development`. So, logically to deploy it in `dev` mode you should execute the `hub-service.yml` file inside the `dev` folder or the `all-services.yml` file that deploys all services.  

The environnement variables that you must set before deploying are writen in the [fluidhub.env.tpl](/ansible/dev/roles/hub/templates/FluidHUB/fluidhub.env.tpl) file. Keep in mind that if there are import errors once the deployment has been executed, it means that there is a variable that you didn't set. Check the `LDAP` variables too.

### Micro Services Deployment


**gittest**

The gittest micro service will be usefull to clone git repos and test certain git clone functionnality's. This micro service uses WSGI to execute, and all it's error logs are stored in separate log file. In the dev environnement this log file is called `dev_error.log`. To git clone a repo using this micro service in a `dev` environnement. You must set the `GIT_SSL_NO_VERIFY` variable to `true` and then git clone the project, since our dev environnement uses a self-signed certificate to enable https, this is the only way git will clone the repo. The full commend should be:

`GIT_SSL_NO_VERIFY=true git clone https://your-repo-url`

There exists 3 main ways to copy the auth, standard, and empty bare git repos inside the managed node. We can use the `with_filetree` option, that traverses all specified folders in a recursive manner. We can use the `copy module`, where we just need to specify the main folder and it will copy recursively the folder in it's destination. And finally we can use the `scp` option that uses an ssh connexion to send the files. they respectively take: 13 minutes, 3 minutes and `1 second` to execute. Since we are going to send much bigger chunks of data that just 3 git bare repos. It was absolutely necessary to find an optimal way of sending information. 

To handle complicated actions that require user input, such as adding an ssh key we can use the expect command. It is an executable that, just like awk. It let's you create scripts that handle complicated user case commands. The main commands of expect are :

```
expect
send
spawn
puts
```

You can learn more about expect [here](https://jdhao.github.io/2019/10/29/expect_script_learning/).

It is also possible to use rsync to send data. However, it was once though that our use case wouldn't need to exploit the advanced use cases rsync offers, but we are iuf fact in need to send big heavy files to up to 2GB, which means that scp is no longer an option. It is possible to use the synchronise ansible module, but it doesn't let us customise rsync in the way we want, knowing that our use case can get even more complicated in the future. An expect script is still needed to handle rsync

We use the `shell` ansible module accompanied by the `delegate_to` keyword to execute this script in localhost. The expect script takes as input the keypass, src, dest, host and hostpass.

If the data is not sent but the command doesn't throw an error. use the `-v` flag to debug. I strongly recommend to use the `-v` flag at all times, this way you can always catch the output of an error. For more details you can even use the `-vvv` flag.

To ensure that our services deploy even if this unsafe task fails, the script will execute the slower but safer copy module to send our data in the case scp-send.ext fails. Please note that for longer files (1GB or more) this process can take up to an hour, I recommend to fix the expect script and retry the deployment.

### File Backup 

In order to deploy all the Openfluid services, this ansible script needs to be executed preemptively. For the moment, we only need to backup 3 folders that belong to the Virtual Machines, them being:

- The wareshub data
- The gogs repositories
- The gittest test repositories
- The OpenFLUID banary files

To deploy the rest of our services the gogs service must be up and running, this is done to ensure that the git repositories can be retrieved even if the original service is down. 

The wareshub data is essential to deploy wareshub, it stores all the configuration files, the code that executed the wareshub app and also the git repositories of the vast majority of Openfluid modules. 

The gittest repositories are needed to deploy the gittest micro-service. Even though it is not essential for the Openfluid environnement, this ensures a full deployment.

The binarie files will then be retrieved by the dloadsproxy and send to the main OpenFLUID website user.

The backup functionnality is at the moment, complicated to execute and needs to be done in multiple commands. I will describe the actual steps to do a backup in the dev environment:

You will need to do a backup with prod credentials, since all the sensible data is stored in prod devices, the backup should be done in the prod folder. To execute a backup simply use the command specified in the [USER](/README.md#backup-with-ansible) guide. Once that is done you should go ahead to the dev folder and deploy the www-services with triple vervosity. This ensures that you understand what happens in the async tasks that are going to be executed once all the www services are deployed. Async tasks do not notify of rsync errors, they are just considered as "changed" by Ansible.

If you wish to add another file to backup you have to understand how the backup works:

The backup is done through a expect script that executes an rsync command `rsync -a $src $dest`. This is a quite simple command, but we do it with expect because it can get complicated real fast and if we wish to add more intricate data management to rsync, we could simply change the expect script instead of a whola Ansible module.

all variables in target vars shouls be in this format:
```
  - { service: "", src: "", dest:"", host:"", keypass:"", hostpass:"" }
```
**service** is not used for the script it just makes it more human readable and easy to identify
**src** is an important variable that just change depending on wether you want to push data or pull data `man rsync` for more information
**dest** is the second variable used directly for the rsync command
**host** is a redundant variable however really important for the expect script execution, it is used to remove the host from the known hosts, this way it is adapted for a dynamic inventory such as docker.
**keypass** this is the variable that contains the password for your private key to unencrypt in order to connect through ssh.
**hostpass** This is the host password, also used to connect through ssh.
 
## Essential Functionnality's

Here is list off all essential functionnality's to understand the environment of the project.

***variables et modules***

Ansible let's you use two formats to create playbooks, either yaml or json. Both of them are conceptually the same, yaml can be transformed into json and vice-versa. Both formats work by creating variables, lists or dictionnaries. Some of them are recognized by Ansible such as - name or task: . The rest of the variables that don't get recognized as key words for the playbook are recognized by ansible ase reusable variables. You can even save the return value of a task with the keyword `register:`, you can then use the variables with `{{var}}` this nomenclature isn't unique to Ansible, it's standard yaml nomenclature. You can find the same syntax in Gitlab pipelines.

Modules are unique to Ansible, they can be part of the core ansible installation, such as the shell, debug or apt module. But there are much more modules accessible with ansible-galaxy (the community par of Ansible) these modules can let you connect with routers with Cisco.Ios [³]. Another one is the community.docker [⁴] module, usefull for a stage of this service deployment.

***module community.docker.containers***

The community.docker.containers [²] module, part of community.docker [⁴] is a dynamic inventory plugin. That let's you create an automatic connexion with docker containers. Really usefull for our first stage of this service deployment.

Syntax :

`plugin: community.docker.docker_containers`

`docker_host: (par default) unix://var/run/docker.sock`

***python interpreter***

Ansible uses python to send, execute and interpret playbooks. Bu default Ansible is going to search multiple possible interpreters and unse the latest version [⁵]. However, there are times that Ansible Playbook can't find the correct interpreter or uses the wrong one. In this case it's important to manually set the correct interpreter.  

`ansible-playbook -i your-inventory.yml your-playbook.yml -e "ansible_python_interpreter=/usr/bin/python3"`

another usefull variable `discovered_interpreter_python`

***loops***
Ansible let's you create loops for any task that you have written. This let's us have a more compact and comprehensible code that in the long run would also be easier to refactor. Loops syntax :

```
 task: 
       - name: example task
              ansible.builtin.apt:
                     name: {{line}}
                     status: latest
              loop: 
                     - somePackage
                     - anotherPackage
```

This would be the equivalent of doing two tasks with two names. Please note that this is an example since there are more optimal ways of doing this specifical task.

***roles***

Roles let you automatically load related vars, files, tasks, handlers, and other Ansible artifacts based on a known file structure. They are easy to share via Ansible Galaxy  and easier to understand when it is a complex use case.

A role architecture is separated in different files:

<pre><code>
roles/
       _role-name_/
              tasks/
              handlers/
              templates/
              files/
              vars/
              defaults/
              meta/
              library/
              module_utils/
              lookup_plugins/
</code></pre>

In each role subfolder, Ansible will be looking for a main.yml .yaml or just main to load the tasks,vars,files etc.. This is useful for segmenting our Ansible deployment in comprehensible parts. We can compare them to classes or functions.

***handlers***

Handlers are tasks that can be trigered once a certain task has been acomplished. This behaviour is useful when we want to always let in a certain state the managed Node. Once all the tasks have been executed. No matter the result of said tasks. By default handlers are executed once all task have been executed. Notified handlers are executed automatically after each of the following sections: `pre_tasks`, `roles`/`tasks` ans `post_tasks`. This is done to eliminate redundancy in the case we call multiple times a handler. If we want to change this behaviour we could add a a task to flush using the meta module[⁶]:

```
tasks:
       - name: Some task
         ansible.builtin.shell: test
         notify: "execute handler"
       - name: flush handlers
         meta: flush_handlers

handlers: 
       - name: Some handler
         ansible.builtin.shell: echo "Handler executed !"
         listen: "execute handler"
```
[⁷]

***ansible-vault***

To handle secrets, Ansible offers the posibililty to encrypt and decrypt in a secure manner using Ansible vault.

This system is compatible with ansible playbook making it possible to encrypt our variable files and still be able to execute our playbooks. 

To encrypt a file you use the command:

`ansible-vault encrypt foo.yml`

to decrypt :

`ansible-vault decrypt foo.yml`

You can also encrypt/decrypt multiple files concatenating them command shown below.

***Ansible Vault : vault-id***

To encrypt an file you have to supply a password. You can either be prompted to write it in the terminal. Keep it raw in an unencrypted file or create a script that outputs in stdout the password. Making it possible to connect Ansible Vault to your favorite password manager keyring. See this [script](../ansible/dev/keepass-client) that connects Ansible to KeePass2.

Ansible will prompt your script like this:

`your_script --vault-id your-vault-id` if you call ansible vault with the `--vault-id your-vault-id@your_script` flag. Don't forget to add `-client` or `-client.EXTENSION` to let Ansible understand this script is used for Ansible Vault and correctly send the `--vault-id` flag.[⁹]

To encrypt a file using ansible vault and a script connecting to your password manager, you should use the command:

` ansible-vault encrypt --vault-id dev@your-script-client /path/to/your/file `

Please note that dev is your vault-id.

***Blocks***

Blocks create logical groups of tasks, making it possible to handle errors. You can add all sorts of data or directives common to a group of task. As an example you could add a when directive to your block. You can use block with `rescue` and `always` sections to handle errors.Similar to a try in Java; The `rescue` section executes, if one of the block tasks failed. And the `always` section always executes even if an error is thrown.

Ex Given By Ansible Documentation [¹⁰]:

```
 tasks:
   - name: Handle the error
     block:
       - name: Print a message
         ansible.builtin.debug:
           msg: 'I execute normally'

       - name: Force a failure
         ansible.builtin.command: /bin/false

       - name: Never print this
         ansible.builtin.debug:
           msg: 'I never execute, due to the above task failing, :-('
     rescue:
       - name: Print when errors
         ansible.builtin.debug:
           msg: 'I caught an error, can do stuff here to fix it, :-)'
```

***Groups and group_vars***

Ansible is an incredibly versatible tool, that can be used to an enterprise scale. This means that it's inventory handling is pristine. The way Ansible connects to a remote node is by handling an Inventory. An inventory is a file where you will store all your host information, needed to execute a connexion. You can create a static inventory with it's groups in this manner :

```
usa:
  children:
    southeast:
      children:
        atlanta:
          hosts:
            host1:
            host2:
        raleigh:
          hosts:
            host2:
            host3:
      vars:
        some_server: foo.southeast.example.com
        halon_system_timeout: 30
        self_destruct_countdown: 60
        escape_pods: 2
    northeast:
    northwest:
    southwest:
```

This inventory example provided by Ansible [¹²] will create a main group: USA. And assign to them children groups until arriving to the hosts. They can be assigned vars in static way too.

If you want to stock your group vars inside a file, Ansible let's you do it by creating a folder named `group_vars`, Ansible will search for this folder name in the directory relative to your inventory or your playbooks[¹³]. If found it will match your var file names (.json, .yaml, .yml or no extension ) to your inventory groups. It is also possible to create directory's to stock multiple variable group files.

```
/etc/ansible/your_inventory.ini (yml, yaml)
/etc/ansible/group_vars
                |
                ---- group1/
                         |
                         ---- File.yml
                         ---- File.json
                         ---- file.yaml
                     group2/
```

If you want to create specific host vars, you should create the `host_vars` folder.

## Usefull apache configuration data

To see apache2 full configuration use the `apache2ctl -S` command

| Command | Description |
| --- | --- |
| `Include` | Adds a configuration file to tha apache conf |
| `<Directory>` | configuration pour un repertoire particuler |
| `DocumenRoot` | specifies the position of your website |
| `ErrorLog` | specifies the position of the error logs |
| `Define` | Define an environnement variable |
| `VirtualHost` | Contains directives that apply only to a specific hostname or IP address |

**Sites enabled / Sites available**

The `sites enabled` and `sites available` directories are folders inside the `/etc/apache2/` configuration. They are used to maintain readability and keep easy to understand complex apache architectures.

When you add a new site to your apache2 server you should add all the necessary configuration inside a configuration file located in the `sites-available` folder. Once you have done so and would like to enable the site, you can simply create a symlink of your conf file to the `sites-enabled` directory or use the command `a2ensite`. 

This is done to ensure coherence and signifanctly reduce configuration errors. Only the `sites-enabled` directory is included in the `apache2.conf` file. If you edit your configuration file you should always edit it in the `sites-available` folder, any change to the `sites-enabled` folder can cause an unexpected configuration error which could take a lot of time to solve.

You might read in some forums that this architecture is deprecated, they are refering to the Nginx architecture that tries to replicate the Apache architecture. The NGINX packages moved away from this concept many years ago.

### Vars file specifications

In the case you want to create another var file or add a variable it is important to understand what variables are used in the deployment playbook. All this variables can be found in the vars folder inside the main.yml file, this folder is encrypted.

If you wish to edit this folder you can do so with the `ansible-vault edit` command. Before doing so please refer to the Ansible guidelines on [how to secure your editor](https://docs.ansible.com/ansible/latest/vault_guide/vault_encrypting_content.html#steps-to-secure-your-editor)

| Variable | Description |
| --- | --- |
| `git.user` | This variable is used to clone some https git repos note that user is a subindentation of git |
| `git.pass` | This variable is used to clone some https repos |
| `docker.ip` | [deprecated] This variable is changed through the deployment script of docker each time the machine is launched since it's ip is variable |
| `docker.ipX` | This variables are created or replaced everytime the [init](/docker/init.sh) script is executed, it will create one variable for each IP address that the deployed Docker containers possess  |
| `hugo_config` | This variable is used to execute a loop with the module lineinfile in the site vitrine task |
| `mkdocs_config` | This variable is used to execute a loop with the module lineinfile in the community task |
| `tpl` | all variables nested inside a var name tpl, are variables exclusively used in a template located in the templates folder inside a role |
| `domain.` | The domain variable is defined in the www role, it is used for the mainn Virtual Machine apache configuration, this also meaning the reverse proxy. |
| `gogsDN` | If you wish you can specify this variable or register it during a task so that the Script uses this domain name, or IP Address to git clone the necessary repository's inside the gogs service, by default the gogs domain name is: `team.openfluid-project.org` |

Please be mindfull in the syntax and indentation of the vars. Especially `hugo_config` and `mkdocs_config` since both of them need to have exact keynames and are case sensitive. In the production script you will find this variables in the mains.yml file in the vars folder. Nonetheless, in the dev environnement this variables are visible in the code, this structure supposes that you will copy only the dev or prod part of the code in a device. However this is supposing that the deployment of the services get's automated and done in a routinely basis.  

**KeePass Database vars**

It is important to note that the intended way to use the KeePass-Ansible_Vault script. Is by creating variables inside your KeePass(2/XC) Database that hold your password info. However you should **always** name their title and username the same. This is done to ensure a correct and distinguishable ID'ing of this vars, so that the script selects the correct var.


## How to continue this project

I would like to add, as the author of this project, the different functionality's that could be added in order to improve this project and ensure that it is usefull for the years to come. 

But first, I think it is important to note that this project can not do very much in it's actual state. It can deploy an environment in a collection of Docker containers, it will eventually, with some fixes, be able to deploy the same environment in the Virtual Machines that host the real services. And it can make a simple backup of a selection of files with rsync. 

In order to propose this improvements we will take a critical approach of the actual state of my project, and state from my point of view what would add value to this Script.  

**Use Kubernetes ?**

In order to scale this project and make it usefull for as to scale the entire environment, it could be a good idea to deploy the service in a docker swarm, or even better a kubernetes cluster, this would improve the scalability of the environment and it's security, since kubernetes handles very smoothly a vast amount of security protocols. The environment would also be deployed in a contained environment, this means that if it gets infiltrated, not much harm could be done. I understand that the needs of the OpenFLUID services do not amount to the effort that this transformation will need. However, for academic purposes I think it usefull to note that it could be possible to do it with Ansible, thus, reusing my script. 

**Use Amazon KWS Manager ?**

This Ansible Script uses a Symetric Key generated by the Application Keepass2 to Decrypt in an indirect way all the different sensible data necessary to the OpenFLUID services deployment. For the moment this key is stored in physical USB drive, this is a HUGE security concern, since the USB key could be lost, or corrupted or stolen and all this data would be compomised. Not to forget that it's not that practical to need an USB stick everytime you need to deploy the services. A solution for this problem is to share the key in a secure server so that it could be retrieved online, this way it can't be lost. Moreover, this again causes some security concerns. The "professional" way of doing this, from my point of view would be to use an existing solution. And what better solution than Amazon KWS Manager, for only an Euro per Month, this services offers the access to the KWS API that let's you encrypt/decrypt data free (up to 10 000 calls per month). The intended way to use it would be encrypting a symmetric or asymetric key, and decrypting it when needed. This would mean that the key could be stored with the code and decrypted through the KWS API, upping the security of the environnement to a corporate standard. Again, not needed for this environment. However, interesting to point out. 

**Secure the Backup ?**

The backup is done through rsync, which is in fact very secure. Even so, once the data has been transfered, it isn't encrypted and can easily be accesed by an intruder. Not to mention that it is locally stored, and it can at any time be corrupted, removed or stolen. It is not a secure way of doing a backup. What could be done to implement a more secure backup would be to seclude it from the network, this means that we would only access the network for doing our routine backup and then inmediately disconnect, this way we instantly remove any digital threat posed to our backup data. Data should be secured in all of it's states, them being:

- At-rest
- In Use
- In transit

This meaning that it should be secure when we create,read,modify and delete the data. When being stored in a device, and when being transfered to another device. For the moment the data is secure while being transfered. So, how do we secure it in it's other states ?

For complex use-case one solution could be using an existing solution, or even creating our own solution. But what I propose is something way simpler, we don't fully secure it, we just secure it enough so that it deters most attackers. To do that we just need to secure our data at one more state: At-rest. For this, we should encrypt the data right after receiving it, and seclude it from the internet. We can even keep the backup script inside a Kubernetes cluster that will connect to a device in demand and will inmediately disconnect. Or just store the encrypted data in a physical drive. You could even store all data inside a dedicated Hard Drive (this option would actually be prefered). 

**Increase OS Compatibility ?**

This script is meant to be compatible for Unix devices. However there have only been test for MacOS and Ubuntu. It is very likely that if this script is executed in other OS than Ubuntu or MacOs, it throws an error, or worse the script enters an infinite loop. It would be also usefull if this script is adapted to windows users, although it means that there should be a Windows and Unix version of the script.

[¹]: doc.interne.OpenFLUID
[²]: https://docs.ansible.com/ansible/latest/collections/community/docker/docker_containers_inventory.html#ansible-collections-community-docker-docker-containers-inventory
[³]: https://docs.ansible.com/ansible/latest/collections/cisco/ios/index.html
[⁴]: https://docs.ansible.com/ansible/latest/collections/community/docker/index.html
[⁵]: https://docs.ansible.com/ansible/latest/reference_appendices/interpreter_discovery.html
[⁶]: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/meta_module.html#meta-module
[⁷]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html
[⁸]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html
[⁹]: https://docs.ansible.com/ansible/latest/vault_guide/vault_encrypting_content.html
[¹⁰]: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html#handling-errors-with-blocks
[¹¹]: https://docs.docker.com/reference/cli/docker/network/create/#specify-advanced-options
[¹²]: https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#inheriting-variable-values-group-variables-for-groups-of-groups
[¹³]: https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#organizing-host-and-group-variables
[¹⁴]: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-175Br1.pdf