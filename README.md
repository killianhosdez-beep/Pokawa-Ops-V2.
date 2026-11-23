# Pokawa Ops

Application de gestion de cuisine : listes de production, inventaires, contrôle automatique des quantités et signalement obligatoire des écarts. Utilisable sur téléphone, tablette et ordinateur.

**Aucune étape de compilation.** Ce sont des fichiers HTML/CSS/JS simples, hébergeables sur GitHub Pages.

## Essayer tout de suite (mode démo)

Ouvre `index.html` (ou le site GitHub Pages) : l'application démarre en **mode démo**, sans rien configurer. Les données restent dans le navigateur. Comptes d'essai :

| Profil | Identifiant | Mot de passe |
|---|---|---|
| Manager | `manager` | `manager1234` |
| Employée | `emma` | `emma1234` |
| Administrateur | `admin` | `admin1234` |

> Le mode démo sert à tester l'interface. Il n'est **pas sécurisé** : n'y saisis jamais de vrais mots de passe.

## Passer en mode réel (Supabase)

1. Crée un projet gratuit sur [supabase.com](https://supabase.com).
2. **SQL Editor** : colle le contenu de `supabase/schema.sql` et exécute-le (il peut être relancé sans danger).
3. **Authentication → Providers → Email** : désactive « Confirm email » et **« Allow new users to sign up »** (les comptes sont créés uniquement par un administrateur).
4. **Authentication → Users → Add user** : crée ton compte (e-mail `ton-identifiant@pokawa-ops.local`, mot de passe de ton choix, case « Auto Confirm User » cochée), puis dans le SQL Editor :
   ```sql
   update public.profiles set role = 'admin', active = true
   where id = (select id from auth.users where email = 'ton-identifiant@pokawa-ops.local');
   ```
5. **Fonction de gestion des comptes** (création d'employés, réinitialisation de mot de passe) :
   ```bash
   npm i -g supabase
   supabase login
   supabase link --project-ref <ton-projet>
   supabase functions deploy admin-users
   ```
6. Dans `js/config.js`, renseigne `SUPABASE_URL` et `SUPABASE_ANON_KEY` (**Project Settings → API**, clé « anon / publishable »). Cette clé est publique par conception. **Ne mets jamais la clé `service_role` dans ce dépôt.**
7. Publie sur GitHub Pages (**Settings → Pages → Deploy from a branch → main / root**).

Connexion : les employés tapent un identifiant simple (`emma`) ; il est converti en `emma@pokawa-ops.local`. Si tu changes `AUTH_DOMAIN` dans `js/config.js`, change-le aussi dans la fonction `admin-users` (variable d'environnement `AUTH_DOMAIN`).

## Ce que l'application fait

**Employé**
- Voit uniquement les listes **publiées**, avec la quantité demandée en gros chiffres.
- Saisit la quantité préparée (boutons − / +, virgule acceptée, « Tout est prêt » en un appui).
- Le calcul de l'écart est automatique : égal → validation ; inférieur → validation normale impossible, fenêtre « Quantité insuffisante », raison obligatoire (commentaire obligatoire pour « Autre raison ») ; supérieur → surplus signalé (commentaire obligatoire ou non, au choix du manager par liste).
- Suit ses signalements et les consignes du management.
- Ne peut jamais modifier une quantité demandée ni une consigne.

**Management**
- Tableau de bord : listes du jour, tâches à réaliser / terminées / en cours / partielles / à corriger, signalements, retards, modifications après publication.
- Création de listes avec types personnalisables, unités, consignes, heure limite, collage rapide de plusieurs lignes, duplication d'une liste d'un autre jour.
- Publication / dépublication / clôture. Modifier une liste publiée est possible et **tracé**.
- Signalements : compléter, nouvelle vérification, consigne, accepter l'écart (justification obligatoire), clôturer. **Une exception acceptée ne devient jamais « terminé »** : elle reste « Écart accepté par le management ».
- Historique : recherche par date, type, produit, employé ; journal de toutes les actions ; export CSV (Excel).
- Administrateur : création et désactivation de comptes, rôles (administrateur / manager / employé), réinitialisation de mot de passe.

## Sécurité (mode réel)

Les droits sont vérifiés **par la base de données** (`supabase/schema.sql`), pas seulement par l'écran :
- les employés n'ont aucun droit d'écriture direct ; ils passent par `submit_item_result`, qui impose les règles (motif obligatoire, quantité valide, pas de double validation, liste ouverte) ;
- les inscriptions libres créent un compte **inactif** ;
- un compte désactivé ne voit plus rien et ne peut plus écrire ;
- l'historique est alimenté automatiquement (déclencheurs) et lisible uniquement par le management.

## Tests

```bash
node --test tests/*.test.js          # règles métier (13 tests)
bash tests/run_schema_test.sh        # schéma contre un vrai PostgreSQL (39 vérifications)
python3 tests/e2e.py                 # scénario complet dans un navigateur (47 vérifications, Playwright)
```

## Limites connues

- Notifications : alertes **dans l'application** (tableau de bord, pastille de signalements, actualisation en direct). Pas de notification push téléphone dans cette version.
- Pas d'accès hors connexion : si le réseau tombe, l'écran indique « Hors connexion » et les saisies ne sont pas enregistrées.
- Sauvegardes : à activer côté Supabase (sauvegardes quotidiennes sur les offres payantes ; sinon exporter régulièrement via l'écran Historique → CSV, ou `pg_dump`).
- Le mode réel n'a pas pu être testé contre un vrai projet Supabase hébergé (schéma testé sur PostgreSQL local, interface testée en mode démo). Fais le premier essai avec un projet vide.
- Modules futurs (stocks, recettes, hygiène, contrôle qualité, rapports, multi-établissements) : **non développés**, comme indiqué dans le cahier des charges.
