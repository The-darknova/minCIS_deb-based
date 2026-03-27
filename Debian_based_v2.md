**Debian based hardening [v2]**  
**Guide de Durcissement (Hardening) pour Systèmes Debian**  
Ce document est un guide opérationnel destiné aux administrateurs système pour sécuriser les serveurs basés sur la distribution Debian. Il suit le principe du moindre privilège, de la réduction de la surface d'attaque et de la défense en profondeur.  
ATTENTION : L'application de ces paramètres doit toujours être testée dans un environnement de pré-production avant toute mise en production. Certaines restrictions peuvent altérer le fonctionnement de vos services légitimes.  
**Phase 1: Protection des Systèmes de Fichiers**  
L'objectif est d'empêcher le montage de systèmes de fichiers obsolètes ou non sécurisés et de restreindre les permissions sur partitions temporaires ou utilisateurs.  
**1.1 Désactivation des modules de systèmes de fichiers non utilisés**  
Ces systèmes de fichiers (souvent liés à d'anciens supports ou périphériques) augmentent la surface d'attaque du noyau.  
**Action :** Blacklister les modules.  
```bash
sudo bash -c 'cat <<EOF > /etc/modprobe.d/hardening-fs.conf  
 install cramfs /bin/false  
 blacklist cramfs  
 install freevxfs /bin/false  
 blacklist freevxfs  
 install jffs2 /bin/false  
 blacklist jffs2  
 install hfs /bin/false  
 blacklist hfs  
 install hfsplus /bin/false  
 blacklist hfsplus  
 install squashfs /bin/false  
 blacklist squashfs  
 install udf /bin/false  
 blacklist udf  
 install usb-storage /bin/false  
 blacklist usb-storage  
 install firewire-core /bin/false  
 blacklist firewire-core  
 install afs /bin/false  
 blacklist afs  
 EOF'  
```
**Action :** Décharger les modules s'ils sont actuellement en mémoire.  
```bash
sudo bash -c \  
 'for mod in cramfs freevxfs jffs2 hfs hfsplus udf usb-storage firewire-core afs; do  
     modprobe -r $mod 2>/dev/null && rmmod $mod 2>/dev/null  
 done'  
```
   
**1.2 Sécurisation des partitions**  
🛑Attention: Cette partie est uniquement applicable si la partition existe.  
Il faut seulement ajouter les parametres correspondantes en fin de lignes existante. Ne pas ajouter de ligne dans le fichier fstab si elle est absente.  
Limiter les droits d'exécution (noexec), la création de périphériques (nodev) et les permissions SUID (nosuid) sur les dossiers sensibles.  
**Action :** Éditez le fichier /etc/fstab et ajoutez les options requises :  
- /tmp : nodev, nosuid, noexec  
- /var/tmp : nodev, nosuid, noexec  
- /var : nodev, nosuid, noexec  
- /dev/shm : nodev, nosuid, noexec  
- /home : nodev, nosuid  
- /var/log : nodev, nosuid, noexec  
- /var/log/audit : nodev, nosuid, noexec  
**Action :** Remonter les partitions à chaud.  
```bash
sudo mount -o remount /tmp  
 sudo mount -o remount /var/tmp  
 sudo mount -o remount /dev/shm  
 sudo mount -o remount /home  
 sudo mount -o remount /var/log/audit  
```
**Phase 2: Gestion Sécurisée des Paquets**  
**2.1 S’assurer que les repos utilisés sont correctes**  
```bash
cat /etc/apt/sources.{list,list.d/*}
```
**2.2 Limitation des dépendances faibles**  
Pour éviter d'installer des paquets inutiles (surface d'attaque supplémentaire), désactivez les "Recommends" et "Suggests" d'APT.  
```bash
printf '%s\n%s\n' 'APT::Install-Recommends "0";' 'APT::Install-Suggests "0";' | \  
 sudo tee /etc/apt/apt.conf.d/60-no-weak-dependencies  
```
**2.3 Sécurisation des dépôts APT**  
Restreindre la modification des fichiers de dépôts à l'utilisateur root.  
```bash
sudo chown -R root:root /etc/apt  
 sudo find /etc/apt -type d -exec chmod 755 {} +  
 sudo find /etc/apt -type f -exec chmod 644 {} +  
```
**2.4 Mises à jour du système**  
S'assurer que le système dispose des derniers correctifs de sécurité.  
```bash
sudo apt update && sudo apt full-upgrade -y  
 # --- OU ---  
 sudo dist-upgrade # Si la distribution est obsolète  
 # Vérifier si un redémarrage complet est nécessaire  
 [ -f /var/run/reboot-required ] && cat /var/run/reboot-required && sudo reboot  
```
**Phase 3: Contrôle d'Accès Obligatoire (AppArmor)**  
AppArmor confine les programmes à un ensemble limité de ressources, atténuant ainsi les attaques de type 0-day.  
**3.1 Installation et activation dans le GRUB**  
# Installation  
```bash
 sudo apt update && sudo apt install -y apparmor apparmor-utils  
   
 # Pour activer AppArmor au démarrage, éditez /etc/default/grub et modifiez GRUB_CMDLINE_LINUX :  
 # GRUB_CMDLINE_LINUX="apparmor=1 security=apparmor"  
   
 # Mettre à jour GRUB  
 sudo update-grub  
```
**3.2 Mise en mode "Enforce" et restriction des namespaces**  
# Passer tous les profils en mode restrictif  
```bash
 sudo aa-enforce /etc/apparmor.d/*  
   
 # Restreindre les namespaces non privilégiés  
 echo "kernel.apparmor_restrict_unprivileged_unconfined = 1" | sudo tee -a /etc/sysctl.d/60-apparmor-namespace.conf  
 sudo sysctl --system  
```
*Note : Si une application dysfonctionne, utilisez * *aa-complain* * pour la passer en mode apprentissage, analysez les logs (* *journalctl -e | grep 'apparmor="DENIED"'* *), puis générez un profil avec * *aa-genprof* *.*  
**Phase 4: Sécurisation du Bootloader**  
Empêcher un utilisateur local non privilégié de modifier les paramètres de démarrage (comme le passage en single-user mode). 
```bash
sudo chown root:root /boot/grub/grub.cfg  
 sudo chmod u-x,go-rwx /boot/grub/grub.cfg  
```
**Phase 5: Durcissement du Noyau et des Processus**  
**5.1 Optimisation des paramètres Sysctl**  
Ce fichier configure la randomisation de l'espace d'adressage (ASLR), restreint l'accès aux journaux du noyau, et protège les liens symboliques/physiques.  
```bash
sudo bash -c 'cat <<EOF > /etc/sysctl.d/60-process-hardening.conf  
 kernel.randomize_va_space = 2  
 kernel.yama.ptrace_scope = 1  
 fs.suid_dumpable = 0  
 kernel.dmesg_restrict = 1  
 kernel.kptr_restrict = 2  
 fs.protected_hardlinks = 1  
 fs.protected_symlinks = 1  
 EOF'  
 sudo sysctl --system  
```
**5.2 Restriction des Core Dumps et Outils de Débogage**  
Empêcher le système de vider la RAM sur le disque lors d'un crash (pourrait contenir des mots de passe).  
```bash
echo "* hard core 0" | sudo tee -a /etc/security/limits.d/60-disable-core-dumps.conf  
 # Suppression de prelink et apport  
 sudo bach -c 'prelink -ua; apt purge prelink apport &>/dev/null'  
```
**5.3 Installer auditd**  
```bash
sudo apt update && sudo apt install auditd audispd-plugins  
 sudo systemctl enable auditd  
 # Pour activer auditd, éditez /etc/default/grub et modifiez GRUB_CMDLINE_LINUX :  
 # Exemple: GRUB_CMDLINE_LINUX="apparmor=1 security=apparmor audit=1"  
   
 # Mettre à jour GRUB  
 sudo update-grub  
```
**Phase 6: Bannières et Avertissements Légaux**  
La présence d'un avertissement légal (bannière) est indispensable avant toute authentification pour se prémunir juridiquement.  
**Action :** Appliquez le message suivant aux fichiers /etc/motd, /etc/issue et /etc/issue.net.  
```plaintext
*******************************************************************************  
 AVERTISSEMENT: Accès autorisé uniquement.  
 Ce système est surveillé à des fins de sécurité. Tout accès ou utilisation  
 non autorisé(e) de ce système est interdit(e) et peut entraîner des sanctions  
 pénales ou civiles.  
   
 https://changeme.com
 *******************************************************************************  
```
**Action :** Sécurisez les permissions de ces fichiers.  
```bash
sudo chown root:root /etc/motd /etc/issue /etc/issue.net  
sudo chmod 644 /etc/motd /etc/issue /etc/issue.net  
```
**Phase 7 & 8: Suppression des Services Inutiles**  
Un serveur sécurisé ne doit faire tourner que ce dont il a besoin.  
**7.1 Suppression de l'interface graphique (GDM/X11)**  
```bash
sudo apt purge -y gdm3; sudo apt purge 'xserver-*' 'libx11-.*'  
 sudo apt autoremove -y  
```
**8.1 Nettoyage des services réseaux obsolètes ou non requis**  
L'exécution des commandes de nettoyage supprimera les services listés ci-dessous, qui représentent un risque s'ils ne sont pas explicitement requis :  
- **Partage de fichiers et protocoles hérités :**  
- vsftpd, ftp, tnftp : Serveurs/clients FTP (les identifiants transitent en clair).  
- tftpd-hpa : Serveur TFTP (Trivial FTP, ne possède aucune authentification).  
- nfs-kernel-server : Serveur de partage réseau NFS (souvent mal sécurisé par défaut).  
- samba, smbd : Partage de fichiers Windows (SMB/CIFS).  
- autofs : Service de montage automatique de systèmes de fichiers distants.  
- **Services de découverte et d'impression :**  
- avahi-daemon : Service Zeroconf (mDNS/DNS-SD) utile pour les ordinateurs de bureau, mais dangereux sur un serveur.  
- cups : Système d'impression (Common UNIX Printing System), inutile sur un serveur classique.  
- bluez : Daemon Bluetooth, augmentant la surface d'attaque physique/sans fil.  
- **Annuaire et authentification :**  
- slapd, ldap-utils : Serveur d'annuaire LDAP et ses outils.  
- ypserv : Serveur NIS (Network Information Service), protocole hérité obsolète et non chiffré.  
- **Infrastructure réseau et messagerie :**  
- bind9, named, dnsmasq : Serveurs et relais DNS.  
- isc-dhcp-server : Serveur d'adresses IP DHCP.  
- rpcbind : Mappeur de ports RPC, souvent ciblé pour des attaques DDoS par amplification.  
- dovecot-imapd, dovecot-pop3d : Serveurs de réception d'emails.  
- **Protocoles distants en clair :**  
- rsh-client, talk, telnet, inetutils-telnet : Utilitaires de communication hérités sans chiffrement (à remplacer par SSH).  
- **Serveurs Web / Proxy (si non requis par le rôle du serveur) :**  
- apache2, nginx : Serveurs web.  
- squid : Serveur proxy.  
- **Divers :**  
- snmp : Protocole de supervision réseau (les v1 et v2c transmettent en clair).  
- rsync : Outil de synchronisation, s'il tourne en mode daemon autonome.  
- xinetd : Super-serveur réseau étendu, de nos jours remplacé par systemd.  
Exécutez ce script pour arrêter et supprimer massivement ces services de manière tolérante aux erreurs (ignorera ceux qui ne sont pas installés) :  
```bash
sudo apt-get update -qq && \  
 sudo DEBIAN_FRONTEND=noninteractive apt-get purge -y -qq \  
 vsftpd ftp tnftp tftpd-hpa nfs-kernel-server samba smbd autofs \  
 avahi-daemon cups bluez slapd ldap-utils ypserv bind9 bind9utils \  
 bind9-doc dnsmasq isc-dhcp-server rpcbind dovecot-imapd dovecot-pop3d \  
 rsh-client talk telnet inetutils-telnet apache2 nginx squid snmp \  
 rsync xinetd && \  
 sudo apt-get autoremove -y -qq && \  
 sudo apt-get clean  
```
**8.2 Sécurisation du serveur Mail (MTA)**  
Si votre serveur ne sert pas de relais mail, le MTA (Postfix/Exim) ne doit écouter que sur localhost (127.0.0.1).  
- **Vérification :**sudo ss -plntu | grep -E ":(25|465|587)"  
- **Remédiation (Exemple Postfix) :** Dans /etc/postfix/main.cf, définissez inet_interfaces = loopback-only, puis redémarrez Postfix.  
**8.3 Corriger les permission de crontab**  
```bash
sudo chown root:root /etc/{crontab,cron.hourly,cron.daily,cron.weekly,cron.monthly,cron.d}  
 sudo chmod og-rwx /etc/{crontab,cron.hourly,cron.daily,cron.weekly,cron.monthly,cron.d}  
```
**Phase 9: Sécurisation du Réseau et Pare-feu**  
**9.1 Désactivation des protocoles réseaux exotiques**  
```bash
sudo bash -c 'cat <<EOF > /etc/modprobe.d/hardening-networking.conf  
 install dccp /bin/false  
 blacklist dccp  
 install tipc /bin/false  
 blacklist tipc  
 install rds /bin/false  
 blacklist rds  
 install sctp /bin/false  
 blacklist sctp  
 EOF'  
 sudo modprobe -r dccp tipc rds sctp 2>/dev/null  
```
**9.2 Durcissement TCP/IP via Sysctl**  
Prévenir le spoofing IP, ignorer les requêtes ICMP broadcast, et activer les SYN cookies.  
```bash
sudo bash -c 'cat <<EOF >> /etc/sysctl.d/60-network-hardening.conf  
 net.ipv4.ip_forward = 0  
 net.ipv6.conf.all.forwarding = 0  
 net.ipv6.conf.default.forwarding = 0  
 net.ipv4.conf.all.accept_redirects = 0  
 net.ipv4.conf.default.accept_redirects = 0  
 net.ipv4.conf.all.secure_redirects = 0  
 net.ipv4.conf.default.secure_redirects = 0  
 net.ipv4.conf.all.send_redirects = 0  
 net.ipv4.conf.default.send_redirects = 0  
 net.ipv6.conf.all.accept_redirects = 0  
 net.ipv6.conf.default.accept_redirects = 0  
 net.ipv4.conf.all.rp_filter = 1  
 net.ipv4.conf.default.rp_filter = 1  
 net.ipv4.conf.all.log_martians = 1  
 net.ipv4.conf.default.log_martians = 1  
 net.ipv4.tcp_syncookies = 1  
 net.ipv4.icmp_echo_ignore_broadcasts = 1  
 net.ipv4.icmp_ignore_bogus_error_responses = 1  
 net.ipv6.conf.all.accept_ra = 0  
 net.ipv6.conf.default.accept_ra = 0  
 net.ipv4.conf.all.accept_source_route = 0  
 net.ipv4.conf.default.accept_source_route = 0  
 net.ipv6.conf.all.accept_source_route = 0  
 net.ipv6.conf.default.accept_source_route = 0  
 sysctl net.ipv4.conf.default.log_martians = 1  
 sysctl net.ipv4.conf.all.log_martians = 1  
 EOF'  
 sudo sysctl --system  
```
**9.4 Configuration du Pare-feu (UFW)**  
Remplacement de Iptables classique par UFW avec une politique de type "Deny All / Allow Exception".  
```bash
sudo apt -y purge iptables iptables-persistent nftables  
 sudo apt install -y ufw  
 sudo systemctl enable --now ufw  
 sudo bash -c 'ufw default deny incoming  
 ufw default allow outgoing  
 ufw default deny routed  
 ufw allow in on lo  
 ufw allow out on lo  
 ufw deny in from 127.0.0.0/8  
 ufw deny in from ::1  
 ufw default deny incoming  
 ufw default deny outgoing  
 ufw default deny routed  
 ufw deny in from 127.0.0.0/8  
 ufw allow proto tcp from any to any port 22'  
 sudo ufw --force enable  
```
**Phase 10: Authentification, Contrôle d'Accès et SSH**  
**10.1 Sécurisation du démon SSH**  
Création d'un profil sécurisé désactivant l'accès root et forçant l'usage de clés.  
🛑Assurez-vous que vous avez une clés ssh enregistrée sur le serveur avant d’effectuer l’action suivante  
```bash
sudo bash -c 'cat <<EOF > /etc/ssh/sshd_config.d/60-ssh-hardening.conf  
 Banner /etc/issue.net  
 PermitRootLogin no  
 # PasswordAuthentication no  
 PermitEmptyPasswords no  
 MaxAuthTries 4  
 ClientAliveInterval 30  
 ClientAliveCountMax 3  
 LoginGraceTime 60  
 MaxAuthTries 4  
 X11Forwarding no  
 AllowTcpForwarding no  
 IgnoreRhosts yes  
 HostbasedAuthentication no  
 UsePAM yes  
 PermitUserEnvironment no  
 DisableForwarding yes  
 MaxStartups 10:30:60  
 LogLevel VERBOSE  
 EOF'  
 sudo systemctl restart ssh  
```
Correction des permissions  
```bash
sudo chmod u-x,og-rwx /etc/ssh/sshd_config && \  
 sudo chown root:root /etc/ssh/sshd_config && \  
 sudo find /etc/ssh/sshd_config.d -type f -exec chmod u-x,og-rwx {} + 2>/dev/null && \  
 sudo find /etc/ssh/sshd_config.d -type f -exec chown root:root {} + 2>/dev/null  
```
**10.2 Durcissement de Sudo**  
Activer les logs Sudo pour l'audit.  
```bash
sudo bash -c 'cat <<EOF > /etc/sudoers.d/00-cis-hardening  
 Defaults use_pty  
 Defaults logfile="/var/log/sudo.log"  
 EOF'  
```
**10.3 Politique des Mots de Passe (PAM)**  
Imposer une politique stricte sur la complexité et l'expiration des mots de passe.  
```bash
sudo apt install -y libpam-runtime libpam-modules libpam-pwquality  
```
 # Configurer la complexité  
