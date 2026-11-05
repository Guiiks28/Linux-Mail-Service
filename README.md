# 🐧 Module 07 : Administration d'un Service Linux (Postfix / Dovecot & Troubleshooting Journalctl)

## 🎯 Objectifs
- Déployer et administrer un serveur de messagerie Linux complet sous **Debian 13** (conteneur LXC).
- Configurer **Postfix** (MTA) pour l'acheminement local et réseau du courrier via le port `25`.
- Configurer **Dovecot** (MDA) pour la consultation des boîtes aux lettres via **IMAP** (port `143`).
- Maîtriser le diagnostic système sous **systemd** via `journalctl` pour la traçabilité des flux et des pannes.
- Évaluer et tester la sécurité contre les failles d'**Open Relay** (relais ouvert).

---

## 📋 Concepts Clés

### 1. Séparation des Rôles : MTA vs MDA
- **Postfix (Mail Transfer Agent - MTA)** : Responsable exclusivement de la réception et du transport des messages sur le port SMTP (25). Il dépose les mails dans `/var/mail/<user>`.
- **Dovecot (Mail Delivery Agent - MDA)** : Responsable de la mise à disposition des boîtes aux lettres aux clients finalisés (Thunderbird, `curl`) via le protocole IMAP (143).
- **Principe de Diagnostic** : Une panne d'envoi (SMTP) et une panne de consultation (IMAP) sont techniquement indépendantes et ne se recherchent pas au même endroit.

### 2. Gestion des logs sous Debian 13 (`journalctl`)
- Absence du fichier classique `/var/log/mail.log` (rsyslog retiré par défaut sur Debian 13).
- Utilisation de la facility syslog de `systemd` : `journalctl SYSLOG_FACILITY=2`.
- **Suivi par ID unique** : Chaque message génère une empreinte unique (ex: `01C912E5`) permettant d'isoler l'intégralité de sa traçabilité (`pickup` → `qmgr` → `local` / `status=sent`).

---

## 🛠️ Étapes de Configuration (Lab & Manipulations)

### 1️⃣ Déploiement du Service Postfix
```bash
# Installation des paquets requis
apt update && apt install -y postfix mailutils

# Configuration des variables d'identité du domaine
postconf -e "myhostname = srv-mail.test"
postconf -e "mydomain = srv-mail.test"
postconf -e "myorigin = \$mydomain"
postconf -e "mydestination = \$myhostname, \$mydomain, localhost"

# Ouverture de l'écoute sur toutes les interfaces réseau (défaut = loopback-only)
postconf -e "inet_interfaces = all"
systemctl restart postfix
```

### 2️⃣ Configuration de Dovecot (IMAP)
```bash
# Installation de Dovecot IMAP
apt install -y dovecot-imapd

# Configuration du stockage des boîtes aux lettres (Dovecot 2.4+)
cat /etc/dovecot/conf.d/99-atelier.conf <<'EOF'
mail_driver = mbox
mail_path = ~/mail
mail_inbox_path = /var/mail/%{user}
auth_allow_cleartext = yes
EOF

systemctl restart dovecot
```

### 3️⃣ Test d'Open Relay & Sécurisation
1. **Tentative de relayage externe (devrait échouer)** :
   ```bash
   curl -v --url "smtp://<IP-SERVEUR>" \
     --mail-from "pirate@ailleurs.test" \
     --mail-rcpt "victime@exemple-externe.test" \
     --upload-file /tmp/relais.txt
   # Résultat attendu : 454 4.7.1 Relay access denied
   ```
2. **Simulation d'une vulnérabilité (Open Relay)** :
   ```bash
   postconf -e "mynetworks = 0.0.0.0/0"
   systemctl restart postfix
   # La tentative de relayage passe (faille ouverte)
   ```
3. **Remédiation immédiate** :
   ```bash
   postconf -e "mynetworks = 127.0.0.0/8"
   systemctl restart postfix
   ```

---

## 🧪 Validation & Troubleshooting

1. **Vérification de l'écoute des ports** :
   ```bash
   ss -tulpn | grep -E ":25|:143"
   ```
2. **Test de lecture des mails via `curl`** :
   ```bash
   curl --url "imap://<IP-SERVEUR>" --user "bob:<password>"
   curl --url "imap://<IP-SERVEUR>/INBOX;UID=1" --user "bob:<password>"
   ```
3. **Analyse en temps réel du trafic de messagerie** :
   ```bash
   # Écoute en direct des événements mail
   journalctl SYSLOG_FACILITY=2 -f

   # Isolation d'un identifiant spécifique
   journalctl SYSLOG_FACILITY=2 | grep <ID_MESSAGE>
   ```

---

## ⚠️ Pièges Récurents & Retours d'Expérience
- **`systemctl status` ne valide pas le fonctionnement** : Un service affiché `active (running)` peut être incapable de livrer du courrier. Seule la présence du marqueur `status=sent` dans les logs atteste de la livraison effective.
- **Syntaxes obsolètes de Dovecot** : Sur les versions récentes (Dovecot 2.4+), le paramètre `mail_location` n'existe plus et empêche le démarrage du service.
- **Relais externe bloqué** : La livraison vers un domaine externe réel échouera en l'absence de nom de domaine public et d'enregistrements DNS appropriés (SPF, DKIM, DMARC, Reverse DNS).
