---
Attention !: |-
  Cette documentation n'est pas complete. Vous devez vous réporter a la documentation en anglais qui est à jour et complète. 
---


# Projet automatisation services OpenFLUID 

Ce projet est réalisée au sein du laboratoire LISAH pendant un stage de 4 mois. L'objectif est de pouvoir déployer de manière automatisée la totalité des services utilisées pour la plateforme OpenFLUID.

Ces services sont déployeés dans 3 VM differentes:
-  `mtp-lisah-openfluid`
-  `024-1221-openfluid-prj`
-  `024-1221-openfluid-hub` 

Les services sont respectivement distrbuées de la manière suivante:

<pre><code class="language-text hljs plaintext">                          / --- team --- (proxypass http)[024-1221-openfluid-prj]
                         /
                        /
                       / --- community --- (localhost)[mtp-lisah-openfluid]
                      /
                     /
WEB --- (https)[mtp-lisah-openfluid] --- www --- (localhost)[mtp-lisah-openfluid]
                     \
                      \
                       \ --- dev --- (localhost)[mtp-lisah-openfluid]              
                        \
                         \
                          \ --- hub --- (proxypass http)[024-1221-openfluid-hub]

</code></pre> 
[¹]

Voici une recompilation des étapes qui se sont effectuées jusqu'a la réalisation du projet. Ainsi qu'une explication du fonctionnement clé du projet et des fonctionnalités indispensables à comprendre.


## Historique du Projet
### Service site-vitrine (11-04-2024)
***maintenir le container ouvert***

Avant de déployer le service sur la machine virtuelle correspondante (`mtp-lisah-openfluid`). C'etait nécessaire de déployer les services sur un environnement de test. L'equipe OpenFLUID a donc choisit de le déployer sur docker. La prémière étape fut donc de déployer un environnement debian12 (bookworm) sur docker. Puis de faire les configurations nécessaires. Pourtant le container docker ne va pas se maintenir active sur ce type d'images. C'est pour cela que c'est nécessaire de faire la commande suivante a la fin de tout script executée sur le container docker:

` tail -f /dev/null `

Cette commande permet de maintenir occupé le container jusqu'a un prochain appel. C'est l'equivalent de faire `sleep infinity`.

_Il semble evident mais c'est important a savoir qu'apache2 n'installe pas php automatiquement._

***configuration ssh***

Angular est un outil qui a besoin de faire une connexion ssh du control avec le node managé. C'est un outil basé sur python, c'est donc essentiel d'avoir ssh et python installé (python3 de préférence). De plus, notre container debian ne permet pas par défault la connexion ssh par root, on doit donc changer certains fichier configuration. On écris donc ces commande dans notre fichier [docker-entrypoint.sh](./docker/docker-entrypoint.sh)


***git clone avec ansible***

Une fois la connexion ssh à été faite on peut executer notre playbook, on commmmence par installer apache, puis php, puis hugo (outil de déploiement de sites statiques) et git. On veut clone le projet qui possède le code et les configurations permettant de déployer le site web vitrine d'OpenFLUID. Une problèmatique se pose, comment spécifier le mot de passe et l'utilisateur de manière sécurisée ?

Il suffit uniquement d'utiliser des fichiers ou les variables seront stockées, ces variables pourront même être stockées avec **Ansible Vault** dans un futur. Puis importer le fichier avec `var_files` avant de spécifier les `tasks`. On met ensuite les identifiants sous fourme de variable: 

`git clone https://{{identifiant}}:{{mdp}}@lien-du-repo.git`


***identification ip docker***

Le pas finale pour le déploiement du site web est d'adapter les fichier de configuration à nos besoins. Pour cela il nous faut savoir l'ip que notre container docker utilise. Hors cet ip est dynamique, il change lors de chaque déploiement. Pour contourner cela lors de la connexion avec Ansible on utilise le plugin d'inventaire dynamique docker_containers [²]. Néanmoins, le plugin ne fait pas une connexion avec l'ip du container sinon avec le nom ou id du container. C'était donc necessaire de créer une succession de commandes qui nous permettent de récuperer l'ip du container docker et de le mettre dans le fichier de variables dont on avait auparavant parlé. Les commandes ont été incorporées dans [docker-entrypoint.sh](./docker/docker-entrypoint.sh).

***scripts folder in git repo***

It is no longer useful since the deployment is done with Ansible
Although I have looked at the deployment scripts and tried to replicate it in the closest possible way with Ansible.

