# Guide de réinitialisation de l'authentification à deux facteurs (2FA) Bitwarden

## 📋 Table des matières
1. [Contexte](#contexte)
2. [Prérequis](#prérequis)
3. [Architecture de l'installation](#architecture-de-linstallation)
4. [Procédure de désactivation du 2FA](#procédure-de-désactivation-du-2fa)
5. [Vérification et redémarrage](#vérification-et-redémarrage)
6. [Recommandations de sécurité](#recommandations-de-sécurité)
7. [Dépannage](#dépannage)

---

## Contexte

Ce guide documente la procédure pour désactiver l'authentification à deux facteurs (2FA) d'un compte Bitwarden en accédant directement à la base de données SQL Server. Cette procédure est nécessaire lorsque :
- L'utilisateur a perdu l'accès à son application d'authentification
- Les codes de récupération ne sont plus disponibles
- L'accès au compte est bloqué par le 2FA

⚠️ **Avertissement** : Cette procédure nécessite un accès direct au serveur hébergeant Bitwarden et doit être effectuée avec précaution.

---

## Prérequis

### Accès système
- Accès SSH au serveur hébergeant Bitwarden
- Permissions sudo/root
- Docker installé et fonctionnel

### Informations nécessaires
- Mot de passe SA de la base de données SQL Server
- Adresse email du compte à modifier

### Connaissances requises
- Commandes Linux de base
- Commandes Docker
- Requêtes SQL basiques

---

## Architecture de l'installation

### Structure des répertoires
```
~/bitwarden/
├── bitwarden.sh          # Script de gestion Bitwarden
└── bwdata/
    ├── env/
    │   └── uid.env       # Configuration UID/GID
    └── scripts/
```

### Conteneurs Docker en cours d'exécution

```bash
# Vérifier les conteneurs actifs
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
```

Conteneurs principaux :
- `bitwarden-mssql` : Base de données SQL Server
- `bitwarden-api` : API backend
- `bitwarden-identity` : Service d'authentification
- `bitwarden-web` : Interface web
- `bitwarden-nginx` : Reverse proxy

---

## Procédure de désactivation du 2FA

### Étape 1 : Récupérer le mot de passe de la base de données

```bash
docker exec bitwarden-mssql printenv | grep SA_PASSWORD
```

**Exemple de sortie :**
```
SA_PASSWORD=f4zPqmrAKPyAKqi0VMBR1XudzdLDn8uM
```

### Étape 2 : Se connecter à SQL Server

```bash
docker exec -it bitwarden-mssql /opt/mssql-tools/bin/sqlcmd \
  -S localhost \
  -U sa \
  -P 'VOTRE_MOT_DE_PASSE_SA'
```

### Étape 3 : Explorer la base de données (optionnel)

```sql
-- Sélectionner la base de données vault
USE vault;
GO

-- Lister toutes les tables disponibles
SELECT name FROM sys.tables;
GO

-- Afficher la structure de la table User
SELECT COLUMN_NAME, DATA_TYPE 
FROM INFORMATION_SCHEMA.COLUMNS 
WHERE TABLE_NAME = 'User';
GO
```

### Étape 4 : Vérifier l'état actuel du 2FA

```sql
USE vault;
GO

SELECT 
    Email, 
    TwoFactorProviders, 
    TwoFactorRecoveryCode 
FROM [User] 
WHERE Email = 'utilisateur@example.com';
GO
```

**Exemple de résultat avec 2FA activé :**
```
TwoFactorProviders: {"0":{"Enabled":true,"MetaData":{"Key":"W4V27T27NOKC3MCWP3SOFGZMIMXRAFHG"}}}
TwoFactorRecoveryCode: pbc4mzekzc1aq63sqb8ad07xmm32asy9
```

### Étape 5 : Désactiver le 2FA

```sql
USE vault;
GO

UPDATE [User] 
SET TwoFactorProviders = NULL, 
    TwoFactorRecoveryCode = NULL
WHERE Email = 'utilisateur@example.com';
GO
```

**Résultat attendu :**
```
(1 rows affected)
```

### Étape 6 : Vérifier la modification

```sql
SELECT 
    Email, 
    TwoFactorProviders, 
    TwoFactorRecoveryCode 
FROM [User] 
WHERE Email = 'utilisateur@example.com';
GO
```

**Résultat attendu :**
```
Email: utilisateur@example.com
TwoFactorProviders: NULL
TwoFactorRecoveryCode: NULL
```

### Étape 7 : Quitter SQL Server

```sql
EXIT
```

Ou appuyez sur `CTRL+C`

---

## Vérification et redémarrage

### Méthode 1 : Redémarrage complet (recommandé)

```bash
cd ~/bitwarden
./bitwarden.sh restart
```

### Méthode 2 : Redémarrage des services concernés uniquement

```bash
docker restart bitwarden-identity bitwarden-api bitwarden-web
```

### Vérifier que les conteneurs redémarrent correctement

```bash
docker ps --format "table {{.Names}}\t{{.Status}}"
```

Tous les conteneurs doivent afficher `Up X minutes (healthy)`

### Tester la connexion

1. Accédez à votre interface web Bitwarden
2. Entrez votre email et mot de passe principal
3. La connexion devrait réussir **sans** demander de code 2FA

---

## Recommandations de sécurité

### ⚠️ Risques de sécurité

La désactivation du 2FA réduit **considérablement** la sécurité de votre compte Bitwarden :
- Le compte n'est protégé que par le mot de passe principal
- En cas de compromission du mot de passe, l'attaquant a un accès direct
- Tous les mots de passe stockés sont vulnérables

### ✅ Actions recommandées immédiatement après

1. **Réactiver le 2FA dès que possible**
   - Connectez-vous à votre compte
   - Allez dans Paramètres → Sécurité → Authentification à deux facteurs
   - Configurez une nouvelle méthode 2FA (TOTP recommandé)

2. **Sauvegarder les codes de récupération**
   - Téléchargez les codes de récupération
   - Stockez-les dans un endroit sûr (offline de préférence)
   - Ne les partagez jamais

3. **Vérifier l'activité du compte**
   - Consultez l'historique des connexions
   - Vérifiez les appareils autorisés
   - Révoquez les sessions suspectes

4. **Considérer un changement de mot de passe principal**
   - Si vous soupçonnez une compromission
   - Utilisez un mot de passe fort et unique

### 🔐 Bonnes pratiques 2FA

**Applications d'authentification recommandées :**
- **Authy** : Sauvegarde cloud sécurisée
- **Google Authenticator** : Simple et fiable
- **Microsoft Authenticator** : Intégration Microsoft
- **1Password** : Si vous utilisez déjà 1Password

**Ne pas utiliser :**
- SMS (vulnérable aux attaques SIM swap)
- Email (peut être compromis)

---

## Dépannage

### Problème : Impossible de trouver le mot de passe SA

**Solution :**
```bash
# Vérifier les variables d'environnement du conteneur
docker exec bitwarden-mssql env | grep -i password

# Chercher dans les fichiers de configuration
find ~/bitwarden/bwdata -name "*.env" -exec grep -l "PASSWORD" {} \;
```

### Problème : Erreur "Invalid object name 'User'"

**Causes possibles :**
- Mauvaise base de données sélectionnée
- Table nommée différemment

**Solution :**
```sql
-- Vérifier la base de données actuelle
SELECT DB_NAME();
GO

-- Changer vers la base vault
USE vault;
GO

-- Lister toutes les tables
SELECT name FROM sys.tables WHERE name LIKE '%User%';
GO
```

### Problème : Le 2FA est toujours demandé après redémarrage

**Solutions à essayer :**

1. Vider le cache du navigateur
2. Essayer un autre navigateur ou mode privé
3. Vérifier à nouveau la base de données :
```sql
SELECT Email, TwoFactorProviders FROM [User] WHERE Email = 'votre@email.com';
GO
```

4. Redémarrage complet de Bitwarden :
```bash
cd ~/bitwarden
./bitwarden.sh stop
./bitwarden.sh start
```

### Problème : Conteneurs ne démarrent pas après modification

**Vérifier les logs :**
```bash
# Logs du conteneur API
docker logs bitwarden-api --tail 100

# Logs du conteneur Identity
docker logs bitwarden-identity --tail 100

# Logs du conteneur MSSQL
docker logs bitwarden-mssql --tail 100
```

### Problème : Accès refusé à SQL Server

**Vérifier :**
```bash
# Le conteneur est bien en cours d'exécution
docker ps | grep mssql

# Tester la connexion
docker exec bitwarden-mssql /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'PASSWORD' -Q "SELECT @@VERSION"
```

---

## Commande rapide (one-liner)

Pour les utilisateurs avancés, voici une commande unique pour désactiver le 2FA :

```bash
docker exec -it bitwarden-mssql /opt/mssql-tools/bin/sqlcmd \
  -S localhost \
  -U sa \
  -P "$(docker exec bitwarden-mssql printenv | grep SA_PASSWORD | cut -d'=' -f2)" \
  -Q "USE vault; UPDATE [User] SET TwoFactorProviders = NULL, TwoFactorRecoveryCode = NULL WHERE Email = 'utilisateur@example.com';"
```

⚠️ Remplacez `utilisateur@example.com` par l'email concerné.

---

## Informations complémentaires

### Structure de la table User

Colonnes principales liées à l'authentification :
- `Id` : Identifiant unique (GUID)
- `Email` : Adresse email
- `EmailVerified` : État de vérification de l'email
- `MasterPassword` : Hash du mot de passe principal (chiffré)
- `TwoFactorProviders` : Configuration JSON du 2FA
- `TwoFactorRecoveryCode` : Code de récupération 2FA
- `SecurityStamp` : Token de sécurité
- `FailedLoginCount` : Nombre de tentatives échouées
- `LastFailedLoginDate` : Date de dernière tentative échouée

### Format TwoFactorProviders

Quand activé, le champ contient un JSON structuré :
```json
{
  "0": {
    "Enabled": true,
    "MetaData": {
      "Key": "BASE32_ENCODED_SECRET"
    }
  }
}
```

Où `"0"` représente le type de 2FA (0 = Authenticator/TOTP)

---

## Changelog

| Date | Version | Changements |
|------|---------|-------------|
| 2025-11-06 | 1.0 | Documentation initiale |

---

## Licence et responsabilité

Ce document est fourni à titre informatif uniquement. L'auteur et les contributeurs déclinent toute responsabilité en cas de perte de données ou de problèmes de sécurité résultant de l'utilisation de ces procédures.

**Utilisez à vos propres risques.**

---

## Ressources supplémentaires

- [Documentation officielle Bitwarden](https://bitwarden.com/help/)
- [Bitwarden Self-Hosting Guide](https://bitwarden.com/help/install-on-premise-linux/)
- [SQL Server Documentation](https://docs.microsoft.com/en-us/sql/)
- [Docker Documentation](https://docs.docker.com/)

---

*Document créé le : 06 novembre 2025*  
*Dernière mise à jour : 06 novembre 2025*
