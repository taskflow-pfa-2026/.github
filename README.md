# 🛡️ TaskFlow - Plateforme DevSecOps de Gestion et Rotation de Secrets

> **Projet de Fin d'Année (PFA) 2026**  
> Architecture complète de prévention, détection et réponse automatisée aux fuites de secrets applicatifs dans un environnement hétérogène.

**Équipe :**
- 👤 **Mohamed TALBI** – *Volet A* : Architecture, Vault, Rotation API, Résilience Applicative.
- 👤 **Ismail MABCHOUR** – *Volet B* : Détection (SIEM Wazuh), Scanning, Playbook de réponse, MTTR.

---

## 📖 Résumé Exécutif

TaskFlow est une application de gestion de tâches conçue comme un **laboratoire DevSecOps**. Au-delà de la fonctionnalité métier, ce projet démontre une chaîne de sécurité complète pour le cycle de vie des secrets, déployée sur des environnements croisés (Windows/Linux) reliés de manière sécurisée :
1. **Prévention (Shift-Left)** : Blocage des fuites dès le `pre-commit` via Gitleaks.
2. **Détection** : Surveillance continue de l'historique Git et des configurations par un SIEM (Wazuh).
3. **Réponse Automatisée** : Rotation atomique et instantanée des secrets compromis via une API dédiée, sans interruption de service.

---

## 💻 Environnements d'Exécution et Interconnexion

Ce projet simule un environnement d'entreprise réel où les développeurs et les équipes sécurité utilisent des systèmes d'exploitation différents. La communication est assurée de manière sécurisée via **Tailscale** (VPN Zero Trust).

| Volet | Responsable | OS Hôte | Environnement d'exécution | Particularités techniques gérées |
| :--- | :--- | :--- | :--- | :--- |
| **Volet A** (App & Secrets) | Dmitri | **Windows 11** | Docker Desktop (Backend WSL2), PowerShell, VS Code | Gestion des permissions de volumes WSL2 pour Vault, hot-reloading, résolution des conflits de ports Windows. |
| **Volet B** (SIEM & Détection) | [Nom du binôme] | **Linux** (Ubuntu/Debian) | Docker Engine (natif), Bash, Systemd, Cron | Déploiement natif de Wazuh, services systemd pour le watcher, automatisation via crontab. |
| **Liaison** | Les deux | N/A | **Tailscale** | Réseau privé maillé permettant au Volet B d'appeler l'API de rotation du Volet A via l'IP magique `100.x.x.x` sans exposition sur Internet. |

---

## 🏛️ Architecture Unifiée

```text
┌───────────────────────────────────────────────────────────────────────────────┐
│                            TASKFLOW ECOSYSTEM                                 │
├───────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  [VOLET B : DÉTECTION & RÉPONSE - LINUX]         [VOLET A : PRÉVENTION & EXÉCUTION - WINDOWS]
│                                                                               │
│  ┌─────────────────┐      ┌─────────────────┐      ┌──────────────────────┐  │
│  │  Repos Git      │─────▶│  Scanner        │─────▶│  Wazuh SIEM          │  │
│  │  (Historique)   │      │  (Gitleaks +    │      │  (Règles & Scoring)  │  │
│  └─────────────────┘      │   TruffleHog)   │      └──────────┬───────────┘  │
│                           └─────────────────┘                 │               │
│                                                               ▼               │
│                                                   ┌──────────────────────┐    │
│                                                   │  Playbook.py         │    │
│                                                   │  (Contrat Événement) │    │
│                                                   └──────────┬───────────┘    │
│                                                              │ (HTTP POST)    │
│  ┌─────────────────┐      ┌─────────────────┐               ▼               │
│  │  Frontend       │─────▶│  Backend        │◀─────┐ ┌──────────────────────┐  │
│  │  (React :3000)  │      │  (Flask :5000)  │      │ │  Rotation API        │  │
│  └─────────────────┘      └────────┬────────┘      │ │  (FastAPI :9000)     │  │
│                                    │               │ └──────────┬───────────┘  │
│                                    ▼               │            │               │
│                           ┌─────────────────┐      │            ▼               │
│                           │  Vault          │◀─────┘   ┌──────────────────────┐  │
│                           │  (Mode Server   │          │  Notification API    │  │
│                           │   Persistant)   │          │  (Factice :8080)     │  │
│                           └────────┬────────┘          └──────────────────────┘  │
│                                    │                                              │
│                           ┌────────▼────────┐                                     │
│                           │  PostgreSQL     │                                     │
│                           │  (:5432)        │                                     │
│                           └─────────────────┘                                     │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🗺️ Carte des Dépôts

| Dépôt | Rôle | Responsable |
| :--- | :--- | :--- |
| 📂 **[repo-infra](./repo-infra)** | Orchestration Docker, configuration Vault (persistant), scripts d'initialisation. | Dmitri |
| 📂 **[repo-backend](./repo-backend)** | API métier Flask, client Vault, authentification JWT, Notification API. | Dmitri |
| 📂 **[repo-frontend](./repo-frontend)** | Interface utilisateur React, programmation défensive contre les erreurs d'API. | Dmitri |
| 📂 **[repo-secrets-mgmt](./repo-secrets-mgmt)** | Moteur de rotation (FastAPI), logique de routage contextuel, rotateurs spécifiques. | Dmitri |
| 📂 **[repo-security](./repo-security)** | SIEM Wazuh, pipelines de scan, playbook d'orchestration, suivi du MTTR. | [Nom du binôme] |

---

## 🚀 Guide de Lancement Détaillé

Pour une démonstration réussie, il est crucial de lancer les environnements dans le bon ordre.

### 🪟 Étape 1 : Lancement du Volet A (Machine Windows - Dmitri)

1. **Cloner le dépôt d'infrastructure et les depots de l'application :**
Dans le meme dossier:
   ```powershell
   git clone https://github.com/taskflow-pfa-2026/repo-infra.git
   git clone https://github.com/taskflow-pfa-2026/repo-backend.git
   git clone https://github.com/taskflow-pfa-2026/repo-frontend.git
   git clone https://github.com/taskflow-pfa-2026/repo-secrets-mgmt.git
   cd repo-infra
   ```
3. **Exécuter le script de démarrage maître :**
   ```powershell
   .\start-taskflow.ps1
   ```
   *Ce script effectue automatiquement :*
   - Le lancement de tous les conteneurs Docker.
   - L'initialisation et le déscellement (Unseal) de Vault.
   - L'injection des 6 secrets de démonstration.
   - La sauvegarde du Token Root dans le fichier `.env`.
4. **Récupérer l'IP Tailscale :**
   ```powershell
   tailscale ip -4
   ```
   *(Communiquer cette IP `100.x.x.x` au binôme pour la configuration du Volet B).*
5. **Vérification :** Accéder à `http://localhost:3000` (Frontend) et `http://localhost:8200/ui` (Vault).