This is the architecture of the scripts folder in the openfluid-web repo:

<pre><code class="language-text hljs plaintext">scripts/
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
- Helpers.py

this python script will set different Directory Path such as www, src, proxies. As well as functions like makedirs, execCommand and versionToNumeric. this will be usefull to our next python scripts.

- build

build is a python script and executes the `command hugo -d www_build_dir` with the Helpers function **execCommand**

- checkdloads

dloads est un proxy qui permet à un utilisateur de telecharger un fichier et de notifier a l'administrateur du telechargement. Le fichier checkdloads utilise la librarie argsParser. Il va trouver la version la plus nouvelle de openfluid et va verifier que tous les fichiers soient accessibles depuis le proxy. Pour cela il va trouver tous les fichiers indexes dans le fichier ` www/data/downloads ` et faire un appel dloadsproxy/check/. On souhaite realiser le même test sur Ansible c'est donc intéressant de reutiliser ce script python à futur. 

- deploy

Ce fichier utilise aussi la librairie python argsParser. Il va donner la posibilité de déployer tous les sites openfluid (community site-vitrine). Uniquement le site vitrine ou uniquement le site community. Les proxy (api et dloads) ou uniquement de build le service. 


### Service Community (15-04-2024)

***python mkdocs dependency's***

The community service is a documentation website deployed with the help of Mkdocs, a very usefull and popular tool used for generating documentation from .md files (like this one). To install Mkdocs in our debian12 device we would have to install python an pip module first since mkdocs works with python modules to deploy websites. On top of needing mkdocs installed we need some extra modules to enable the `build` fonctionnality of mkdocs these modules are: 
- mkdocs-markdownextradata-plugin
- pymdown-extensions
- markdown-include
- jinja2
- bibtexparser

From this modules the most importants are : `markdownextradata` that let's us use custom templates for our site. Ans `pymdown-extensions` that are used all along the deployment of the site. 

**configure and deploy mkdocs**

To configure the website there is actually just one variable we have to configurate ant that is *site_url* we use the module ansible.builtin.libneinfile to change configure this variable in an automated form. To deploy the website we just need to execute the command:

mkdocs --build and a new folder named *site* will deploy. Having all of the necessary files to index the website with help of apache2. 

***realocation of "site-vitrine"***

Once the website is deployed we needed to ask ourselfs if the current architecture would continue being logical and optimal for our deployment. Since we already a website in the root folder of apache. We would have to create another folder inside the root folder of apache to host our community website. Which would complicate a lot of things in the next stpes of our deployment and is very disorganized architecture. So the other alternative was to deploy our site-vitrine website in a separate folder inside apache's root. That way we would have an architecture like this :

<pre><code class="language-text hljs plaintext">                         [ / ]
                           |
                 _____________________
                |                     |
         [ site-vitrine ]       [ community]
</code></pre> 

We had to change the automation script of both services but it was quite straightforward and dind't pose any major complication.

***configure apache2 redirection***

Finally, once the deploymend had been succesfully automated we needed to configurate make a redirection that in a production environnement would be to the domain site of community and site-vitrine respectively. In our case since we are testing it's deployment. We would like to start a redirection only by changing the port. The port 8100 would redirect to the site-vitrine website and the port 8200 would redirect to the community website. In a fisrt instance I considered using the proxy apache module. However the proxy module would have been usefull if we wanted to be redirected to these ports. But we wanted to redirect from the port to /site-vitrine and /community. So proxy/reverse-proxy was not an option. The solution was quite simple we would just set the `DocumentRoot` of a VirtualHost with the respective port to the location of our websites. then we should : `service apache2 restart` or just start the service when automating it, to enable our configuration.  

***scripts folder in git repo***

It is no longer useful since the deployment is done with Ansible
Although I have looked at the deployment scripts and tried to replicate it in the closest possible way with Ansible.

This is the architecture of the scripts folder

***Proxy Deployment***



Collect Information
Model or Sketch of the proxy
Disassembly of the proxy
Evaluation of the product
Reassemble

## Fontionnalités indispensables à comprendre


Voici une liste de fonctionnalités indispensables à comprendre pour se mettra a jour dans le projet.
<br>

***variables et modules***

