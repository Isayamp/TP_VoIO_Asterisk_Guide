# Projet Réseaux Multimédia – TP VoIP avec Asterisk

## 🎯 Objectif
Mettre en place un serveur **Asterisk** fonctionnel sous **Ubuntu**, en utilisant le codec **G.711** pour la VoIP, et tester des communications internes avec des **softphones (Zoiper)**.  
Nous allons combiner **théorie** (questions du TP) et **pratique** (installation, configuration et dépannage).

---

## 1. Partie théorique (Résumé des calculs)
- **Q1 – Temps de remplissage :** 20 ms/trame, transmission 1,7 µs → latence ≈ 20 ms.
- **Q2 – Taille minimale Ethernet :** Trame = 218 octets > 64 octets → OK.
- **Q3 – Occupation utile :** 160/218 ≈ 73 % → 27 % overhead.
- **Q4 – Latence bout-en-bout :** ≈ 20 ms (acceptable ITU-T < 150 ms).
- **Q5 – Multiplexage :** ≥ 4 flux nécessaires pour efficacité > 90 %.
- **Q6 – Conférence téléphonique :** OK, limité par CPU/RAM serveur Asterisk.
- **Q7 – Vidéo HD :** Brute = 1.49 Gbit/s (impossible), compressée (H.264) = 2-5 Mbit/s (faisable).

---

## 2. Partie pratique : Installation d’Asterisk

### Pré-requis
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wget curl gnupg2 lsb-release git build-essential -y
sudo apt install libncurses5-dev libssl-dev libxml2-dev libsqlite3-dev uuid-dev -y
```

### Télécharger et compiler
```bash
cd /usr/src
sudo wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-20-current.tar.gz
sudo tar xvf asterisk-20-current.tar.gz
cd asterisk-20.*/
sudo contrib/scripts/install_prereq install
./configure
make menuselect
make
sudo make install
sudo make samples
sudo make config
sudo ldconfig
```

### ⚠️ Problèmes rencontrés & solutions
- **Erreur Permission denied sur `config.log`**  
  ➝ Corrigée par :  
  ```bash
  sudo chown -R $USER:$USER /usr/src/asterisk-20.*/
  ./configure
  ```

- **Pas de transport SIP (`pjsip show transports` → No objects found)**  
  ➝ Solution : ajouter en haut de `/etc/asterisk/pjsip.conf` :
  ```ini
  [transport-udp]
  type=transport
  protocol=udp
  bind=0.0.0.0:5060
  ```

- **Firewall bloquant SIP/RTP**  
  ➝ Solution :  
  ```bash
  sudo ufw allow 5060/udp
  sudo ufw allow 10000:20000/udp
  ```

---

## 3. Configuration d’Asterisk

> ℹ️ Les fichiers de configuration se trouvent dans `/etc/asterisk/`.  
> ⚠️ **Important : vous devez avoir les droits d'administrateur (root)** pour les éditer.  
> Par exemple :  
> - Avec `nano` : `sudo nano /etc/asterisk/pjsip.conf`  
> - Avec **VS Code** : ouvrez en mode `sudo code .` (il demandera le mot de passe admin).

### `/etc/asterisk/pjsip.conf`
```ini
[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0:5060

; Poste 7001
[7001]
type=endpoint
context=from-internal
disallow=all
allow=ulaw
auth=auth7001
aors=7001

[auth7001]
type=auth
auth_type=userpass
username=7001
password=motdepasse1

[7001]
type=aor
max_contacts=1

; Poste 7002
[7002]
type=endpoint
context=from-internal
disallow=all
allow=ulaw
auth=auth7002
aors=7002

[auth7002]
type=auth
auth_type=userpass
username=7002
password=motdepasse2

[7002]
type=aor
max_contacts=1
```

### `/etc/asterisk/extensions.conf`
```ini
[from-internal]
exten => 7001,1,Dial(PJSIP/7001)
exten => 7002,1,Dial(PJSIP/7002)

; Salle de conférence (optionnelle)
exten => 6000,1,Answer()
 same => n,ConfBridge(myroom)
 same => n,Hangup()
```

---

## 4. Démarrage et commandes utiles

### Démarrer Asterisk
```bash
sudo systemctl start asterisk
sudo systemctl enable asterisk
```

Vérifier état :  
```bash
sudo systemctl status asterisk
```

Entrer dans la CLI :  
```bash
sudo asterisk -rvvv
```

### Commandes utiles de debug
```bash
pjsip show transports    ; Vérifier transport SIP
pjsip show endpoints     ; Voir les comptes configurés
pjsip show contacts      ; Voir les softphones connectés
pjsip show endpoint 7001 ; Vérifier un compte spécifique
```

---

## 5. Configuration Zoiper (Client)

1. Installer Zoiper (mobile ou PC).  
2. Créer un compte SIP avec :  
   - **Host/Domain** : IP Ubuntu → (ex: `172.16.0.134`)  
   - **Port** : 5060  
   - **User** : 7001 (ou 7002)  
   - **Password** : motdepasse1 (ou motdepasse2)  
   - **Transport** : UDP  

3. Vérifier l’enregistrement :  
   ```bash
   pjsip show contacts
   ```

✅ Si l’endpoint apparaît avec l’IP de ton téléphone → connecté.

---

## 6. Tester un appel
- Depuis Zoiper connecté en `7001`, composer `7002`.  
- Sur l’Asterisk CLI, voir l’exécution du dialplan :  
  ```
  -- Executing [7002@from-internal:1] Dial("PJSIP/7001-00000001", "PJSIP/7002")
  ```

---

## 🚀 Conclusion
- Théorie : évaluation de latence, efficacité, multiplexage et faisabilité VoIP/Vidéo.  
- Pratique : installation **Asterisk sur Ubuntu**, configuration SIP & extensions, tests avec Zoiper.  
- Dépannage abordé (permissions root, transport SIP absent, firewall).  
- Résultat : **mini-PBX opérationnel** sur ton réseau local, permettant appels VoIP et conférence.

---
