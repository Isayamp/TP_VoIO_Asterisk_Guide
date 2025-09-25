Guide pratique Asterisk 20 – VoIP, conférences audio et vidéo (ConfBridge)

Ce guide documente l’installation, la configuration et les tests d’un serveur Asterisk 20 pour la VoIP, avec mise en place de conférences audio et audio+vidéo via ConfBridge. Il est conçu pour être un support pratique, avec des commandes directement copiables et des explications concises.

Pourquoi PortSIP UC ?
Nous utilisons PortSIP UC comme client SIP car il offre une excellente gestion des codecs vidéo modernes (H.264, VP8) et une interface utilisateur complète, ce qui est essentiel pour tester les fonctionnalités de visioconférence d’Asterisk, contrairement à d’autres softphones plus basiques.

1. Prérequis système et réseau

Assurez-vous que votre système Ubuntu/Debian est à jour et que les ports nécessaires sont ouverts.

# Mettre à jour le système
sudo apt update && sudo apt upgrade -y

# Vérifier l'adresse IP locale du serveur Asterisk
ip addr show | grep -E "inet .* scope global"

# Ouvrir les ports nécessaires avec UFW (si activé)
sudo ufw allow 5060/udp comment 'SIP signaling'
sudo ufw allow 10000:20000/udp comment 'RTP audio/video'
sudo ufw reload

2. Installation d’Asterisk (depuis les sources)

Cette procédure installe Asterisk 20 LTS en compilant depuis les sources.

# Installer les dépendances de compilation
sudo apt install -y build-essential git wget curl libxml2-dev libncurses5-dev uuid-dev libjansson-dev libsqlite3-dev pkg-config libssl-dev libedit-dev

# Télécharger et décompresser les sources d'Asterisk (version 20 LTS)
cd ~
wget https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-20-current.tar.gz
tar xzf asterisk-20-current.tar.gz
cd asterisk-20.*

# Installer les prérequis et les sources MP3 (si besoin)
contrib/scripts/get_mp3_source.sh
contrib/scripts/install_prereq install

# Configurer le build et sélectionner les modules
./configure
make menuselect


Dans le menu menuselect (utiliser les flèches et la barre espace):

Aller dans Applications et cocher app_confbridge.

Aller dans Formats/codecs et activer format_mp3 si vous utilisez des musiques d'attente MP3.

Aller dans Sounds Packages et sélectionner les prompts CORE-SOUNDS-* et MOH-* (par exemple CORE-SOUNDS-EN-ULAW et MOH-OPSOUND-ULAW).

Sauvegarder et quitter.

# Compiler Asterisk (peut prendre du temps)
make -j$(nproc)

# Installer Asterisk et les fichiers de configuration par défaut
sudo make install
sudo make samples
sudo make config
sudo ldconfig


Permissions :
Il est crucial de définir les bonnes permissions pour qu'Asterisk puisse écrire dans ses dossiers. Si vous rencontrez des erreurs de permission, exécutez:

sudo adduser --system --group --home /var/lib/asterisk asterisk
sudo chown -R asterisk:asterisk /etc/asterisk /var/{lib,log,spool,run}/asterisk

3. Démarrage et arrêt d’Asterisk

Après une compilation depuis les sources, Asterisk n'est pas toujours géré nativement par systemctl. Voici les méthodes de démarrage.

Démarrage en mode console (pour le développement et le débogage) :

sudo asterisk -cvvv


Ceci démarre Asterisk et ouvre la ligne de commande interactive (CLI). Pour vous connecter à cette instance depuis un autre terminal :

sudo asterisk -rvvv


Démarrage en tant que service Systemd (recommandé pour la production) :
Créez un fichier de service Systemd pour gérer Asterisk comme un service standard.

sudo nano /etc/systemd/system/asterisk.service


Collez le contenu suivant :

[Unit]
Description=Asterisk PBX and telephony daemon
After=network.target