Ansible permet d'utililser deux formats pour la creation de ces playbook yaml ou json. Les deux formats sont conceptuellement les mêmes, yaml peut être transformée en json et vice-versa. Ces deux formats permettent de créer des variables, listes, dictionnaires etc... Queslques unes comme par exemple - name: ou task: sont reconnues par Ansible. Le reste de variables qui ne sont pas reconnues par Ansible vont être stockées comme des variables reutilisables dans les tasks executées. On peut enregistrer le message de retour d'une task avec `register:` et on les utilise avec `{{var}}` cette nomenclature est unique a yaml et non pas a Ansible. On peut identifier la même syntaxe dans une pipeline Gitlab par exemple.

Les modules sont en échange uniques à Ansible, ils existent des modules incorporées dans le "core" de l'application comme par exemple le module shell, debug, ou apt. Mais, ils existent plus de modules faites par la communautés qui permettent de se connecter avec un routeur par exemple avec le module Cisco.Ios [³]. Entre eux il existe le module community.docker [⁴], qui est utile pour le déploiement de ce service. 


***module community.docker.containers***

Le module community.docker.containers [²]  faisant partie du module community.docker [⁴] est un plugin d'inventaire dynamique qui permet une connexion automatique avec les containers docker qu'on souhaite. Il à été extrêmement utile pour la première étape du déploiement de ce service.

Syntaxe: 

`plugin: community.docker.docker_containers`

`docker_host: (par default) unix://var/run/docker.sock`


***python interpreter***

Ansible utilise python pour envoyer executer et interpreter les playbooks. Par default Ansible va chercher plusieurs interpreteurs et utiliser la version la plus recente [⁵]. Néanmois, il y a des fois qu'il ne reussi pas à trouver le correct pour X raison. Dans ce cas il faut spécifier l'interpreter: 

`ansible-playbook -i your-inventory.yml your-playbook.yml -e "ansible_python_interpreter=/usr/bin/python3"`

une autre variable utile est `discovered_interpreter_python`

## Important Concepts

### Bash

#### What is the difference between <<<, << and < < in Bash

**Here Document**

« `<<` is known as `here-document` structure. You let the program know what will be the ending text, and whenever that delimiter is seen, the program will read all the stuff you've given to the program as input and perform a task upon it. » [⁶]

**Here String** 

«`<<<` is known as `here string`. you give a pre-made string of text to a program. For example, with such program as bc we can do bc <<< 5*4 to just get output for that specific case, no need to run bc interactively. Think of it as the equivalent of echo '5*4' | bc. » [⁶] 

**Process Substitution**

« As tldp.org explains,

> Process substitution feeds the output of a process (or processes) into the stdin of another process. » [⁶]

« So in effect this is similar to piping stdout of one command to the other , e.g. echo foobar barfoo | wc . But notice: in the bash manpage you will see that it is denoted as <(list). So basically you can redirect output of multiple (!) commands.

Note: technically when you say < < you aren't referring to one thing, but two redirection with single < and process redirection of output from <( . . .).

Now what happens if we do just process substitution?

