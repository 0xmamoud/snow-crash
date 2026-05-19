# Methodologie CTF

Ce fichier est un memo personnel pour apprendre les CTF sans deviner au hasard.
Le but n'est pas de memoriser des solutions, mais d'apprendre quoi observer, quoi trouver suspect, et comment tester proprement.

## Idee principale

Dans un CTF, il y a souvent une anomalie volontaire.
Ton travail consiste a enumerer le systeme, trouver ce qui sort de l'ordinaire, formuler une hypothese, tester, puis documenter.

Boucle de travail :

```text
1. Observer : collecter les faits.
2. Comparer : qu'est-ce qui est bizarre par rapport a un Linux normal ?
3. Hypothese : quelle famille de faille cela peut etre ?
4. Tester : faire le plus petit test possible.
5. Expliquer : noter pourquoi la solution fonctionne.
```

## Commandes de depart sur un niveau Linux

```bash
whoami
id
pwd
ls -la
env
cat /etc/passwd
```

Pourquoi :

- `whoami` dit quel utilisateur tu es.
- `id` affiche UID, GID et groupes.
- `pwd` confirme le dossier courant.
- `ls -la` montre les fichiers caches et les permissions.
- `env` montre les variables d'environnement, notamment `PATH`.
- `/etc/passwd` liste les utilisateurs locaux et peut reveler une mauvaise configuration.

## Commandes `find` importantes

Fichiers appartenant a l'utilisateur cible :

```bash
find / -user flagXX 2>/dev/null
```

Recherche par nom :

```bash
find / -name "*flag*" 2>/dev/null
find / -iname "*password*" 2>/dev/null
```

Fichiers lisibles par toi :

```bash
find / -readable -type f 2>/dev/null
```

Dossiers dans lesquels tu peux ecrire :

```bash
find / -writable -type d 2>/dev/null
```

Binaires SUID :

```bash
find / -perm -4000 -type f 2>/dev/null
```

Binaires SGID :

```bash
find / -perm -2000 -type f 2>/dev/null
```

Fichiers modifiables par tout le monde :

```bash
find / -perm -o+w -type f 2>/dev/null
```

Fichiers modifies recemment :

```bash
find / -mtime -1 2>/dev/null
```

```bash
ls -la /path/to/file
file /path/to/file
cat /path/to/file
strings /path/to/file
```

Si c'est executable :

```bash
/path/to/file
strings /path/to/file
ltrace /path/to/file
strace /path/to/file
```

Questions a se poser :

- Qui possede le fichier ?
- Est-ce que je peux le lire ?
- Est-ce que je peux l'executer ?
- Est-il SUID ou SGID ?
- Appelle-t-il des commandes externes ?
- Lit-il un fichier dans `/tmp` ou un dossier writable ?
- Utilise-t-il directement une entree utilisateur ?
- Contient-il un mot de passe, hash, token ou texte encode ?

## Ce qu'il faut apprendre a reperer

### Permissions suspectes

Exemples :

```text
-rwsr-xr-x owner flagXX ... program
drwxrwxrwx ... directory
-rw-rw-rw- ... file
```

Pourquoi c'est suspect :

- SUID/SGID peut permettre de franchir une frontiere de privileges.
- Un fichier writable par tout le monde peut etre modifie par n'importe qui.
- Un dossier writable par tout le monde peut servir pour des attaques via fichiers temporaires ou `PATH`.

### Hash expose

Ligne suspecte dans `/etc/passwd` :

```text
flag01:42hDRfypTqqnw:3001:3001::/home/flag/flag01:/bin/bash
```

Sur un Linux moderne, on attend plutot :

```text
flag01:x:3001:3001::/home/flag/flag01:/bin/bash
```

Si le deuxieme champ n'est pas `x`, il peut s'agir d'un hash de mot de passe crackable hors ligne avec John the Ripper.

### Texte encode ou obfusque

Exemples :

```text
cdiiddwpgswtgt       possible Cesar/ROT
aGVsbG8=             possible Base64
68656c6c6f           possible hexadecimal
5d41402abc4b2a76...  possible hash
```

Tests courants :

- Cesar/ROT
- Base64
- Hexadecimal
- Codes ASCII
- Texte inverse
- XOR avec cle simple
- Identification de hash

### Execution de commandes dangereuse

Patterns suspects dans scripts ou binaires :

```text
system("ls")
system("tar ...")
os.system(user_input)
`command $input`
```

Pourquoi c'est suspect :

- Si l'entree utilisateur atteint un shell, une injection de commande peut etre possible.
- Si une commande est appelee sans chemin absolu, un detournement de `PATH` peut etre possible.

