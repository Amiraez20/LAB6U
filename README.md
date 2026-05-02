# Sécurité Mobile — Analyse Statique d'un APK avec MobSF

> **Contexte légal :** Ce travail pratique s'inscrit exclusivement dans un cadre pédagogique encadré. Toute analyse doit porter uniquement sur le fichier APK remis par le formateur ou compilé depuis un projet personnel de formation. Analyser une application tierce sans autorisation écrite constitue une infraction. L'objectif est uniquement défensif : comprendre les mécanismes de sécurité mobile, jamais les contourner.

---

## Prérequis techniques

- Machine virtuelle **Mobexler** installée, démarrée et accessible
- Fichier APK pédagogique fourni par le formateur (ou généré en mode debug depuis un projet personnel)
- Accès à un navigateur web à l'intérieur de la VM

---

## Présentation du TP

Ce travail pratique initie à l'**analyse statique d'applications Android** via **MobSF** (*Mobile Security Framework*), un outil open-source d'audit automatisé. L'ensemble des manipulations se déroule dans l'environnement isolé de la VM, ce qui garantit qu'aucune donnée réelle n'est exposée.

L'analyse statique consiste à examiner une application *sans l'exécuter* : on inspecte le code compilé, le fichier de configuration, les ressources et les métadonnées pour détecter des failles potentielles.

---

## Ce que vous allez apprendre

- Organiser un environnement de travail d'audit reproductible
- Soumettre un APK à une analyse automatisée et interpréter les résultats
- Lire un manifeste Android pour repérer des configurations risquées
- Évaluer la politique de sécurité réseau d'une application mobile
- Détecter des secrets ou ressources sensibles exposés dans le code
- Relier les failles découvertes aux contrôles du référentiel **OWASP MASVS**
- Produire un rapport d'audit structuré, clair et exploitable

---

## Lexique

| Terme | Définition |
|---|---|
| **APK** | Archive Android contenant le code, les ressources et la signature de l'application |
| **DEX** | Format de bytecode exécuté par la machine virtuelle Android |
| **AndroidManifest.xml** | Fichier de déclaration central d'une application (composants, permissions, configuration) |
| **Certificat / Signature** | Mécanisme cryptographique garantissant l'authenticité et l'intégrité de l'APK |
| **Permission** | Droit d'accès qu'une application sollicite auprès du système Android |
| **Exported** | Attribut autorisant d'autres applications à interagir avec un composant |
| **Triage** | Classement des vulnérabilités selon leur criticité réelle |
| **Faux positif** | Alerte générée à tort par l'outil d'analyse automatisée |
| **MASVS** | Standard OWASP définissant les exigences de sécurité des applications mobiles |

---

## Étape 1 — Mise en place de l'espace de travail (≈ 10 min)

**But :** Créer un répertoire organisé et tracer les informations essentielles avant toute manipulation.

**Pourquoi ?** La traçabilité est une exigence fondamentale en audit de sécurité. Elle garantit la reproductibilité de l'analyse et constitue la base d'un rapport crédible.

### Procédure

**1. Démarrer la VM et ouvrir un terminal**

Démarrer Mobexler depuis le gestionnaire de virtualisation, attendre l'affichage complet du bureau, puis ouvrir un terminal (`clic droit > Ouvrir un terminal`).

**2. Créer le répertoire de travail daté**

```bash
mkdir -p ~/travaux_securite/$(date +%Y-%m-%d)
cd ~/travaux_securite/$(date +%Y-%m-%d)
```

**3. Déplacer l'APK dans ce répertoire**

```bash
mv ~/Téléchargements/application-debug.apk ./
```

**4. Calculer l'empreinte SHA-256 de l'APK**

```bash
sha256sum application-debug.apk | tee empreinte_apk.txt
```

**5. Vérifier la taille du fichier**

```bash
ls -lh application-debug.apk
```

**6. Créer le journal de session**

```bash
cat > journal_session.txt << EOF
Date           : $(date)
Analyste       : [Votre prénom et nom]
VM             : Mobexler $(cat /etc/mobexler-version 2>/dev/null || echo "version inconnue")
Fichier analysé: application-debug.apk
EOF
cat empreinte_apk.txt >> journal_session.txt
```

### Points de contrôle

- [ ] Répertoire daté créé
- [ ] APK présent dans le répertoire
- [ ] Empreinte SHA-256 consignée
- [ ] Journal de session initialisé

