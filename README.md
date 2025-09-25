# TP Réseaux Multimédia – VoIP et Asterisk

Ce dépôt contient le **projet de TP en Réseaux Multimédia**, portant sur l’étude de la VoIP et la mise en place d’un serveur **Asterisk** avec fonctionnalités de conférences audio et vidéo.

---

## 🎯 Objectifs du TP

1. **Partie théorique**  
   - Comprendre le fonctionnement de la voix sur IP (VoIP).  
   - Réaliser des calculs : bande passante, latence, multiplexage, occupation utile…  
   - Étudier les codecs (G.711, G.722, H.264, VP8) et leurs contraintes.  

2. **Partie pratique**  
   - Installer et configurer un serveur Asterisk 20.  
   - Créer et tester des comptes SIP utilisateurs (7001, 7002, 7003).  
   - Mettre en place des conférences :  
     - `6000` → **conférence audio seule**  
     - `6100` → **conférence audio + vidéo** (follow talker)  
     - `6001` → **conférence administrateur** (droits de mute/kick)  
   - Tester avec le softphone **PortSIP UC**, recommandé pour sa gestion complète des codecs vidéo modernes (H.264, VP8).

---

## 📂 Contenu du dépôt

- **docs/**  
  - `projets-reseaux-multimedia-1.pdf` → Sujet du TP et énoncés des questions.  
  - `rapport_calculs.md` → Résolution des questions théoriques.  
  - `guide_pratique_asterisk.md` → Guide détaillé pour l’installation, la configuration et les tests.  

- **config/**  
  - `pjsip.conf` → Configuration des endpoints SIP (7001–7003).  
  - `extensions.conf` → Plan de numérotation interne et conférences (6000, 6100, 6001).  
  - `confbridge.conf` → Profils de conférences (audio, vidéo, admin).  

- **resources/**  
  - Répertoire prévu pour les captures d’écran et schémas illustrant le TP.  

- **README.md** → Présentation générale du projet.  

---

## 🛠️ Résumé pratique

- Les explications complètes sont dans :  
  👉 `docs/guide_pratique_asterisk.md`  
- Les fichiers nécessaires au bon fonctionnement d’Asterisk sont dans :  
  👉 `config/`  

---

## 📌 Conclusion

Ce TP permet d’allier **analyse théorique** et **application pratique**, en abordant :  
- les calculs de performance VoIP,  
- la gestion de la voix et de la vidéo,  
- et le déploiement d’un serveur de communication open-source (**Asterisk**).