---

### 🐧 Étape 2 : Lancement du Volet B (Machine Linux - Binôme)

1. **Cloner le dépôt de sécurité :**
   ```bash
   git clone https://github.com/taskflow-pfa-2026/repo-security.git
   cd repo-security
   ```
2. **Installer les dépendances système et Python :**
   ```bash
   chmod +x setup_vm.sh
   ./setup_vm.sh
   pip install -r scripts/requirements.txt
   ```
3. **Configurer la liaison avec le Volet A :**
   ```bash
   nano config/rotation_api.conf
   ```
   *Remplacer l'URL par : `http://<IP_TAILSCALE_DE_DMITRI>:9000`*
4. **Déployer Wazuh (Single-Node) :**
   ```bash
   cd ~/wazuh-docker/single-node
   docker compose up -d
   # Attendre ~1 minute que les conteneurs soient "healthy"
   ```
5. **Charger les règles et décodeurs personnalisés :**
   ```bash
   docker cp wazuh/config/local_rules.xml single-node-wazuh.manager-1:/var/ossec/etc/rules/local_rules.xml
   docker cp wazuh/config/local_decoder.xml single-node-wazuh.manager-1:/var/ossec/etc/decoders/local_decoder.xml
   docker restart single-node-wazuh.manager-1
   ```
6. **Activer le service de surveillance (Watcher) :**
   ```bash
   sudo systemctl enable --now taskflow-watcher
   sudo systemctl status taskflow-watcher  # Doit être "active (running)"
   ```
7. **Programmer le scan automatique (optionnel pour la démo) :**
   ```bash
   crontab -e
   # Ajouter : */5 * * * * /home/<user>/volet_sec/scripts/scan.sh >> /home/<user>/volet_sec/logs/cron.log 2>&1
   ```

---

## 🎬 Scénario de Démonstration (Le "Wow Effect")

Voici le flux que nous présentons au jury pour illustrer la valeur ajoutée de l'architecture :

1. **La Faute** : Un développeur commite par erreur une clé API dans `repo-backend`.
2. **La Barrière (Shift-Left)** : Le hook `pre-commit` Gitleaks **bloque** l'opération. *(Démo 1)*
3. **Le Contournement** : Le développeur force le push (`git commit --no-verify`). Le secret est maintenant dans l'historique.
4. **La Détection** : Le script de scan du Volet B détecte la fuite. Wazuh génère une alerte de niveau "Critical".
5. **La Réponse Automatisée** : Le playbook intercepte l'alerte, calcule un score de risque > 90, et envoie le contrat JSON à la Rotation API via le réseau Tailscale.
6. **La Rotation** : L'API valide le contrat, génère une nouvelle clé cryptographiquement sûre, et l'écrit dans Vault. Le MTTR est de **< 5 secondes**.
7. **La Résilience** : L'application continue de fonctionner. Lors de la prochaine requête, le backend récupère silencieusement la nouvelle clé depuis Vault. *(Démo 2)*

---

## 🔐 Concepts DevSecOps Clés Implémentés

- **Shift-Left Security** : Détection au plus tôt dans le cycle de développement.
- **Zero-Trust Architecture** : Aucun secret en dur dans le code ; tout est récupéré dynamiquement via Vault.
- **Fail-Closed Validation** : L'API de rotation rejette toute requête ne respectant pas strictement le schéma Pydantic.
- **Routage Contextuel** : Capacité à distinguer et traiter des secrets de même type (ex: `api_key`) en fonction de leur dépôt d'origine.
- **Interopérabilité Sécurisée** : Communication fluide et chiffrée entre un environnement Windows (Dev) et Linux (SecOps) via Tailscale.

---

## 📚 Documentation Détaillée

Pour des informations techniques spécifiques à chaque composant, veuillez consulter les README individuels :
- [Documentation Infrastructure & Vault](https://github.com/taskflow-pfa-2026/repo-infra/README.md)
- [Documentation Backend & API](https://github.com/taskflow-pfa-2026/repo-backend/README.md)
- [Documentation Frontend](https://github.com/taskflow-pfa-2026/repo-frontend/README.md)
- [Documentation Rotation API](https://github.com/taskflow-pfa-2026/repo-secrets-mgmt/README.md)
- [Documentation Détection & SIEM (Volet B)](https://github.com/taskflow-pfa-2026/repo-security/README.md)

---

> **⚠️ Note de Sécurité** : Ce dépôt contient des configurations pour un environnement de démonstration. Les fichiers `.env`, `vault-keys.json` et les tokens de démonstration ne doivent **jamais** être poussés vers des dépôts publics en production.
