# Rapport du Projet Final DevOps – LEGENI CONSULTING
**Auteur(s) :** Borel & Binôme  
**Date :** Novembre 2026  
**Application :** Legeni WebApp (Flask + Redis)

---

## 1. Architecture Générale de la Chaîne CI/CD

```
  [Développeur] 
       │
   git push / PR
       ▼
   [GitHub] ── (Webhook) ──▶ [Jenkins]
                                │
   ┌────────────────────────────┴─────────────────────────────┐
   │                                                          │
   ▼                                                          ▼
[Étape CI - Tests en parallèle]                  [Étape CD - Ansible + K8s]
  • pytest + couverture (≥ 80%)                    • Staging : legeni-staging
  • flake8 + hadolint                              • Smoke tests automatiques
  • Syntaxe Ansible                                • Approbation manuelle
  • Build Docker Multi-stage                       • Prod : legeni-prod (RollingUpdate)
  • Scan Trivy (CRITICAL)                          • Rollback auto en cas d'erreur
  • Publication Docker Hub
```

---

## 2. Détail des Lots Réalisés

### Lot 0 – Organisation Git
- Dépôt structuré avec protection de branche `main`.
- Versionnage sémantique stocké dans le fichier `VERSION` (ex: `1.0.0`).
- Chaque image produite est taggée avec `${VERSION}-${BUILD_NUMBER}` (jamais `latest` en production).

### Lot 1 – Conteneurisation (Docker)
- `Dockerfile` multi-stage :
  - **Stage `base`** : Installation des dépendances applicatives isolées en cache.
  - **Stage `test`** : Linting avec `flake8` et exécution des tests unitaires `pytest`.
  - **Stage `runtime`** : Image finale minimale, exécution sous utilisateur non-privilégié `appuser` (UID 10001), sonde `HEALTHCHECK` native et commande exec `gunicorn`.

### Lot 2 – Infrastructure as Code (Ansible)
- Playbook `ansible/provision.yml` orchestrant les rôles réutilisables :
  - `docker` : installation du moteur Docker et configuration des droits.
  - `jenkins` : instanciation du conteneur Jenkins avec montage du socket Docker.
  - `minikube` : démarrage du cluster local avec addons `ingress` et `metrics-server`.
- Idempotence complète garantie sur toutes les tâches.
- Chiffrement des mots de passe sensibles via **Ansible Vault** (`vars/vault.yml`).

### Lot 3 – Déploiement Kubernetes & Self-Healing
- Deux environnements cloisonnés par namespaces : `legeni-staging` et `legeni-prod`.
- **Redis** déployé en `StatefulSet` avec persistance `PersistentVolumeClaim` et mot de passe extrait d'un `Secret`.
- **Legeni WebApp** en `Deployment` :
  - Sondes `readinessProbe` (`/ready`) et `livenessProbe` (`/health`).
  - Stratégie `RollingUpdate` avec `maxUnavailable: 0` pour garantir un temps d'indisponibilité nul (zero-downtime).
- Playbook `ansible/deploy.yml` intégrant :
  - L'attente active de la disponibilité des réplicas.
  - Un **smoke test** interrogeant `/api/info` pour valider la version déployée.
  - Un bloc `rescue` réalisant un `kubectl rollout undo` automatique en cas d'échec du test.

### Lots 4 & 5 – Pipelines CI/CD Jenkins
- **Job Multibranch** configuré via `Jenkinsfile`.
- Contrôles qualité parallélisés (tests, couverture, Hadolint, syntaxe Ansible).
- Scan de vulnérabilités par `Trivy`.
- Déploiement continu progressif : **Staging automatique ➔ Validation humaine (input) ➔ Production**.
- Gestion stricte des identifiants (Credentials Jenkins pour Docker Hub, Kubeconfig et Vault).

---

## 3. Scénarios de Soutenance & Démonstration

1. **Livraison Nominale :**
   - Mise à jour de `VERSION` et commit sur branche `feature`.
   - Création de la Pull Request : déclenchement du build sans déploiement.
   - Merge sur `main` : validation des tests, scan Trivy, push Docker Hub, déploiement staging, validation humaine, déploiement prod avec boucle `curl` sans coupure.
2. **Gestion d'un Échec de Test :**
   - Injection volontaire d'une régression dans `tests/test_app.py`.
   - Échec immédiat au stage CI : le pipeline passe au rouge, aucune image n'est générée ni déployée.
3. **Rollback Automatique :**
   - Modification de la probe `/ready` vers un chemin invalide.
   - Constat de non-disponibilité du pod lors du smoke test Ansible ➔ déclenchement automatique du `kubectl rollout undo`.

