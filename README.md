
# TP Réseaux Multimédia – VoIP et Asterisk

Ce dépôt présente un projet pédagogique autour de la **VoIP** et de la configuration d'un serveur **Asterisk 20**.  
Il est divisé en deux parties :

-   **Théorie** : Calculs de bande passante, latence, multiplexage (voir `docs/tp_voip_theorique.pdf`)
    
-   **Pratique** : Installation et configuration d'Asterisk avec conférences audio et vidéo (voir `docs/guide_pratique_asterisk.md`)
    

----------

## 📂 Structure principale

-   **docs/**
    
    -   `tp_voip_theorique.pdf` : Sujet et partie théorique du TP
        
    -   `guide_pratique_asterisk.md` : Guide complet d'installation, configuration et tests Asterisk
        
-   **config/**
    
    -   `pjsip.conf` : Endpoints utilisateurs SIP (7001–7003)
        
    -   `extensions.conf` : Plan de numérotation, appels internes et conférences (6000, 6100, 6001)
        
    -   `confbridge.conf` : Profils de conférences (audio, vidéo, admin)
        
    -   `modules.conf` : Chargement automatique des modules, dont `app_confbridge`
        

----------

## 🚀 Démarrage rapide

Cloner le dépôt :

```bash
git clone <url-du-repo>
cd voip-asterisk-tp

```

Installer les dépendances et compiler Asterisk :

```bash
sudo apt install -y build-essential git wget curl libxml2-dev libncurses5-dev uuid-dev libjansson-dev libsqlite3-dev pkg-config libssl-dev libedit-dev
wget https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-20-current.tar.gz
tar xzf asterisk-20-current.tar.gz
cd asterisk-20.*
./configure && make menuselect && make -j$(nproc)
sudo make install
sudo make samples

```

Copier les fichiers de configuration dans `/etc/asterisk/` (contenus dans `config/`), puis lancer Asterisk :

```bash
sudo asterisk -cvvv

```

----------

## 📞 Tests

-   7001 ↔ 7002 : Appel audio/vidéo interne
    
-   `6000` : Conférence **audio seule**
    
-   `6100` : Conférence **audio + vidéo** (follow talker)
    
-   `6001` : Conférence **admin** (gestion mute/kick)
    

👉 Clients testés avec **PortSIP UC** (meilleure prise en charge H.264/VP8 que Zoiper).

----------

## 🛠️ Documentation

-   Partie théorique → `docs/tp_voip_theorique.pdf`
    
-   Partie pratique → `docs/guide_pratique_asterisk.md`
    

----------

## 📋 Licence

Projet open-source sous licence MIT.