```bash
sudo bash -c 'cat <<EOF > /etc/security/pwquality.conf.d/60-hardening-pw.conf  
 difok = 2  
 minlen = 14  
 minclass = 3  
 dcredit = -1  
 ucredit = -1  
 ocredit = -1  
 lcredit = -1  
 maxrepeat = 3  
 maxsequence = 3  
 EOF'  
```
**Actions complémentaires :**  
1. Supprimez toute mention nullok dans les fichiers /etc/pam.d/common-auth et /etc/pam.d/common-password.  
2. Éditez /etc/login.defs pour configurer l'expiration :  
- PASS_MAX_DAYS 90  
- PASS_MIN_DAYS 1  
- PASS_WARN_AGE 7  
- ENCRYPT_METHOD SHA512  
**10.4 Vérification d'intégrité des comptes**  
- Vérifier que seul root possède l'UID 0 :  
```bash
awk -F: '$3 == 0 {print $1}' /etc/passwd  
```
   
- S'assurer que les comptes systèmes n'ont pas de shell valide:  
- Le resultat devrait contenir “root” et possiblement “sync” qui a pour shell /bin/sync  
```bash
awk -F: '$3 < 1000 && $7 != "/usr/sbin/nologin" && $7 != "/bin/false" {print $1}' /etc/passwd  
```
**Phase 11: Appliquer Redemarrage final**  
```bash
sudo reboot  
```