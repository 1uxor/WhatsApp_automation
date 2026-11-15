# WhatsApp Appointment Automation - Guide de configuration

## Architecture

```
WhatsApp (client envoie message)
    → Meta Cloud API (webhook)
    → n8n (webhook node)
    → Gemini Flash (parse le message)
    → Google Sheets (stockage)
```

## Format de message attendu

```
Souad
0707460751
ballayage
vendredi à 16h00
```

+ image de la conversation (optionnel)

## Les données extraites dans Google Sheets

| date_received | client_name | phone | service | appointment_date | appointment_time | image_url | raw_message |
|---|---|---|---|---|---|---|---|
| 2024-11-15T10:30:00Z | Souad | 0707460751 | ballayage | 2024-11-22 | 16:00 | | Souad\n0707460751\n... |

---

## Etape 1 : Clé API Gemini Flash (GRATUIT)

1. Va sur https://aistudio.google.com/apikey
2. Connecte-toi avec ton compte Google
3. Clique sur **"Create API Key"**
4. Copie la clé, tu en auras besoin dans n8n

---

## Etape 2 : WhatsApp Business API (Meta Cloud API)

### Est-ce gratuit ?
- **Recevoir** des messages : **GRATUIT**
- Pour ton cas (juste recevoir et stocker), c'est **100% gratuit**

### Comment configurer (etape par etape)

1. **Creer un compte Meta Developer**
   - Va sur https://developers.facebook.com/
   - Connecte-toi avec ton compte Facebook
   - Clique sur **"Mes applications"** (en haut a droite) → **"Creer une application"**
   - Type d'app : choisis **"Business"** (ou "Other" → "Business")
   - Donne un nom (ex: "Salon RDV Bot")
   - Clique **"Creer l'application"**

2. **Ajouter le produit WhatsApp**
   - Sur la page de ton app, descends et trouve **"WhatsApp"**
   - Clique **"Configurer"**
   - Ca te donne automatiquement :
     - Un **numero de test** WhatsApp
     - Un **token d'acces temporaire** (valide 24h)
   - **Note** : le token temporaire suffit pour tester, tu feras le permanent apres

3. **Configurer le Webhook** (apres avoir lance n8n)
   - Dans le menu gauche : **WhatsApp** → **Configuration**
   - Section **Webhook** → clique **"Modifier"**
   - **URL du callback** : `https://TON-DOMAINE-N8N/webhook/whatsapp-webhook`
   - **Token de verification** : tape un mot de passe au choix (ex: `mon_token_secret_123`)
   - Clique **"Verifier et enregistrer"**
   - Ensuite, clique **"Gerer"** a cote de **"messages"** et **active-le**

4. **Obtenir un token permanent** (optionnel, apres les tests)
   - Va dans **Parametres de l'entreprise** (business.facebook.com)
   - **Utilisateurs systeme** → Creer un utilisateur
   - Attribue l'app → Generer un token avec la permission `whatsapp_business_messaging`

> **Important** : ton n8n doit etre accessible sur internet (pas localhost).
> Pour tester en local, utilise ngrok : `ngrok http 5678`
> Ca te donne une URL publique comme `https://abc123.ngrok.io`

### Filtrage par numero

Le workflow filtre automatiquement les messages pour ne traiter que ceux d'**un seul numero**.

**Ou mettre le numero dans n8n :**
1. Ouvre le workflow dans n8n
2. Clique sur le noeud **"Is Allowed Number?"**
3. Dans le champ **"Value"** (a droite du `equals`), remplace `METS_TON_NUMERO_ICI` par le numero au format international **sans le +** (ex: `212707460751` pour un numero marocain 0707460751)
4. Les messages venant d'autres numeros seront ignores

---

## Etape 3 : Google Sheets

1. **Créer le Google Sheet**
   - Crée un nouveau Google Sheet
   - Nomme-le "RDV WhatsApp" (ou ce que tu veux)
   - Ajoute ces en-têtes en **ligne 1** :

   ```
   date_received | client_name | phone | service | appointment_date | appointment_time | image_url | raw_message
   ```

   (chaque en-tête dans une colonne séparée, A1 à H1)

2. **Copie l'ID du sheet**
   - L'ID est dans l'URL : `https://docs.google.com/spreadsheets/d/ICI_L_ID/edit`

---

## Etape 4 : Configurer n8n

### Installer n8n (si pas encore fait)

```bash
# Avec Docker (recommandé)
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  -e GEMINI_API_KEY=ta_cle_gemini_ici \
  docker.n8n.io/n8nio/n8n

# Ou avec npm
npm install n8n -g
export GEMINI_API_KEY=ta_cle_gemini_ici
n8n start
```

### Importer le workflow

1. Ouvre n8n dans ton navigateur : `http://localhost:5678`
2. Clique sur **"..."** → **"Import from file"**
3. Sélectionne le fichier `workflow.json`

### Configurer les credentials

1. **Variable d'environnement Gemini** :
   - Assure-toi que `GEMINI_API_KEY` est définie (voir la commande Docker ci-dessus)

2. **Google Sheets** :
   - Clique sur le noeud **"Google Sheets - Save"**
   - Clique sur "Credential" → "Create new"
   - Choisis **"Google Sheets OAuth2"**
   - Suis les instructions pour connecter ton compte Google
   - Sélectionne ton Google Sheet et le bon onglet (Sheet1)

3. **Webhook WhatsApp** :
   - Après avoir activé le workflow, n8n génère l'URL du webhook
   - Copie cette URL et colle-la dans la config Meta (Etape 2.4)

### Activer le workflow

1. Clique sur le toggle en haut à droite pour **activer** le workflow
2. Le webhook est maintenant en écoute

---

## Etape 5 : Tester

1. Envoie un message WhatsApp au numéro configuré :
   ```
   Souad
   0707460751
   ballayage
   vendredi à 16h00
   ```

2. Vérifie dans n8n que l'exécution apparaît (onglet "Executions")
3. Vérifie dans Google Sheets qu'une nouvelle ligne a été ajoutée avec les bonnes données

---

## Dépannage

- **Le webhook ne reçoit rien** : Vérifie que l'URL est accessible depuis internet (ngrok pour les tests)
- **Gemini retourne une erreur** : Vérifie que ta clé API est correcte et que la variable `GEMINI_API_KEY` est définie
- **Google Sheets ne se remplit pas** : Vérifie les credentials OAuth et que les noms de colonnes correspondent exactement
- **Le message n'est pas bien parsé** : Le prompt Gemini est en français et gère la plupart des formats. Tu peux l'ajuster dans le noeud "Gemini Flash - Parse Message"