### Erreurs fréquentes

- Oublier de noter la provenance exacte de l'APK
- Travailler directement dans le dossier de téléchargement (risque d'écrasement)

---

## Étape 2 — Démarrage de MobSF (≈ 5-10 min)

**But :** Lancer le serveur MobSF et vérifier l'accessibilité de l'interface d'analyse.

**Pourquoi ?** MobSF centralise de nombreuses vérifications de sécurité automatisées conformes aux standards OWASP MASVS et aux bonnes pratiques Android.

### Procédure

**1. Lancer le serveur MobSF**

Ouvrir un *nouveau* terminal (conserver le précédent ouvert) et exécuter :

```bash
cd ~/outils/Mobile-Security-Framework-MobSF
./run.sh 127.0.0.1:8000
```

Attendre les messages confirmant le démarrage (`Starting MobSF` puis `Server is running`). **Ne pas fermer ce terminal.**

**2. Ouvrir l'interface web**

Lancer Firefox et naviguer vers : `http://127.0.0.1:8000`

**3. Vérifier l'interface**

S'assurer que la page d'accueil affiche le formulaire d'envoi de fichier et les menus principaux (*Static Analysis*, *Dynamic Analysis*, *API Tester*).

**4. Enregistrer la version de MobSF**

```bash
echo "MobSF version : [version affichée en bas de page]" >> ~/travaux_securite/$(date +%Y-%m-%d)/journal_session.txt
```

### Points de contrôle

- [ ] Serveur MobSF actif sans erreur
- [ ] Interface accessible sur `http://127.0.0.1:8000`
- [ ] Version de MobSF notée dans le journal

### Problèmes courants

- Port déjà occupé → vérifier avec `ss -tlnp | grep 8000` et arrêter le processus conflictuel
- Dépendances Python manquantes → relire le README de MobSF et relancer l'installation

---

## Étape 3 — Soumission de l'APK à MobSF (≈ 10-15 min)

**But :** Importer l'APK et déclencher l'analyse statique complète.

**Pourquoi ?** L'analyse statique examine le code, les ressources et la configuration sans exécuter l'application, révélant des failles que les tests fonctionnels classiques ne détectent pas.

### Procédure

**1. Importer l'APK**

Dans l'interface MobSF, cliquer sur *Upload & Analyze*, sélectionner `application-debug.apk` dans le répertoire de travail, puis valider.

**2. Surveiller la progression**

```bash
echo "Début analyse : $(date)" >> ~/travaux_securite/$(date +%Y-%m-%d)/journal_session.txt
```

Ne pas interrompre le navigateur. L'analyse dure entre 2 et 10 minutes selon la taille de l'APK.

**3. Explorer la page de résultats**

Une fois l'analyse terminée, MobSF redirige automatiquement vers le rapport. Repérer :
- Le **score de sécurité global** (affiché en haut)
- Les catégories : *Info*, *Warning*, *Critical*
- Les sections disponibles dans le menu de navigation

**4. Consigner le score**

```bash
echo "Fin analyse    : $(date)" >> ~/travaux_securite/$(date +%Y-%m-%d)/journal_session.txt
echo "Score global   : [score affiché]" >> ~/travaux_securite/$(date +%Y-%m-%d)/journal_session.txt
```

### Points de contrôle

- [ ] APK importé sans erreur
- [ ] Analyse terminée avec succès
- [ ] Page de résultats accessible
- [ ] Score de sécurité noté

---

## Étape 4 — Lecture du manifeste et des permissions (≈ 15-20 min)

**But :** Identifier les configurations et permissions problématiques déclarées dans le manifeste Android.

**Pourquoi ?** Le manifeste est le premier point d'entrée de toute analyse de sécurité Android : il expose la surface d'attaque de l'application (composants accessibles, droits demandés, options de débogage).

### Procédure

**1. Consulter la section *App Information***

Relever :
- Identifiant de package (ex. `fr.formation.monapp`)
- Version de l'application
- Niveaux d'API minimum et cible
- Présence de `android:debuggable="true"` (danger en production)
- Valeur de `android:allowBackup`

**2. Analyser la section *Manifest Analysis***

Observer les avertissements signalés par MobSF et explorer le manifeste décompilé.

**3. Trier les permissions**

```bash
cat > permissions_identifiees.txt << EOF
=== Permissions dangereuses ===
[Lister ici les permissions de la catégorie DANGEROUS]

=== Permissions normales ===
[Lister ici les permissions de la catégorie NORMAL]
EOF
```

**4. Recenser les composants exposés**

```bash
cat > composants_exposes.txt << EOF
=== Activités exportées ===
[Lister les Activity avec exported=true ou intent-filter implicite]

=== Services exportés ===
[Lister les Service avec exported=true]

=== Broadcast Receivers exportés ===
[Lister les BroadcastReceiver avec exported=true]

=== Content Providers exportés ===
[Lister les ContentProvider avec exported=true]
EOF
```

### Éléments à surveiller

| Attribut | Valeur risquée | Impact |
|---|---|---|
| `android:debuggable` | `true` | Accès au débogage en production |
| `android:allowBackup` | `true` | Extraction des données via ADB |
| `android:usesCleartextTraffic` | `true` | Trafic HTTP non chiffré autorisé |
| `android:exported` | `true` sans protection | Composant accessible par n'importe quelle appli |

### Points de contrôle

- [ ] Permissions dangereuses listées et justifiées
- [ ] Composants exposés recensés
- [ ] Configurations sensibles documentées

### Erreur fréquente

Ne pas oublier les composants **implicitement exportés** via un `<intent-filter>` sans `android:exported="false"` explicite (comportement par défaut selon la version d'API cible).

---

## Étape 5 — Audit de la configuration réseau (≈ 15 min)

**But :** Évaluer si l'application protège correctement ses échanges réseau.

**Pourquoi ?** Une configuration réseau laxiste peut permettre l'interception de données sensibles (identifiants, tokens, informations personnelles) via une attaque man-in-the-middle.

### Procédure

**1. Rechercher le fichier de politique réseau**

Dans la section *Files*, rechercher `network_security_config.xml`. Si absent, noter cette lacune.

**2. Vérifier les attributs critiques du manifeste**

Retourner dans *Manifest Analysis* et repérer :
- `android:networkSecurityConfig` (référence au fichier de politique)
- `android:usesCleartextTraffic="true"` (HTTP autorisé)

**3. Documenter la configuration**

```bash
cat > config_reseau.txt << EOF
=== Configuration réseau ===
Fichier network_security_config.xml : [présent / absent]
usesCleartextTraffic               : [valeur ou non défini]
Domaines de confiance définis      : [oui / non]
Certificats personnalisés          : [oui / non]
Overrides de débogage              : [oui / non]
EOF
```

**4. Recenser les endpoints identifiés**

Dans la section *Strings* ou *Files*, rechercher des URLs (préfixes `http://`, `https://`) :

```bash
cat > endpoints_identifies.txt << EOF
=== Endpoints détectés ===
[URL 1]
[URL 2]
...
EOF
```

### Points de contrôle

- [ ] Présence/absence d'une politique réseau vérifiée
- [ ] Autorisation de trafic HTTP évaluée
- [ ] Endpoints hardcodés recensés
- [ ] Risques TLS documentés

---

## Étape 6 — Inspection du code et des ressources (≈ 20-25 min)

**But :** Détecter les secrets exposés, les mauvaises pratiques de codage et les ressources sensibles.

**Pourquoi ?** Le code source et les ressources embarquées sont fréquemment source de fuites d'informations (clés API, tokens d'authentification, URLs internes).

### Procédure

**1. Explorer la section *Code Analysis***

Observer :
- Les catégories de vulnérabilités détectées
- Le nombre d'alertes par niveau de sévérité
- Les descriptions et recommandations associées à chaque alerte

**2. Inspecter la section *Hardcoded Secrets***

Pour chaque secret potentiel détecté, évaluer son contexte réel (vraie clé vs valeur de test).

**3. Analyser *URLs and Emails***

Repérer les environnements exposés (développement, recette, production) et les endpoints sensibles.

**4. Documenter les découvertes critiques**

```bash
cat > synthese_code.txt << EOF
=== Secrets potentiels détectés ===
[Décrire chaque secret : type, emplacement, contexte]

=== Vulnérabilités de code critiques ===
[Décrire chaque vulnérabilité : description, fichier/classe concerné]

=== Ressources sensibles ===
[Fichiers de configuration, bases de données locales, assets sensibles]
EOF
```

### Éléments particulièrement à surveiller

- Clés API ou tokens d'authentification en clair dans `strings.xml` ou le code
- URLs d'environnements internes (développement, staging)
- Appels à des API cryptographiques non sécurisées
- Logs contenant des données sensibles (`Log.d`, `Log.e`)
- Mode debug activé (`BuildConfig.DEBUG` toujours vrai)

### Points de contrôle

- [ ] Secrets hardcodés identifiés et contextualisés
- [ ] Vulnérabilités de code priorisées
- [ ] Ressources sensibles listées
- [ ] Faux positifs potentiels notés

---

## Étape 7 — Mise en correspondance avec OWASP MASVS (≈ 15-20 min)

**But :** Relier chaque vulnérabilité confirmée à une exigence du référentiel OWASP MASVS.

**Pourquoi ?** Référencer les failles selon un standard reconnu renforce la crédibilité du rapport et facilite la priorisation des corrections par les équipes de développement.

### Ressources à consulter

- MASVS (exigences) : [https://mas.owasp.org/MASVS/](https://mas.owasp.org/MASVS/)
- MASTG (procédures de test) : [https://mas.owasp.org/MASTG/](https://mas.owasp.org/MASTG/)

### Procédure

**1. Se familiariser avec les catégories MASVS**

Les principales catégories à connaître pour ce TP :

| Catégorie | Thème |
|---|---|
| MASVS-STORAGE | Stockage local des données |
| MASVS-CRYPTO | Pratiques cryptographiques |
| MASVS-NETWORK | Sécurité des communications |
| MASVS-AUTH | Authentification |
| MASVS-CODE | Qualité et robustesse du code |

**2. Associer chaque vulnérabilité à une référence MASVS**

```bash
cat > correspondances_masvs.txt << EOF
=== Vulnérabilité 1 : [titre descriptif] ===
Référence MASVS  : MASVS-[CATEGORIE]-[N]
Exigence         : [Copier ici l'intitulé exact de l'exigence]
Non-conformité   : [Décrire comment l'application ne respecte pas cette exigence]
Localisation     : [Fichier ou classe concerné]
Impact           : [Conséquences potentielles]

=== Vulnérabilité 2 : [titre descriptif] ===
[Même structure]

=== Tests MASTG complémentaires ===
Test 1 : [Référence et description du test]
Test 2 : [Référence et description du test]
EOF
```

### Points de contrôle

- [ ] Au moins 2 vulnérabilités associées à des références MASVS valides
- [ ] Preuves de non-conformité documentées pour chaque référence
- [ ] 2 tests MASTG complémentaires identifiés

### Erreurs fréquentes

- Associer une faille à une catégorie MASVS inexacte (lire attentivement l'intitulé de l'exigence)
- Manquer de preuves concrètes (mentionner précisément le fichier et la ligne concernés)

---

## Étape 8 — Export et triage du rapport MobSF (≈ 10-15 min)

**But :** Extraire le rapport complet, identifier les problèmes prioritaires et distinguer vrais positifs et faux positifs.

**Pourquoi ?** L'analyse automatisée produit du bruit. Le triage humain est indispensable pour distinguer les risques réels des alertes non pertinentes dans le contexte de l'application.

### Procédure

**1. Exporter le rapport**

Dans l'interface MobSF, cliquer sur *Generate PDF Report* (ou *JSON Report* pour l'exploitation automatisée).

```bash
mv ~/Téléchargements/MobSF_Report*.pdf ~/travaux_securite/$(date +%Y-%m-%d)/rapport_mobsf_$(date +%Y%m%d).pdf
echo "Rapport exporté : rapport_mobsf_$(date +%Y%m%d).pdf" >> ~/travaux_securite/$(date +%Y-%m-%d)/journal_session.txt
```

**2. Parcourir le rapport exporté**

Ouvrir le PDF et repérer la structure des sections.

**3. Prioriser les vulnérabilités**

```bash
cat > classement_vulnerabilites.txt << EOF
=== CRITIQUE ===
[Lister les vulnérabilités de niveau critique avec description et localisation]

=== ÉLEVÉE ===
[Lister les vulnérabilités de niveau élevé]

=== MOYENNE ===
[Lister les vulnérabilités de niveau moyen]

=== FAIBLE / INFORMATIF ===
[Lister les problèmes mineurs]

=== FAUX POSITIFS IDENTIFIÉS ===
[Alerte] : [Justification du faux positif]
EOF
```

### Grille de sévérité

| Niveau | Critères |
|---|---|
| **Critique** | Exploitable facilement, impact majeur sur la confidentialité ou l'intégrité |
| **Élevée** | Compromission potentielle de données sensibles, exploitation réalisable |
| **Moyenne** | Impact limité ou exploitation nécessitant un accès préalable |
| **Faible** | Risque théorique, impact négligeable dans ce contexte |

### Points de contrôle

- [ ] Rapport PDF exporté et archivé
- [ ] Vulnérabilités classées par niveau de sévérité
- [ ] Faux positifs identifiés et justifiés

---

## Étape 9 — Rédaction du rapport d'audit (≈ 20-30 min)

**But :** Consolider l'ensemble des observations dans un document professionnel, clair et exploitable.

**Pourquoi ?** Un bon rapport de sécurité doit permettre à une équipe de développement — même sans expertise en sécurité — de comprendre les risques et de mettre en œuvre les corrections.

### Procédure

```bash
touch rapport_audit_final.md
```

### Structure recommandée

```markdown
# Rapport d'Audit de Sécurité — [Nom de l'application]

## Informations de session
| Champ | Valeur |
|---|---|
| Date | [date] |
| Analyste | [nom] |
| Fichier audité | application-debug.apk |
| SHA-256 | [hash] |
| Outil principal | MobSF v[version] — VM Mobexler v[version] |

## Synthèse exécutive
[3 à 5 phrases : niveau de risque global, principales catégories de problèmes, 
recommandation générale. Rédigé pour un lecteur non technique.]

## Vulnérabilités identifiées

### [VUL-01] Titre de la vulnérabilité
- **Sévérité :** Critique / Élevée / Moyenne / Faible
- **Référence MASVS :** MASVS-[CAT]-[N]
- **Description :** [Explication factuelle et concise]
- **Preuve :** [Nom du fichier, classe, ligne ou capture d'écran référencée]
- **Impact :** [Conséquences concrètes pour l'utilisateur ou le système]
- **Remédiation :** [Action précise et actionnable pour l'équipe de développement]

### [VUL-02] ...
[Même structure]

## Autres observations
[Liste des points de moindre importance ne justifiant pas une section complète]

## Plan d'action priorisé
1. [Action prioritaire 1 — à traiter immédiatement]
2. [Action prioritaire 2 — à planifier rapidement]
3. [Action prioritaire 3 — à intégrer dans la prochaine release]

## Annexes
### A. Liste complète des permissions
### B. Composants Android exposés
### C. Endpoints et domaines détectés
### D. Éléments considérés comme faux positifs
```

### Critères de qualité du rapport

| Critère | Description |
|---|---|
| **Clarté** | Compréhensible par un développeur sans expertise sécurité |
| **Preuves** | Chaque vulnérabilité est localisée précisément |
| **Actionnabilité** | Les remédiations sont concrètes et réalisables |
| **Priorisation** | Les problèmes les plus graves sont traités en premier |
| **Concision** | 1 à 2 pages de synthèse + annexes techniques |

### Points de contrôle

- [ ] Toutes les sections complétées
- [ ] 3 à 5 vulnérabilités documentées avec preuves
- [ ] Recommandations formulées de manière actionnable
- [ ] Mise en forme professionnelle et cohérente

---

## Résolution des problèmes courants

| Problème | Cause probable | Solution |
|---|---|---|
| MobSF ne démarre pas | Port 8000 occupé | `ss -tlnp \| grep 8000` puis arrêter le processus conflictuel |
| Erreur à l'import de l'APK | APK corrompu ou trop volumineux | Vérifier le SHA-256, essayer avec un APK plus petit |
| Analyse bloquée | Espace disque insuffisant | `df -h` pour vérifier, libérer de l'espace si nécessaire |
| Rapport PDF non généré | Permissions du répertoire | `chmod 755 ~/travaux_securite/` |
| Nombreux faux positifs | Analyse automatisée | Vérification manuelle obligatoire pour chaque alerte critique |

---

## Références

| Ressource | URL |
|---|---|
| MobSF (code source) | https://github.com/MobSF/Mobile-Security-Framework-MobSF |
| OWASP Mobile Application Security | https://mas.owasp.org/ |
| OWASP MASVS (exigences) | https://mas.owasp.org/MASVS/ |
| OWASP MASTG (tests) | https://mas.owasp.org/MASTG/ |
| Sécurité Android (officiel) | https://source.android.com/docs/security |
| Bonnes pratiques Android | https://developer.android.com/privacy-and-security/security-best-practices |
| Configuration réseau Android | https://developer.android.com/privacy-and-security/security-config |