[Service]
Type=forking
User=asterisk
Group=asterisk
ExecStart=/usr/sbin/asterisk -f -U asterisk -G asterisk
ExecStop=/usr/sbin/asterisk -rx 'core stop now'
Restart=on-failure

[Install]
WantedBy=multi-user.target


Rechargez Systemd, activez et démarrez le service :

sudo systemctl daemon-reload
sudo systemctl enable asterisk
sudo systemctl start asterisk
sudo systemctl status asterisk


Vérification du socket de connexion :
Si vous ne pouvez pas vous connecter à la CLI (Unable to connect to remote asterisk), vérifiez l'existence du fichier de socket :

ls -l /var/run/asterisk/asterisk.ctl

4. Activation du module ConfBridge

Le module app_confbridge.so est essentiel pour les conférences. Assurez-vous qu'il est chargé et actif.

# Vérifier le statut du module ConfBridge
*CLI> module show like confbridge

# Si le module n'est pas 'Running', le charger manuellement
*CLI> module load app_confbridge.so


Pour que le module se charge automatiquement au démarrage d'Asterisk, éditez /etc/asterisk/modules.conf et assurez-vous d'avoir :

[modules]
autoload=yes
load => app_confbridge.so

5. Fichiers de configuration Asterisk

Les fichiers de configuration essentiels sont disponibles dans le dossier config/ de votre dépôt Git. Vous devez les copier dans /etc/asterisk/.

a) /etc/asterisk/pjsip.conf (Configuration des endpoints SIP)
Ce fichier définit les utilisateurs SIP (7001, 7002, 7003) et les codecs audio/vidéo autorisés.

[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0:5060

[7001]
type=endpoint
aors=7001
auth=7001
disallow=all
allow=ulaw,alaw,g722        ; Codecs audio recommandés
allow=h264,vp8               ; Codecs vidéo recommandés
context=from-internal

[7001]
type=auth
auth_type=userpass
password=motdepasse1
username=7001

[7001]
type=aor
max_contacts=1

[7002](7001)                  ; Hérite des paramètres de l'endpoint 7001
username=7002
password=motdepasse2

[7002]
type=aor
max_contacts=1

[7003](7001)[ (7001)](7001 "url-only")
username=7003
password=motdepasse3

[7003]
type=aor
max_contacts=1


b) /etc/asterisk/extensions.conf (Plan de numérotation)
Ce fichier définit les règles d'appel, y compris les appels internes directs et les extensions pour les différentes conférences.

; ===========================================================
; Plan de numérotation interne avec conférences AUDIO et VIDÉO
; ===========================================================

[from-internal]

; --------------------- Appels internes directs ---------------------
exten => 7001,1,Dial(PJSIP/7001)
exten => 7002,1,Dial(PJSIP/7002)
exten => 7003,1,Dial(PJSIP/7003)

; --------------------- Conférence AUDIO seule (6000) ---------------
exten => 6000,1,Answer()
 same => n,Wait(1)
 same => n,Playback(conf-getconfno)
 same => n,ConfBridge(audio_room,audio_bridge,audio_user)
 same => n,Hangup()

; --------------------- Conférence AUDIO+VIDÉO (6100) ----------------
exten => 6100,1,Answer()
 same => n,Wait(1)
 same => n,Playback(conf-getconfno)
 same => n,ConfBridge(video_room,video_bridge,video_user)
 same => n,Hangup()

; --------------------- Conférence ADMIN (6001) ----------------------
exten => 6001,1,Answer()
 same => n,Wait(1)
 same => n,Playback(conf-getconfno)
 same => n,ConfBridge(adminroom,video_bridge,admin_user)
 same => n,Hangup()


c) /etc/asterisk/confbridge.conf (Profils de conférence)
Ce fichier définit les profils pour les ponts de conférence (bridge) et les utilisateurs (user), en distinguant les conférences audio et vidéo. Important : Sur Asterisk 20, les options video_mode et talk_detection ne sont valides que dans les profils bridge, pas dans les profils user.

