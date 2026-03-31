# Procédure d'installation et de configuration OpenLDAP

---

## Phase 1 : Préparation de l'environnement

### Création de la Machine Virtuelle
4 Go de RAM et 30 Go de disque, avec un adaptateur réseau en mode **Custom (VMnet8 - NAT)**.

```bash
sudo dnf update -y
Connexion SSH
ip a
sudo systemctl enable sshd
sudo systemctl status sshd

Depuis PowerShell Windows :

ssh ton_utilisateur@adresse_IP_de_la_VM
# Exemple
ssh moha@192.168.85.39

⚠️ Attention : avant cela, désactiver le firewall Windows :

firewall.cpl
ping 192.168.85.139
Phase 2 : Préparation de l'identité du serveur et Installation
Définir le nom d'hôte (Hostname)
sudo hostnamectl set-hostname ldap.cmc.agadir
hostname -f
Configurer la résolution locale (/etc/hosts)
echo "192.168.85.144 ldap.cmc.agadir ldap" | sudo tee -a /etc/hosts
Installation des paquets OpenLDAP
sudo dnf install openldap-servers openldap-clients -y
Démarrage du service
sudo systemctl enable --now slapd
systemctl status slapd
Phase 3 : Configuration du mot de passe de l'administrateur global
Générer le mot de passe crypté
slappasswd
nano rootpw.ldif
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}COLLE_TON_HASH_ICI
Exemple
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}Vmjk0audIlbZLCC8s93ZEumkhg2Xd4MB
Injection de la configuration
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f rootpw.ldif
Phase 4 : Importation des schémas de base
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
Phase 5 : Configuration de ton domaine
nano db.ldif

Copier-coller :

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

Injection :

sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f db.ldif
Phase 6 : Création de la structure de base de l'annuaire
nano base.ldif
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
ldapadd -x -D "cn=Manager,dc=cmc,dc=agadir" -W -f base.ldif
Phase 7 : Ajout d'un groupe et d'un utilisateur
nano utilisateur.ldif
# 1. Création du groupe
dn: cn=IDOSR-2026,ou=Groupes,dc=cmc,dc=agadir
objectClass: posixGroup
cn: IDOSR-2026
gidNumber: 2000

# 2. Utilisateur Mohamed
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

# 3. Utilisateur Youness
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

# 4. Utilisateur Youssef
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
ldapadd -x -D "cn=Manager,dc=cmc,dc=agadir" -W -f utilisateurs.ldif
Test ultime : recherche LDAP
ldapsearch -x -b "dc=cmc,dc=agadir" "(objectClass=inetOrgPerson)"
Phase 8 : Installation de l'interface web (phpLDAPadmin)
sudo dnf install httpd php php-ldap php-xml -y
sudo systemctl enable --now httpd
sudo dnf install epel-release -y
sudo dnf install phpldapadmin -y
sudo systemctl stop firewalld
Phase 9 : Configuration de phpLDAPadmin
Modifier le fichier
sudo nano /etc/phpldapadmin/config.php
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
Modification du fichier interne
sudo nano +208 /usr/share/phpldapadmin/lib/ds_ldap.php

Remplacer par :

if ($this->getValue('server','port'))
    $resource = @ldap_connect("ldap://" . $this->getValue('server','host') . ":" . $this->getValue('server','port'));
else
    $resource = @ldap_connect("ldap://" . $this->getValue('server','host'));
Autoriser l'accès réseau

Remplacer :

Require local

par :

Require all granted
SELinux
sudo setsebool -P httpd_can_connect_ldap on
sudo systemctl restart httpd
Étape 1 : Première connexion

Naviguer vers :

http://192.168.85.144/phpldapadmin
Cliquer sur Connexion
Identifiant : cn=Manager,dc=cmc,dc=agadir
Mot de passe administrateur

Vous verrez l’arborescence avec :

dc=cmc
ou=Groupes
ou=Utilisateurs
Création via interface web
Groupe
ou=Groupes → Créer une nouvelle entrée → Generic: Posix Group
Group Name : Reseaux
GID Number : 2000
Utilisateur
ou=Utilisateurs → Generic: User Account
Nom : Mohamed Naittaouel
UID : m.naittaouel
Password : simple
UID Number : 2000
GID : 2000
Home : /home/m.naittaouel
Client (Mint / Ubuntu)
Phase 4 : Installation SSSD
sudo apt update
sudo apt install sssd libnss-sss libpam-sss ldap-utils -y
Phase 5 : Configuration
sudo nano /etc/sssd/sssd.conf
[sssd]
config_file_version = 2
services = nss, pam
domains = LDAP

[domain/LDAP]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://192.168.85.144
ldap_search_base = dc=cmc,dc=agadir

ldap_tls_reqcert = never
ldap_auth_disable_tls_never_use_in_production = true

cache_credentials = true
enumerate = true
Phase 6 : Activation
sudo chmod 600 /etc/sssd/sssd.conf
sudo systemctl restart sssd
sudo systemctl enable sssd
Phase 7 : Dossier personnel
sudo pam-auth-update --enable mkhomedir
Test
id m.naittaouel
sudo systemctl restart sssd
su - m.naittaouel
Configuration NFS
Serveur
sudo dnf install nfs-utils -y
sudo systemctl enable --now nfs-server
sudo mkdir -p /export/home
sudo chmod 777 /export/home
Partage
sudo mkdir -p /export/atmani_share
sudo chgrp 2000 /export/atmani_share
sudo chmod 2777 /export/atmani_share
sudo nano /etc/exports
/export/atmani_share 192.168.85.0/24(rw,sync,no_root_squash)
sudo exportfs -arv
sudo systemctl restart nfs-server
Client NFS
sudo umount /home/reseau
sudo mkdir -p /home/reseau/atmani_share
sudo nano /etc/fstab
192.168.85.144:/export/atmani_share /home/reseau/atmani_share nfs defaults 0 0
sudo systemctl daemon-reload
sudo mount -a
Migration utilisateur
sudo useradd a.ouahnayn
sudo passwd a.ouahnayn
grep hassan /etc/passwd
nano asmae.ldif
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
sudo apt install ldap-utils -y
ldapadd -x -H ldap://192.168.85.144 -D "cn=Manager,dc=cmc,dc=agadir" -W -f asmae.ldif
