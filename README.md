Guide Complet d'Administration : Déploiement d'OpenLDAP, phpLDAPadmin et NFS
Partie 1 : Serveur OpenLDAP (Fedora)
Phase 1 : Préparation de l'environnement 

Création de la Machine Virtuelle : Allouez 4 Go de RAM et 30 Go de disque. Configurez la carte réseau avec le choix "custom (vmnet8'NAT')".

Mise à jour du système :

Bash
sudo dnf update -y

Connexion SSH : Vérifiez votre IP et le service SSH.

Bash
ip a
sudo systemctl enable sshd
sudo systemctl status sshd
Note pour Windows (PowerShell) : Connectez-vous via ssh ton_utilisateur@adresse_IP_de_la_VM (exemple : ssh moha@192.168.85.39). Avant de faire cela depuis Windows, pensez à désactiver le pare-feu (firewall.cpl) et à tester le ping (ping 192.168.85.139).

Phase 2 : Préparation de l'identité du serveur et Installation 


Définir le nom d'hôte (Hostname) : 

Bash
sudo hostnamectl set-hostname ldap.cmc.agadir
hostname -f

Configurer la résolution locale (/etc/hosts) : 

Bash
echo "192.168.85.144 ldap.cmc.agadir ldap" | sudo tee -a /etc/hosts

(Note : Utilisation de sudo tee -a pour écrire dans le fichier).


Installation des paquets OpenLDAP : 

Bash
sudo dnf install openldap-servers openldap-clients -y

Démarrage du service : 

Bash
sudo systemctl enable --now slapd
systemctl status slapd
Phase 3 : Configuration du mot de passe de l'administrateur global 


Générer le mot de passe crypté : 

Bash
slappasswd

Créer le fichier de configuration : 

Bash
nano rootpw.ldif
Insérez le contenu suivant (remplacez le hash d'exemple par le vôtre) : 

LDIF
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}COLLE_TON_HASH_ICI

# Exemple de rendu final :
# dn: olcDatabase={0}config,cn=config
# changetype: modify
# add: olcRootPW
# olcRootPW: {SSHA}Vmjk0audIlbZLCC8s93ZEumkhg2Xd4MB

Injecter la configuration : 

Bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f rootpw.ldif
Phase 4 : Importation des schémas de base 

Importez les schémas requis pour structurer l'annuaire :

Bash
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif

(Source des commandes).

Phase 5 : Configuration de ton domaine 


Créer le fichier de domaine : 

Bash
nano db.ldif
Copiez-collez ce contenu : 

LDIF
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=cmc,dc=agadir
-
replace: olcRootDN
olcRootDN: cn=Manager,dc=cmc,dc=agadir
-
add: olcRootPW
olcRootPW: {SSHA}Vmjk0audIlbZLCC8s93ZEumkhg2Xd4MB

Injecter la configuration dans ton serveur : 

Bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f db.ldif
Phase 6 : Création de la structure de base de l'annuaire 


Créer la structure : 

Bash
nano base.ldif
Ajoutez ce contenu : 

LDIF
dn: dc=cmc,dc=agadir
objectClass: top
objectClass: dcObject
objectClass: organization
o: OFPPT CMC Agadir
dc: cmc

dn: cn=Manager,dc=cmc,dc=agadir
objectClass: organizationalRole
cn: Manager
description: Administrateur de l'annuaire

dn: ou=Utilisateurs,dc=cmc,dc=agadir
objectClass: organizationalUnit
ou: Utilisateurs

dn: ou=Groupes,dc=cmc,dc=agadir
objectClass: organizationalUnit
ou: Groupes

Insérer dans l'annuaire : 

Bash
ldapadd -x -D "cn=Manager,dc=cmc,dc=agadir" -W -f base.ldif
Phase 7 : Ajout d'un groupe et d'un utilisateur 


Créer le fichier des utilisateurs : 

Bash
nano utilisateur.ldif
(Note : Assurez-vous d'utiliser utilisateurs.ldif pour l'injection si vous le nommez ainsi). Contenu à copier : 

LDIF
# 1. Création du groupe de la classe
dn: cn=IDOSR-2026,ou=Groupes,dc=cmc,dc=agadir
objectClass: posixGroup
cn: IDOSR-2026
gidNumber: 2000

# 2. Création de l'utilisateur Mohamed
dn: uid=m.naittaouel,ou=Utilisateurs,dc=cmc,dc=agadir
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: m.naittaouel
sn: Naittaouel
givenName: Mohamed
cn: Mohamed Naittaouel
uidNumber: 2000
gidNumber: 2000
userPassword: {SSHA}c2lcJG2wttgFNC6eNrUSZzHK299XPUCb
loginShell: /bin/bash
homeDirectory: /home/m.naittaouel

