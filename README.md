# Activa HealthPass — App mobile (React Native / Expo)

Application mobile native (Android + iOS) pour les profils **Agent** et
**Superviseur**, construite avec Expo Router et connectée au backend
Google Apps Script existant (`Code_v3_3_0.gs`). Le profil **Admin** reste
volontairement sur le Control Panel desktop, conformément au design source.

## 1. Installer et lancer

```bash
npm install
npx expo start
```

Scanne le QR code avec l'app **Expo Go** (Android/iOS), ou lance un simulateur :

```bash
npx expo start --android
npx expo start --ios      # nécessite macOS
```

## 2. Connecter au backend Apps Script

0. **Important** : remplace `Code_v3_3_0.gs` par **`Code_v3_3_1.gs`** (fourni
   à côté de ce zip) dans ton projet Apps Script — c'est la même base, avec
   uniquement 2 nouveaux cas de routage additifs dans `doPost` (voir section 3,
   "Pré-autorisations"). Sans ça, l'écran Pré-autorisations renverra
   `UNKNOWN_ACTION`.
1. Dans le projet Apps Script, **Déployer → Nouveau déploiement**
   → type **Application Web** → *Exécuter en tant que* **Moi** → *Qui a accès*
   **Tout le monde**.
2. Copie l'URL `https://script.google.com/macros/s/AKfycb.../exec` obtenue.
3. Renseigne-la dans `.env` :
   ```
   EXPO_PUBLIC_HEALTHPASS_API_URL=https://script.google.com/macros/s/AKfycb.../exec
   ```
4. Redémarre `npx expo start` (les variables `EXPO_PUBLIC_*` sont lues au build).

## 3. État d'avancement

### Control Panel desktop v4 — refonte complète, 11 écrans
Le Control Panel desktop (servi par `doGet()` dans `Code_v3_3_1.gs`) a été
entièrement reconstruit pour correspondre au design v4 fourni : login Admin
unique (fusion Agent/Superviseur/Admin remplacée par un compte Google +
mot de passe applicatif), sidebar bleue, 11 écrans complets — **tous testés
de bout en bout avec Playwright, pas seulement écrits** :