```
$ echo <(echo bar)
/dev/fd/63
```
As you can see, the shell creates temporary file descriptor /dev/fd/63 where the output goes (which according to Gilles's answer, is an anonymous pipe). That means < redirects that file descriptor as input into a command.

So very simple example would be to make process substitution of output from two echo commands into wc:

```
$ wc < <(echo bar;echo foo)
      2       2       8
```

So here we make shell create a file descriptor for all the output that happens in the parenthesis and redirect that as input to wc .As expected, wc receives that stream from two echo commands, which by itself would output two lines, each having a word, and appropriately we have 2 words, 2 lines, and 6 characters plus two newlines counted. » [⁶]


«  [...] So if we can do grouping with piping, why do we need process substitution? Because sometimes we cannot use piping. Consider the example below - comparing outputs of two commands with diff (which needs two files, and in this case we are giving it two file descriptors)

```
diff <(ls /bin) <(ls /usr/bin)
```

» [⁶]

#### Emulating a try catch clause in Bash

**SubShells in bash**

A subshell is **child process** that runs a separete instance of the shell. When you encole commands inside a '()' you are creating one. 

When inside a subshell there a 3 mecanincs to keep in mind: 

- The subshells variables won't be remebered by it's parent
- The exit status of the subshell is can be different from the parent? And it is the exit status of the last command executed within.
- when doing directory changes (using cd) the parent directory won't be changed. 

**Useful Command operators**

The && and || are very useful command operators that could help us handle errors. While the && operator executes a second command only if the first command has executed. The || operator executes the command only of the precedent command has failed. 

Let's take as an example this command: 
```

echo "something" && false || echo "Command Failed ! " 

```
<code>
something

Command Failed !
</code>

The fist command will execute correctly, which will make the second command execute. Sending an error exit code, this causes the || operator to execute the third command.  

**functions**
You can create a function in bash using this syntax:

```
your_function {
       echo "doSomething"

       echo "The first parameter is $1"
}

# To call you use
your_function "first parameter"

```

**emulate try catch clause simple way**

There are many ways to emulate this clause, it is even possible to use clone a git repo that made it possible to use a perfect try catch clause that can even be nested [⁷].

But for our use case we wouldn't usally need something so complex. Using a Subshell ,the || operator and a function should be enough. 

```
catch {
       echo "There was an error on line $1"
       exit 1
}

(
       echo "Your command"
       false
) || catch $LINENO
```

This should emulate a try catch clause that fails. 

See [Is there a TRY CATCH command in Bash](https://stackoverflow.com/questions/22009364/is-there-a-try-catch-command-in-bash) for more complex solutions

**debugging and error handling**

To stop the script if a command fails

`set -e`

To output an error when a command inside a pipe fails
`set -o pipefail`

#### Awk

**how it works**

Awk is a bash command that allows you to treat text, in it you can program whatever you would like in order to treat text. It has it's own language and executable that can even be turned into a script if needed. 

**awk clauses**

In awk you have a **lot** of different variables that mean different thing and with whom you can interact. I will present you the ones that I used and I know of to create my initial deployment scripts. 

<ins>NF</ins>

NF stands for number of fields, it is used to identify how many caracters a line has, commonly used to find which lines are empty and which ones arent. There are in theory two types of NF clauses. Tha variable NF. And `NF { action }` this clause let's you execute action inside, if you print a line, you would be printing all the non empty lines inside the treated text. 

<ins> END { ... }</ins>

This clause is a special block that let's you execute an action once all the lines of the treated text are read. 

<ins>BEGIN { ... } </ins>

This on the contrary of END, it axecutes all actions before any inputs records are processed. At the very start.

<ins> { ... } </ins>

Code inside this block will execute in each read line.

<ins> /String/ { ... } </ins>
an action executed inside this clause will execute once the said string is found and only there

<ins> ~ </ins>
This is the regex clause, you can include variables in your search inside a regex clause. Ex:

` awk -v var=Sometext '$0 ~ "Text: " var { print } '  `

this would print a line if the string "Text: Sometext" is found

If you want your line to be **exactly** var you can something like `$0 == var`

<ins>Loops</ins>
You have multiple loops in awk, you the for and for each loop, the while loop, the do-while loop. You can use them like you would in any other programming language, you can also use statements such as break, continue, and exit. 

<ins> Conditionals </ins>

You can use an if clause in awk, and the conditionnal next, that would effectively iterate to the next line, very usefull !. 

<details>

<summary>Information utile configuration apache</summary>

| Commande | Description |
| --- | --- |
| `Include` | Ajoute un fichier de configuration |
| `<Directory>` | configuration pour un repertoire particuler |
| `DocumenRoot` | specifie la racine du siteweb |
| `ErrorLog` | specifie la position des ErrorLogs |
| `Define` | Definie une variable d'environnement |
| `VirtualHost` | Contient des directives qui ne s'appliquent qu'à un nom d'hôte spécifique ou à une adresse IP |
</details>



#### A faire 

- [x] Connexion SSH
- [x] Inventaires Dynamiques Docker
- [x] Fichiers config
- [x] Déploiement Hugo
- [x] Rédirection avec apache des deux sites-web community et site-vitrine
- [x] Déploiement dloadsproxy
- [ ] Changement du proxy a python3 
- [ ] :tada:


[¹]: doc.interne.OpenFLUID
[²]: https://docs.ansible.com/ansible/latest/collections/community/docker/docker_containers_inventory.html#ansible-collections-community-docker-docker-containers-inventory
[³]: https://docs.ansible.com/ansible/latest/collections/cisco/ios/index.html
[⁴]: https://docs.ansible.com/ansible/latest/collections/community/docker/index.html
[⁵]: https://docs.ansible.com/ansible/latest/reference_appendices/interpreter_discovery.html
[⁶]: https://askubuntu.com/questions/678915/whats-the-difference-between-and-in-bash
[⁷]: https://github.com/niieani/bash-oo-framework