[general]

; ------------ Profils CONF AUDIO SEULE (6000) ------------
[audio_bridge]
type=bridge
max_members=10
record_conference=no
mixing_interval=20
internal_sample_rate=8000
video_mode=none                 ; Pas de vidéo dans ce bridge

[audio_user]
type=user
music_on_hold_when_empty=yes
dsp_drop_silence=yes
denoise=yes
jitterbuffer=yes

; ------------ Profils CONF AUDIO+VIDÉO (6100/6001) --------
[video_bridge]
type=bridge
max_members=10
record_conference=no
mixing_interval=20
internal_sample_rate=16000
video_mode=follow_talker        ; La vidéo suit l’orateur actif

[video_user]
type=user
music_on_hold_when_empty=yes
dsp_drop_silence=yes
denoise=yes
jitterbuffer=yes

; ------------ Profil UTILISATEUR ADMIN (6001) -------------
[admin_user]
type=user
music_on_hold_when_empty=yes
dsp_drop_silence=yes
denoise=yes
jitterbuffer=yes
marked=yes                      ; Rôle prioritaire
admin=yes                       ; Droits de gestion (mute/kick)
startmuted=no


Après avoir copié ces fichiers, rechargez les configurations :

# Dans la CLI Asterisk
*CLI> dialplan reload
*CLI> pjsip reload
*CLI> module reload app_confbridge.so

6. Gestion des codecs audio/vidéo

Assurez-vous que les codecs configurés dans pjsip.conf sont bien supportés par Asterisk et vos clients SIP.

# Vérifier les codecs audio supportés par Asterisk
*CLI> core show codecs audio

# Vérifier les codecs vidéo supportés par Asterisk
*CLI> core show video codecs

7. Configuration PortSIP UC (client SIP)

Pour chaque poste (7001, 7002, 7003), configurez PortSIP UC comme suit :

SIP ID / Username : 7001 (ou 7002, 7003)

Password : motdepasse1 (ou motdepasse2, motdepasse3)

Domain / Server : L'adresse IP de votre serveur Asterisk (ex: 172.16.0.134)

Port : 5060

Transport : UDP

Codecs : Activez G.711 (uLaw/aLaw), G.722 pour l'audio, et H.264, VP8 pour la vidéo. Priorisez H.264.

Caméra : Activez la caméra dans PortSIP UC pour les tests de visioconférence.

Vérification de l'enregistrement côté Asterisk :

*CLI> pjsip show contacts
*CLI> pjsip show aor 7001

8. Tests fonctionnels – Checklist

Suivez ces étapes pour valider le bon fonctionnement de votre serveur Asterisk et des conférences.

Vérifier le module ConfBridge et les profils :
*CLI> module show like confbridge      # Doit afficher 'Running'
*CLI> confbridge show profile bridges  # Doit lister audio_bridge, video_bridge
*CLI> confbridge show profile users    # Doit lister audio_user, video_user, admin_user

Tests d'appels internes (audio et vidéo) :

Depuis PortSIP UC (7001), appelez 7002.

Vérifiez que l'audio et la vidéo passent correctement.

Test de la Conférence AUDIO seule (Extension 6000) :

Depuis PortSIP UC (7001) et (7002), composez 6000.

Vérifiez que les participants s'entendent, mais qu'aucune vidéo ne circule.

Test de la Conférence AUDIO + VIDÉO (Extension 6100) :

Depuis PortSIP UC (7001), (7002) et (7003), composez 6100.

Vérifiez que l'audio est mixé et que la vidéo de l'orateur actif est diffusée à tous les participants (follow_talker).

Test de la Conférence Administrateur (Extension 6001) :