| Écran | Backend |
|---|---|
| Tableau de bord | `getAdminDashboardData_()` (nouveau, assemble des fonctions existantes) |
| Sinistres | `supervisorListPendingClaims`, `validateClaim`, `adminListClaims` (existantes) |
| Factures (par patient / par famille) | `getInvoicesData_()` (nouveau, regroupe `adminListClaims`) |
| Enrôlements | `supervisorListPendingEnrollments/Approve/Reject`, `adminListEnrollments_()` (nouveau) |
| Rapports | `getReportsScreenData_()` (nouveau, assemble des fonctions existantes) |
| Membres assurés | `getMembersScreenData_()` (nouveau), `adminSaveMember`/`adminDeleteMember` (existantes) |
| Organisations | `adminListOrganisations/Save/Delete` (existantes) |
| Prestataires | `adminListProviders_/Save/Delete` (3 nouvelles — n'existaient pas encore) |
| Plafonds | `adminListOrgCeilings`, `adminSaveOrgCeiling` (existantes) |
| Comptes | `adminListAccounts/Save/Delete` (existantes) |
| Log de connexion | `getAccountsConnectionLog_()`, `adminListSystemLogs_()` (nouvelles) |

**Approche non destructive de bout en bout** : tout est dans une nouvelle
fonction `getHtmlV2_()` — l'ancienne `getHtml()` (le portail vert à 3 rôles
d'origine) reste intacte, inchangée, juste plus appelée. Un seul mot à
changer dans `doGet()` pour revenir en arrière si besoin. Le logo est le
même que celui de l'app mobile (extrait de la maquette, encodé en data URI).

**Écarts avec la maquette v4, corrigés délibérément à chaque fois que le
design proposait un champ ou une structure qui n'existe pas dans les
données réelles** — documentés en détail dans le code et les échanges :
- **Organisations / Prestataires** : pas de "Type de contrat" fictif — les
  vrais champs du backend (police, dates, effectif déclaré / statut KYP)
- **Comptes** : pas de rôle unique Agent/Superviseur/Admin — les vrais
  rôles cumulables (Admin, Controller) du système d'accès à CE panneau,
  distinct des comptes Agent terrain (qui n'y accèdent jamais)
- **Log de connexion** : le "journal technique" est nommé pour ce qu'il est
  réellement (erreurs/incidents captés par `logError_`), pas présenté comme
  un audit métier complet qui n'existe pas dans le système actuel

**Deux bugs réels trouvés et corrigés en testant**, pas en relisant le
code : un échappement `\n`/`\r` mal doublé qui cassait la syntaxe du script
client généré (invisible en ne vérifiant que le `.gs`), et un message de
confirmation "Plafond enregistré" écrasé instantanément par un rechargement
de données mal séquencé.

### Import Excel (Membres, Organisations, Prestataires) — corrigé
Initialement signalé comme hors périmètre (nécessitait un parseur xlsx —
Apps Script n'en a pas de natif). En relisant le fichier, une fonction
complète et déjà éprouvée existait déjà pour les Membres
(`adminImportMembersXlsx`, via conversion temporaire Drive xlsx→Google
Sheet) — je n'ai eu qu'à câbler l'interface dessus. Pour Organisations et
Prestataires, deux nouvelles fonctions (`adminImportOrganisationsXlsx_`,
`adminImportProvidersXlsx_`) réutilisent le même mécanisme de conversion et
appellent ligne par ligne les fonctions de sauvegarde déjà existantes et
déjà validées (`adminSaveOrganisation`, `adminSaveProvider_`) — pas de
logique de validation dupliquée. Testé de bout en bout (upload réel via
Playwright, bannière de résultat "X créé(s), Y mis à jour, Z ignoré(s)").

**Prérequis technique, à vérifier une fois** : ce mécanisme nécessite le
service avancé **Drive API** activé dans le projet Apps Script (éditeur →
Services → `+` → Drive API). Il était déjà requis par l'import Membres
existant avant mes ajouts — si l'import Membres fonctionnait déjà chez toi,
rien à faire. Sinon, l'écran affiche un message clair
("Le service Drive avancé n'est pas activé…") plutôt que d'échouer
silencieusement.

### Application mobile
- **Connexion Agent** (téléphone + mot de passe) → `agentLogin`
- **Connexion Superviseur** (Google Sign-In) → `supervisorLogin`
- **Identifier membre** → `searchMember` (statut, plafonds ambulatoire/hospitalisation)
- **Enrôler un membre** (Agent) → `submitEnrollment`
- **Fiches** (Agent) → `listMyFiches` (historique) + `generateFiche` (génération réelle, voir section dédiée ci-dessous)
- **Prestations** (Agent) → `agentSubmitClaim` (nouvelle fonction, voir section 3)
- **Superviseur — Accueil** (compteur réel), **À valider** (liste, approuver, rejeter
  avec motif — reçoit aussi les prestations soumises par les agents),
  **Enrôlements** (liste, approuver, rejeter avec motif),
  **Historique** (décisions passées), **Rapports** (stats, répartition par
  organisation/mois) → `supervisorListPendingClaims`, `validateClaim`,
  `supervisorListPendingEnrollments`, `supervisorApproveEnrollment`,
  `supervisorRejectEnrollment`, `supervisorListMyHistory`, `getReportsData`
- **Profil** Agent & Superviseur → déconnexion réelle (vide la session sécurisée)

### Fiches — génération réelle du PDF, corrigée
Vérification demandée par l'utilisateur : la fiche maladie est-elle bien
générée en PDF ? **Côté backend, oui, sans ambiguïté** —
`generateFicheMaladiePdf` construit un Google Doc structuré (tableaux,
mise en forme officielle, code de sécurité) puis l'exporte via
`DriveApp.getFileById(doc.getId()).getAs("application/pdf")`, le
mécanisme de conversion standard et fiable de Google — pas un
placeholder.

**Mais en vérifiant côté mobile, un vrai trou est apparu** : la fonction
cliente `generateFiche()` existait déjà dans `src/api/client.ts`, mais
n'était appelée **nulle part** dans l'app — l'écran "Fiches" se contentait
de lister l'historique (`listMyFiches`), sans aucun moyen de déclencher une
nouvelle génération. Corrigé : l'écran `app/(agent)/fiches.tsx` propose
maintenant un vrai formulaire (numéro de carte, sélection du prestataire,
type de traitement Ambulatoire/Hospitalisation), appelle `generateFiche`,
puis :
1. écrit le PDF reçu (base64) dans le cache local de l'appareil via la
   nouvelle API `File`/`Paths` d'`expo-file-system` (SDK 57) ;
2. ouvre le sélecteur de partage natif via `expo-sharing`
   (`Sharing.shareAsync`) — l'agent peut l'ouvrir, l'imprimer, ou l'envoyer
   directement (WhatsApp, e-mail…) sans étape intermédiaire.

Deux nouvelles dépendances ajoutées : `expo-file-system`, `expo-sharing`
(aucune configuration de plugin `app.json` requise, autolinking standard).

**Limite de validation à signaler honnêtement** : ce flux (écriture de
fichier + partage natif) ne peut être testé que sur un vrai appareil ou
simulateur — les API `File`/`Sharing` sont natives et ne s'exécutent pas
dans ce sandbox de développement. `npx tsc --noEmit` et le build web
passent (0 erreur), confirmant l'absence d'erreur de code, mais le flux
complet (génération → partage) reste à valider par toi sur un appareil réel
connecté à ton déploiement Apps Script.

### Audit complet : fonctions backend jamais câblées côté mobile
Suite à la découverte du trou sur les Fiches, audit systématique de
**toutes** les fonctions exportées de `src/api/client.ts` et de **toutes**
les actions `doPost` du backend, pour repérer tout autre écart entre "le
client existe" et "le client est réellement appelé quelque part".

**Trouvé et corrigé** : `requestOtp`/`verifyOtp` — un système complet de
vérification d'identité par code à 6 chiffres (SMS/WhatsApp/E-mail, avec
anti-bruteforce et verrouillage après échecs répétés côté backend) n'était
appelé nulle part. Ajouté dans `app/(agent)/identify.tsx` : une fois le
membre trouvé, l'agent peut envoyer un code, le saisir, et voir "Identité
vérifiée" confirmé avant de poursuivre.

**Trouvé, volontairement pas câblé** : un système complet de
pré-autorisation d'hospitalisation (`createHospitalAuthorization`,
`listHospitalAuthorizationsByAgent`, `listHospitalAuthorizations`,
`decideHospitalAuthorization`) existe côté backend — vérification
automatique du plafond, blocage + alerte e-mail au Superviseur si
insuffisant, génération d'un code d'autorisation à l'approbation — mais
n'a **aucune place dans la maquette mobile d'origine** (navigation à 6
onglets par rôle, sans écran "Pré-autorisation"). Confirmé obsolète par
l'utilisateur — laissé tel quel côté backend, non câblé côté mobile.

**Non fonctionnel, sans impact** : `src/components/PlaceholderScreen.tsx`
est un composant mort (jamais importé nulle part) — aucun écran actif
n'affiche de contenu factice. Quelques imports `React`/`useEffect` inutilisés
repérés via `--noUnusedLocals`, tous cosmétiques (nouveau transform JSX
automatique, ou logique de rafraîchissement déjà gérée par
`useFocusEffect`) — aucun ne correspond à une fonctionnalité manquante.


### Écran "Prestations" — aligné sur l'interface mobile, backend étendu en conséquence
En câblant cet écran une première fois, une lecture du backend avait révélé
un commentaire suggérant que la soumission de prestations avait été retirée
de l'outil Agent (remplacée à l'époque par les pré-autorisations
d'hospitalisation). **Décision finale (tranchée explicitement avec
l'utilisateur) : l'interface mobile fait foi.** L'onglet "Prestations" et le
flux "Ajouter une prestation" → "+ Ajouter à la liste" → "Soumettre pour
validation" → "Statut des soumissions" sont conservés tels que dessinés dans
la maquette d'origine, et c'est le **backend qui a été étendu** pour les
supporter — pas l'inverse.

`app/(agent)/prestations.tsx` est câblé sur une nouvelle fonction
`agentSubmitClaim(payload)`, ajoutée de façon strictement additive dans
**`Code_v3_3_1.gs`** (fourni à côté de ce zip — voir `backend/`). Chaque
prestation ajoutée à la liste locale devient, à la soumission, une ligne
individuelle dans le registre Sinistres (`SHEET_CLAIMS`), au statut
`"Pending Review"` — donc visible immédiatement dans l'écran "À valider" du
Superviseur, **déjà existant et déjà câblé**, sans dupliquer aucune logique
de validation.

Remplace `Code_v3_3_0.gs` par `Code_v3_3_1.gs` dans ton projet Apps Script
pour que cet écran fonctionne (voir section 2, étape 0).

### Solde en temps réel, historique mensuel, doublons et récurrence (itération 2)
Quatre demandes supplémentaires, toutes câblées :

1. **Solde disponible / solde restant après validation** (écran Prestations,
   fidèle à la maquette) — `prestations.tsx` appelle `searchMember` au blur
   du champ "Numéro de carte" et affiche le solde ambulatoire ; le "solde
   restant" se recalcule en direct à chaque ajout/retrait dans la liste
   locale, avant même la soumission au serveur.
2. **Historique des prestations du mois en cours** (écran Identification,
   fidèle à la maquette) — nouveau champ `monthlyHistory` ajouté de façon
   additive au retour de `searchMember()` (aucun champ existant modifié),
   calculé par la nouvelle fonction `getMemberMonthlyHistory_(cardNumber)`.
3. **Blocage systématique des doublons** — `agentSubmitClaim` réutilise le
   système de détection déjà existant côté backend
   (`checkDuplicateStrict/Cross/Soft`, jusque-là seulement branché sur
   l'ancien tunnel desktop) : tout doublon **STRICT** (même membre + même
   prestataire + même description, déjà soumis aujourd'hui) bloque
   **l'intégralité de la soumission avant toute écriture**.
4. **Alertes Superviseur — doublons et récurrence** — les doublons non
   bloquants (CROSS/SOFT) et les prestations récurrentes (> 2 pour le même
   membre sur le mois en cours) déclenchent un e-mail au superviseur
   (réutilise `CONTROLLER_EMAIL`, canal déjà utilisé par le reste de
   l'application) **et** sont désormais visibles directement dans l'app :
   `supervisorListPendingClaims()` expose un nouveau champ `dupFlag`
   (additif), affiché comme bannière d'alerte rouge dans le détail d'une
   réclamation et comme badge "⚠ Alerte" dans la liste "À valider" côté
   mobile.

**Note d'implémentation honnête** : l'écran Prestations mobile ne propose
qu'un champ "Prestataire" en texte libre (pas de sélection dans la liste KYP
Providers comme sur le desktop). `resolveProviderCode_()` tente une
correspondance par nom ; à défaut, utilise le nom normalisé comme clé de
comparaison pour la détection de doublon — cohérent d'une soumission à
l'autre pour un même établissement, mais moins strict qu'une vraie
sélection par code KYP. À améliorer si des doublons inter-orthographes
("JFK Medical Center" vs "JFK Medical Ctr") s'avèrent un problème réel en
usage.

### Sélection du prestataire (liste déroulante, remplace le texte libre)
Le champ "Prestataire" de l'écran Prestations est désormais une **vraie
liste déroulante** (modale avec recherche, `SelectField` dans
`src/components/ui.tsx`), alimentée par `getProviders()` — la même liste de
prestataires KYP validés déjà utilisée côté desktop (Identification / Fiche
Maladie), maintenant aussi routée via `doPost` (nouveau cas additif
`"getProviders"`). L'agent choisit un établissement dans la liste ; le code
KYP officiel est envoyé au serveur (`providerCode`) plutôt qu'un nom tapé à
la main.

`agentSubmitClaim` a été ajusté en conséquence (rétrocompatible) : s'il
reçoit un `providerCode` explicite, il l'utilise directement pour la
détection de doublon (comparaison par code officiel, fiable) ; sinon il
retombe sur l'ancienne résolution par nom (`resolveProviderCode_`), donc
aucune régression si un client mobile plus ancien continue d'envoyer
`hospitalName` seul. La note d'avertissement ci-dessus sur les doublons
inter-orthographes ne s'applique donc plus au flux normal — seulement au
cas de repli.

### Corrections de contrat importantes (v2)
Une première passe de câblage avait deviné certaines formes de réponse sans
les vérifier ligne à ligne dans `Code_v3_3_0.gs`. Une relecture complète du
`doPost` a corrigé plusieurs écarts réels :
- `supervisorListPendingClaims`, `supervisorListPendingEnrollments`,
  `supervisorListMyHistory`, `listMyFiches` renvoient un **tableau brut**, pas
  `{ success, items }` — `src/api/client.ts` expose désormais `isErrorShape()`
  pour distinguer proprement succès (tableau) et échec (`{success:false}`).
- `validateClaim` attend `decision: "Settled" | "Rejected"` (pas `"approve"/"reject"`).
- Les champs réels des réclamations sont `refCode`, `member`, `organisation`,
  `provider`, `ceiling`, `consumed` (pas `ref`, `name`, `org`, `before`).
- `submitEnrollment` attend `lastName`, `firstName`, `dob`, `relationship`,
  `organisation` séparés (pas `fullName`).
- `generateFiche` attend `hospitalName` (pas `hospital`) et renvoie
  `pdfBase64`/`fileName` (pas `pdfUrl`).
- `supervisorLogin` renvoie `fullName` (pas `name`).

### Bloquant connu : connexion Superviseur → configuration Google Cloud requise
Le code est **entièrement câblé** (`src/auth/googleAuth.ts` + `app/auth/index.tsx`
→ `supervisorLogin`) et validé : le bouton "Continuer avec Google" ouvre
correctement le flux OAuth Google avec les bons paramètres
(`response_type=id_token`, `nonce`, `scope=openid email profile`). Il ne
manque qu'une configuration côté Google Cloud Console — voir section 2bis
ci-dessous. Sans cette configuration, l'app affiche un message clair au lieu
de planter.

## 2bis. Configurer Google Sign-In (Superviseur)

Le backend (`verifyGoogleIdToken_` dans `Code_v3_3_0.gs`) vérifie que
l'audience du jeton Google correspond **exactement** à la propriété de script
`GOOGLE_OAUTH_CLIENT_ID` déjà configurée (le même client que le Control Panel
desktop). Il faut donc réutiliser ce même client, pas en créer un nouveau :

1. Dans [Google Cloud Console](https://console.cloud.google.com/apis/credentials),
   ouvre le projet lié à `GOOGLE_OAUTH_CLIENT_ID` et repère ce client OAuth
   **"Application Web"** existant.
2. Ajoute dans ses **URI de redirection autorisés** :
   - `http://localhost:8081` (test web local via `expo start --web`)
   - L'URI Expo Go affichée dans le terminal au premier clic sur "Continuer
     avec Google" pendant `npx expo start` (commence par `https://auth.expo.io/...`
     ou `exp://...` selon la version d'Expo Go)
   - `healthpass://` (build natif autonome / TestFlight / Play Console)
3. Copie l'ID client (`....apps.googleusercontent.com`) dans `.env` :
   ```
   EXPO_PUBLIC_GOOGLE_WEB_CLIENT_ID=....apps.googleusercontent.com
   ```
4. Redémarre `npx expo start`. Le compte Google utilisé pour se connecter doit
   correspondre à une ligne **Active** avec le rôle **Controller** dans la
   feuille des administrateurs (`ensureAdminsSheet()`), exactement comme pour
   le Control Panel desktop.

## 4. Structure

```
app/
  _layout.tsx          Layout racine (SessionProvider, Stack)
  index.tsx             Redirection selon session (login / agent / superviseur)
  auth/index.tsx         Écran de connexion (switch Agent/Superviseur)
  (agent)/               Tabs Agent : accueil, enrôlement, identifier, fiches,
                          prestations, profil
  (superviseur)/          Tabs Superviseur : accueil, à valider, enrôlements,
                          historique, rapports, profil
src/
  api/client.ts           Client HTTP typé sur le contrat exact de doPost()
  state/session.tsx       Session sécurisée (expo-secure-store)
  theme/colors.ts          Palette Activa (extraite de la maquette validée)
  components/              Icônes SVG, boutons, cartes, badges réutilisables
```

## 5. Build de test (APK / .app) sans passer par les stores

**Important — limite de ce bac à sable** : j'ai vérifié concrètement que ce
sandbox ne peut PAS produire un vrai `.apk` : `./gradlew` tente de
télécharger Gradle depuis `services.gradle.org`, domaine hors de la liste
autorisée ici (erreur HTTP 403 constatée). Idem pour le SDK Android
(`dl.google.com`) et pour iOS, qui nécessite de toute façon Xcode (macOS
uniquement, jamais disponible en sandbox Linux). Les deux scripts ci-dessous
sont donc prêts et testés dans leur logique, mais c'est sur **ta machine**
qu'ils doivent tourner.

### Option A — build local (`scripts/build-android.sh` / `build-ios.sh`)

```bash
# Android (nécessite Android Studio installé, fournit SDK + JDK)
npx expo prebuild --platform android   # génère android/ (pas fourni dans ce zip, régénéré à la demande)
./scripts/build-android.sh             # -> builds/activa-healthpass-mobile.apk

adb install -r builds/activa-healthpass-mobile.apk
adb shell am start -n com.groupactiva.healthpass/.MainActivity
```

```bash
# iOS Simulator (nécessite un Mac + Xcode — jamais possible sur Linux/Windows)
npx expo prebuild --platform ios
cd ios && pod install && cd ..
./scripts/build-ios.sh                 # -> builds/Activa HealthPass.app

xcrun simctl boot "iPhone 16" 2>/dev/null; open -a Simulator
xcrun simctl install booted "builds/Activa HealthPass.app"
xcrun simctl launch booted com.groupactiva.healthpass
```

Notes :
- Ce sont des builds **Release** (pas Debug — un build Debug Expo affiche
  l'écran "Development Servers" au lieu de l'app).
- La signature Android reste la clé de debug — suffisant pour du test
  interne, **pas** pour une soumission Play Store.
- Le build iOS ne fonctionne que sur **simulateur** — un vrai iPhone
  nécessite un profil de provisioning et une identité de signature (Option B
  ci-dessous, ou archive Xcode manuelle).

### Option B — EAS Build (recommandé si pas d'Android Studio/Xcode sous la main)

Service cloud d'Expo : construit l'APK (ou l'AAB) et l'`.ipa` à distance,
sans rien installer localement. **Vérifié depuis ce sandbox** :
`api.expo.dev` n'est pas joignable ici (`host_not_allowed`), donc les
commandes ci-dessous doivent être lancées **sur ta machine**, pas dans cet
environnement — tout ce qui pouvait être préparé sans réseau Expo l'a été
(`eas.json` déjà présent à la racine du projet, `app.json` déjà correct :
`name`, `slug`, `android.package`, `ios.bundleIdentifier`).

**1. Installer la CLI et te connecter** (une seule fois) :
```bash
npm install -g eas-cli
eas login              # compte Expo — gratuit, créer sur expo.dev si besoin
```

**2. Lancer le premier build Android** (le plus simple, ne nécessite pas de
compte Apple Developer) :
```bash
eas build --platform android --profile preview
```
La première exécution te demandera de lier le projet à ton compte Expo
(génère automatiquement `extra.eas.projectId` dans `app.json` — ne pas
l'inventer à la main, laisser `eas` le faire). Le build tourne sur les
serveurs Expo (quelques minutes), puis affiche un lien de téléchargement
direct vers un `.apk` installable sur n'importe quel téléphone Android
(active "Sources inconnues" dans les réglages du téléphone pour installer
en dehors du Play Store).

**3. Build iOS** (nécessite un compte Apple Developer, 99 $/an, pour un
vrai iPhone — sans ça, seul le simulateur est possible) :
```bash
# Simulateur (gratuit, pas de compte Apple requis) :
eas build --platform ios --profile preview

# Vrai iPhone / TestFlight (nécessite un compte Apple Developer) :
# éditer eas.json -> profil "preview" -> "ios": { "simulator": false }
eas build --platform ios --profile preview
eas submit --platform ios   # envoie vers TestFlight
```

**Profils déjà configurés dans `eas.json`** :
| Profil | Usage |
|---|---|
| `development` | Build avec `expo-dev-client`, pour itérer avec hot-reload |
| `preview` | Distribution interne — APK direct (Android), simulateur (iOS) — **celui à utiliser pour un premier test** |
| `production` | AAB (Android, requis par le Play Store) + incrément auto de version |

## 6. Prochaines étapes suggérées

1. ~~Google Sign-In pour le flux Superviseur~~ → câblé, voir section 2bis pour
   la configuration Google Cloud restante (seule étape manuelle).
2. ~~Câbler les écrans restants~~ → fait.
3. Test de bout en bout avec un vrai déploiement Apps Script (ce sandbox ne
   peut pas atteindre `script.google.com`) — priorité avant tout build.
4. Build de test interne (APK + simulateur iOS) — voir section 5.
5. Icône d'app + splash screen via le skill `app-icon`.
6. Une fois validé en interne : build EAS pour TestFlight (iOS) et test interne
   Play Console (Android).