# 3. Création de l'utilisateur Youness
dn: uid=y.bendaoud,ou=Utilisateurs,dc=cmc,dc=agadir
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: y.bendaoud
sn: Ben Daoud
givenName: Youness
cn: Youness Ben Daoud
uidNumber: 2001
gidNumber: 2000
userPassword: {SSHA}c2lcJG2wttgFNC6eNrUSZzHK299XPUCb
loginShell: /bin/bash
homeDirectory: /home/y.bendaoud

# 4. Création de l'utilisateur Youssef
dn: uid=y.barbach,ou=Utilisateurs,dc=cmc,dc=agadir
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: y.barbach
sn: Barbach
givenName: Youssef
cn: Youssef Barbach
uidNumber: 2002
gidNumber: 2000
userPassword: {SSHA}c2lcJG2wttgFNC6eNrUSZzHK299XPUCb
loginShell: /bin/bash
homeDirectory: /home/y.barbach

Injecter le fichier : 

Bash
ldapadd -x -D "cn=Manager,dc=cmc,dc=agadir" -W -f utilisateurs.ldif

Le test ultime (La recherche LDAP) : 

Bash
ldapsearch -x -b "dc=cmc,dc=agadir" "(objectClass=inetOrgPerson)"
Phase 8 : Installation de l'interface web (phpLDAPadmin) 


Installer le serveur web et PHP : 

Bash
sudo dnf install httpd php php-ldap php-xml -y

Démarrer le serveur web : 

Bash
sudo systemctl enable --now httpd

Installer phpLDAPadmin : 

Bash
sudo dnf install epel-release -y
sudo dnf install phpldapadmin -y

Autoriser le trafic web dans le pare-feu : 

Bash
sudo systemctl stop firewalld
Phase 9 : Configuration de phpLDAPadmin 


Modifier le fichier de configuration principal : 

Bash
sudo nano /etc/phpldapadmin/config.php
Ou injectez directement la configuration avec tee : 

PHP
sudo tee /etc/phpldapadmin/config.php > /dev/null << 'EOF'
<?php
$servers = new Datastore();
$servers->newServer('ldap_pla');
$servers->setValue('server','name','Annuaire CMC Agadir');
$servers->setValue('server','host','127.0.0.1');
$servers->setValue('server','port',389);
$servers->setValue('server','base',array('dc=cmc,dc=agadir'));
$servers->setValue('login','auth_type','cookie');
$servers->setValue('login','bind_id','cn=Manager,dc=cmc,dc=agadir');
$servers->setValue('server','tls',false);
?>
EOF

(Source de la configuration).


Corriger la librairie de connexion : 

Bash
sudo nano +208 /usr/share/phpldapadmin/lib/ds_ldap.php
Changez le bloc de code par : 

PHP
if ($this->getValue('server','port'))
    $resource = @ldap_connect("ldap://" . $this->getValue('server','host') . ":" . $this->getValue('server','port'));
else
    $resource = @ldap_connect("ldap://" . $this->getValue('server','host'));

Autoriser l'accès réseau à l'interface web : Ton objectif est de trouver la ligne Require local (dans la config apache) et de la remplacer par Require all granted. 


Autoriser la connexion via SELinux et redémarrer : 

Bash
sudo setsebool -P httpd_can_connect_ldap on
sudo systemctl restart httpd
Étape 1 : Ta première connexion officielle 

Va sur ton navigateur à l'adresse : http://192.168.85.144/phpldapadmin.

Clique sur Connexion.

Saisis ton identifiant : cn=Manager,dc=cmc,dc=agadir.

Saisis ton mot de passe administrateur et valide.

Une fois connecté, tu vas voir ton arborescence à gauche, avec dc=projet et tes dossiers ou=Groupes et ou=Utilisateurs.

Gestion via l'interface

Le Groupe : 

Va dans ou=Groupes > Créer une nouvelle entrée ici > Generic: Posix Group.


Group Name : Reseaux.


GID Number : Efface le numéro par défaut et tape 2000.

Valide la création.


L'Utilisateur : 

Va dans ou=Utilisateurs > Créer une nouvelle entrée ici > Generic: User Account.


First / Last Name : Mohamed Naittaouel.


User ID (uid) : m.naittaouel.


Password : Choisis un mot de passe simple, idéalement avec les chiffres du haut du clavier pour éviter tout piège lié aux claviers QWERTY/AZERTY lors de la première connexion.


UID Number : Force-le à 2000.


GID Number : Sélectionne ton groupe Reseaux (qui a le numéro 2000).


Home directory : /home/m.naittaouel.

Valide la création.

Partie 2 : Préparation de la nouvelle VM Mint (Client) ou Ubuntu 


À exécuter sur la machine cliente. 

Phase 4 : Installation du client d'authentification (SSSD) 

