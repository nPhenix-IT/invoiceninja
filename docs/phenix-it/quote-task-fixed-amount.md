# Montants forfaitaires pour les tâches de devis (Phenix IT)

## Objectif
Dans la **création et l'édition des devis uniquement**, permettre la saisie directe d'un montant total de prestation, sans demander un nombre d'heures ni un tarif horaire. Préserver les factures, le suivi des temps et les devis historiques.

## Architecture vérifiée
Le backend de ce dépôt accepte les lignes de devis dans `line_items`. `App\\DataMapper\\InvoiceItem` contient `type_id='2'` (service/tâche), `quantity` et `cost`. Le calcul existant est effectué à partir de `quantity * cost`. Pour un forfait : `quantity=1` et `cost=montant_saisi`.

L'éditeur React de devis n'est **pas présent en sources dans ce dépôt** : `resources/views/react/index.blade.php` héberge une application React précompilée, et les sources actuelles de l'interface proviennent du dépôt public séparé `invoiceninja/ui`. Éviter tout patch direct de bundle compilé, tout remplacement de valeur DOM et toute réécriture globale de `InvoiceSum`.

## Comportement attendu
- Interface devis / tâches : colonnes « Service », « Description », « Montant forfaitaire », taxes éventuelles et « Total ».
- Le champ « Montant forfaitaire » est directement modifiable, en devise du devis ; la quantité interne reste à 1.
- Tâches historiques avec `quantity != 1` : les afficher en mode horaire hérité (ou proposer une conversion **explicite** conservant le total `quantity * cost`). Ne jamais convertir silencieusement.
- L'onglet « Tâches » ne doit plus afficher `(1 h)` pour les lignes forfaitaires.
- Produit inchangé. Facture inchangée, sauf les valeurs attendues à la conversion du devis. Le suivi du temps dans le module Tâches reste inchangé.
- Montant zéro autorisé ; montants invalides rejetés ; taxes et remises gérées par le moteur natif.
- Conversion devis -> facture et rendu PDF conservent le total exact.

## Changement nécessaire dans le projet frontend
Créer un **fork** de `https://github.com/invoiceninja/ui`, branché sur la version React correspondant à cette installation. Les points d'extension identifiés dans le dépôt frontend public :
- `src/pages/quotes/create/components/CreatePage.tsx`
- `src/pages/quotes/edit/Edit.tsx`
- `src/pages/invoices/common/components/ProductsTable.tsx`
- `src/pages/invoices/common/components/TasksTabLabel.tsx`
- `src/pages/invoices/common/hooks/useTaskColumns.ts`
- `src/pages/invoices/common/hooks/useResolveInputField.tsx`
- `src/common/constants/product-aliases.ts`

**Ne pas modifier** le hook partagé de résolution des tâches globalement. Ajouter une variante spécifiquement utilisée par les deux écrans de devis, avec le rendu du montant connecté à `onLineItemPropertyChange('cost', montant, index)` et `quantity=1` à la création de la ligne. Pour les lignes historiques avec des quantités autres que 1, préserver le calcul/les champs d'origine.

Le libellé de l'onglet doit compter les lignes forfaitaires ou afficher « Tâches », et non la somme des quantités en heures. Le tableau PDF peut rester « Service / Description / Total », sans afficher de colonne « Heures » ; les variables PDF sont distinctes des colonnes de l'éditeur.

## Vérifications avant fusion
1. Créer un devis avec service 1 735 520 XOF, quantité interne 1, coût 1 735 520 ; enregistrer puis rouvrir. Le total reste 1 735 520 avant taxes.
2. Ajouter deux services forfaitaires (dont un de valeur 0), et un produit ; vérifier tous les totaux.
3. Modifier le montant, enregistrer, recharger, vérifier la persistance.
4. Générer le PDF, envoyer le devis et convertir en facture ; vérifier les montants.
5. Ouvrir un ancien devis à 3 heures × 10 000 : son total reste 30 000 et aucune transformation silencieuse ne survient.
6. Tester taxes, remises, devis en plusieurs devises, écran mobile et exports.
7. Contrôler qu'aucune interface de facturation ni de suivi d'heures n'a changé.

## Statut
**Analyse et spécification préparées**. Le backend ne nécessite aucun changement pour enregistrer un forfait sous forme d'une ligne de service à quantité 1. **La modification React, sa compilation et les tests en navigateur restent à réaliser** avec les sources de l'application frontend. Ne pas déployer cette branche comme si elle contenait déjà l'interface demandée.