### PATH hijacking

Si un programme privilegie execute :

```text
system("getflag")
```

au lieu de :

```text
system("/bin/getflag")
```

le shell peut chercher `getflag` dans `PATH` et lancer ta fausse commande avant la vraie.

Choses a inspecter :

```bash
echo $PATH
strings ./program
ltrace ./program
```

### Fichiers temporaires

Chemins suspects :

```text
/tmp/file
/var/tmp/file
/tmp/script.sh
```

Pourquoi c'est suspect :

- `/tmp` est souvent writable par tout le monde.
- Un processus privilegie qui utilise un nom de fichier previsible peut etre abuse.
- Les symlinks peuvent devenir dangereux si un programme privilegie suit un chemin controle par l'utilisateur.

### Symlinks

Commande :

```bash
ls -la
```

Exemple suspect :

```text
file -> /some/protected/path
```

Pourquoi c'est suspect :

- Un programme peut suivre le lien et lire/ecrire un fichier qu'il ne devrait pas.
- Les symlinks sont importants quand un programme lit des fichiers dans un dossier writable.

### Parsing de fichiers

Si un programme lit un fichier que tu controles, demande-toi :

- Fait-il confiance au nom du fichier ?
- Fait-il confiance au contenu ?
- Parse-t-il ligne par ligne ?
- Lance-t-il des commandes avec ce contenu ?
- Refuse-t-il les symlinks ou les suit-il ?

## Failles qu'on cherche souvent en CTF

### Linux privilege escalation

Chercher :

- Binaires SUID/SGID
- Scripts writable executes par un utilisateur privilegie
- Mauvaise utilisation de `PATH`
- Cron jobs
- Credentials exposes
- Permissions trop larges
- Hashs dans des fichiers lisibles
- Services avec trop de privileges

### Reverse engineering

Chercher :

- Strings dans les binaires
- Mots de passe hardcodes
- Comparaisons avec l'entree utilisateur
- Checks obfusques
- Appels a `strcmp`, `strncmp`, `system`, `open`, `read`

Commandes :

```bash
file ./binary
strings ./binary
ltrace ./binary
strace ./binary
```

### Crypto basique

Chercher :

- Cesar/ROT
- Vigenere
- XOR
- Base64 ou hex pris pour du chiffrement
- Cle reutilisee
- Aleatoire previsible
- Hash faible

Regle :

```text
Encodage = reversible sans secret.
Chiffrement = reversible avec une cle.
Hash = normalement non reversible ; on crack en devinant l'entree.
```

### Forensics

Chercher :

- Fichiers caches
- Metadata
- Strings dans images/documents
- Archives cachees dans des fichiers
- Donnees supprimees
- Steganographie

Commandes :

```bash
file suspicious_file
strings suspicious_file
exiftool suspicious_file
binwalk suspicious_file
```

### Web

Categories frequentes :

- SQL injection
- OS command injection
- Path traversal
- File upload vulnerabilities
- Authentication bypass
- Access control bugs
- Cross-site scripting
- Server-side request forgery
- Insecure deserialization
- Business logic bugs

Approche :

```text
1. Cartographier les endpoints et les parametres.
2. Modifier une entree a la fois.
3. Observer les messages d'erreur et changements de comportement.
4. Tester si une entree atteint SQL, shell, filesystem, templates ou redirects.
```

## Checklist 80% pour un niveau Linux

```text
1. Identifier l'utilisateur courant : whoami, id.
2. Inspecter le home : ls -la.
3. Lire /etc/passwd et chercher un champ password anormal.
4. Chercher les fichiers du user cible : find / -user flagXX 2>/dev/null.
5. Chercher SUID/SGID : find / -perm -4000 -type f 2>/dev/null.
6. Chercher writable : find / -writable ...
7. Inspecter les fichiers avec ls, file, cat, strings.
8. Si executable : lancer normalement, puis strings/ltrace/strace.
9. Chercher encodage, hash, injection, PATH hijacking, symlink.
10. Documenter chaque commande et pourquoi elle compte.
```

## Comment ne pas te spoiler

Demande des indices progressifs :

```text
Hint level 1 : donne-moi seulement des commandes d'enumeration.
Hint level 2 : dis-moi seulement la famille de faille.
Hint level 3 : confirme ou corrige mon hypothese.
Hint level 4 : explique la solution complete.
```

Bonne formulation :

```text
Je suis au levelXX. Voici mes commandes et outputs. Donne-moi seulement un indice niveau 1.
```