Bash
sudo apt update
sudo apt install sssd libnss-sss libpam-sss ldap-utils -y
Phase 5 : Configuration de SSSD 

Bash
sudo nano /etc/sssd/sssd.conf
Ajoutez cette configuration : 

Ini, TOML
[sssd]
config_file_version = 2
services = nss, pam
domains = LDAP

[domain/LDAP]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://192.168.85.144
ldap_search_base = dc=cmc,dc=agadir

# Désactivation de la vérification TLS pour ce test local
ldap_tls_reqcert = never
ldap_auth_disable_tls_never_use_in_production = true

# Options de confort
cache_credentials = true
enumerate = true
Phase 6 : Sécurisation et activation 

Bash
sudo chmod 600 /etc/sssd/sssd.conf
sudo systemctl restart sssd
sudo systemctl enable sssd
Phase 7 : Création automatique du dossier personnel 

Bash
sudo pam-auth-update --enable mkhomedir
L'Heure de Vérité ! 

Bash
id m.naittaouel

Attention sur le serveur Fedora : Vérifiez le pare-feu. 

Bash
sudo firewall-cmd --list-all

(Regarde la ligne services:. Tu dois absolument y voir ldap à côté de http et ssh). Si non, exécutez : 

Bash
sudo firewall-cmd --add-service=ldap

Le test de connexion locale : 

Bash
sudo systemctl restart sssd
su - m.naittaouel

Déléguer les droits d'administration (sudo) : 

Bash
sudo visudo
Ajouter la ligne : 

Plaintext
%Reseaux ALL=(ALL:ALL) ALL
Partie 3 : Partage Réseau avec NFS
Étape 1 : Préparation du serveur NFS (Sur ton Fedora) 

Bash
sudo dnf install nfs-utils -y
sudo systemctl enable --now nfs-server
Étape 2 : Création du dossier centralisé 

Bash
sudo mkdir -p /export/home
sudo chmod 777 /export/home
Connecter le disque dur du serveur (Montage NFS) 

Nous voulons que Mint "accroche" le dossier du serveur pour faire croire à l’utilisateur qu’il est dans la machine.

Sur le client : 

Bash
sudo mkdir -p /home/reseau
sudo nano /etc/fstab
Descendez tout en bas du fichier et ajoutez cette ligne : 

Plaintext
192.168.85.139:/export/home /home/reseau nfs defaults 0 0
Sauvegardez (Ctrl+O, Entrée) et Quittez (Ctrl+X).


On valide l’accrochage : 

Bash
sudo systemctl daemon-reload
sudo mount -a

Tu peux maintenant accéder à l'utilisateur en mode graphique sur la machine cliente. 

Le partage (Configuration avancée) 


Sur le serveur Fedora : 

La création du dossier : 

Bash
sudo mkdir -p /export/atmani_share
sudo chgrp 2000 /export/atmani_share
sudo chmod 2777 /export/atmani_share
Déclarer dans le fichier NFS : 

Bash
sudo nano /etc/exports
Ajouter ce texte : 

Plaintext
/export/atmani_share 192.168.85.0/24(rw,sync,no_root_squash)
Valider et redémarrer le service : 

Bash
sudo exportfs -arv
sudo systemctl restart nfs-server

Dans le côté client : 

Nettoie l'ancien réseau : 

Bash
sudo umount /home/reseau
sudo mkdir -p /home/reseau/atmani_share
Configurer le montage automatique : 

Bash
sudo nano /etc/fstab
Ajouter cette ligne et effacer toute ancienne : 

Plaintext
192.168.85.144:/export/atmani_share /home/reseau/atmani_share nfs defaults 0 0
Valider : 

Bash
sudo systemctl daemon-reload
sudo mount -a
La Migration 

On va créer l'utilisateur sur le client puis le migrer. 


Création locale : 

Bash
sudo useradd a.ouahnayn
sudo passwd a.ouahnayn
grep hassan /etc/passwd # (Prendre le numéro 1001)

Création du fichier LDIF : 

Bash
nano asmae.ldif
Contenu à injecter : 

LDIF
dn: uid=a.ouahnayn,ou=utilisateurs,dc=cmc,dc=agadir
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: a.ouahnayn
sn: ouahnayn
givenName: asmae
cn: asmae ouahnayn
displayName: asmae ouahnayn
uidNumber: 1001
gidNumber: 2000
userPassword: Admin123
loginShell: /bin/bash
homeDirectory: /export/home/a.ouahnayn

Installation et Injection LDAP : Installez les outils de LDAP sur Linux : 

Bash
sudo apt install ldap-utils -y
Lancer la commande d'injection ou bien de migration : 

Bash
ldapadd -x -H ldap://192.168.85.144 -D "cn=Manager,dc=cmc,dc=agadir" -W -f asmae.ldif