Depuis PortSIP UC (7001), composez 6001 (vous êtes l'administrateur).

Depuis PortSIP UC (7002) et (7003), composez 6001.

Vérifiez que l'administrateur (7001) peut utiliser les commandes de gestion (ex: confbridge mute 6001 7002, confbridge kick 6001 7003).

Monitoring en CLI pendant les tests :
*CLI> confbridge list                  # Lister toutes les conférences actives
*CLI> confbridge list video_room       # Lister les participants d'une salle spécifique
*CLI> rtp set debug on                 # Activer le débogage RTP pour voir les flux média
*CLI> core show channels verbose       # Afficher les détails des canaux actifs

9. Dépannage – Erreurs fréquentes et solutions

Module ConfBridge non chargé :

Symptômes : No such command 'confbridge ...' ou app_confbridge.so Not Running.

Solution : Charger le module manuellement (*CLI> module load app_confbridge.so) et vérifier modules.conf pour l'autoload.

Profils de conférence introuvables :

Symptômes : Conference bridge profile audio_bridge does not exist.

Solution : Vérifier la syntaxe de /etc/asterisk/confbridge.conf, s'assurer que les sections [bridge_name] et [user_name] sont correctement définies, puis redémarrer Asterisk (sudo systemctl restart asterisk).

Erreurs de configuration ConfBridge (options non supportées) :

Symptômes : ERROR: config_options.c: Could not find option suitable for category 'audio_user' named 'video_mode'.

Solution : Supprimer les options non supportées des profils user (ex: video_mode, talk_detection). Ces options sont souvent réservées aux profils bridge.

Problèmes de permissions :

Symptômes : Asterisk ne démarre pas, ne peut pas écrire dans les logs, ou la CLI ne se connecte pas (Unable to connect to remote asterisk).

Solution : Assurez-vous que l'utilisateur asterisk possède les droits sur les dossiers /etc/asterisk, /var/lib/asterisk, /var/log/asterisk, /var/spool/asterisk, /var/run/asterisk.

sudo adduser --system --group --home /var/lib/asterisk asterisk
sudo chown -R asterisk:asterisk /etc/asterisk /var/{lib,log,spool,run}/asterisk


Problèmes de codecs (audio ou vidéo) :

Symptômes : Cannot determine best translation path ou pas de son/vidéo.

Solution : Vérifier que les codecs allow= dans pjsip.conf correspondent aux codecs activés dans PortSIP UC. Utiliser ulaw/alaw/g722 pour l'audio et h264/vp8 pour la vidéo.

Problèmes de NAT / Pare-feu :

Symptômes : Appels qui ne passent pas, audio unidirectionnel, pas de vidéo.

Solution : Ouvrir les ports 5060/UDP et 10000-20000/UDP sur le pare-feu. Si vous êtes derrière un NAT, configurez external_media_address et external_signaling_address dans pjsip.conf et utilisez rtp_symmetric=yes, force_rport=yes.

Pas de vidéo en conférence :

Symptômes : Audio OK, mais pas de vidéo dans la conférence 6100/6001.

Solution : Vérifier que video_mode=follow_talker est bien défini dans le [video_bridge] de confbridge.conf. S'assurer que les clients SIP ont la caméra activée et partagent un codec vidéo commun.

10. Commandes utiles (CLI Asterisk)
# Afficher l'aide pour une commande
*CLI> core show help <commande>

# Augmenter la verbosité des logs (pour le débogage)
*CLI> core set verbose 5
*CLI> core set debug 3

# Activer le logger PJSIP (pour voir les messages SIP)
*CLI> pjsip set logger on

# Recharger les modules spécifiques
*CLI> module reload res_pjsip.so
*CLI> module reload app_confbridge.so

# Recharger le dialplan
*CLI> dialplan reload

# Recharger PJSIP
*CLI> pjsip reload

# Arrêter Asterisk proprement
*CLI> core stop now


Dernière mise à jour: 2025-09-25