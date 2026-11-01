# Devis : heures de travail et total manuel, sans champ « Taux »

## Demande finale (01/11/2026)

**Uniquement dans l'éditeur des tâches des devis, pour React et Flutter Web :**
- conserver le champ **Heures** modifiable ;
- supprimer **uniquement le champ « Taux »** ;
- rendre le champ **Total** modifiable directement ;
- une modification des heures **ne change jamais** le montant total saisi ;
- les produits, les taxes, les remises et les factures préexistantes ne changent pas.

Exemple : Service = Main-d'œuvre ; Heures = 24 ; Total = 1 735 520 FCFA. Après modification à 30 heures, le montant reste 1 735 520 FCFA.

## Format de données (backend Laravel)

Les lignes des devis sont des objets JSON `line_items`, et le moteur d'Invoice Ninja utilise `quantity × cost`.

Pour une ligne de tâche à total manuel (`type_id='2'`), les deux interfaces doivent **émettre** :

```json
{
  "type_id": "2",
  "product_key": "Main-d'œuvre",
  "notes": "Installation et configuration du réseau Wi-Fi",
  "quantity": 1,
  "cost": 1735520,
  "manual_total": true,
  "task_hours": 24
}
```

La quantité de facturation technique est toujours **1**. Le prix du forfait va dans `cost`. Le nombre réel d'heures est conservé dans `task_hours`. Les champs du backend `App\\DataMapper\\InvoiceItem` assurent leur cast/persistance en JSON. Aucune migration SQL n'est nécessaire.

**Ne jamais** enregistrer `quantity=24, cost=1735520` : cela créerait un total erroné.

### Lignes héritées
Si `manual_total` est absent/faux, afficher les heures historiques depuis `quantity`, et le total historique depuis `quantity × cost`. Dès que l'utilisateur édite ce poste via un éditeur de devis modifié, le convertir en `manual_total=true, quantity=1, cost=ancien_total, task_hours=heures` en préservant le total. Les factures à tarification horaire ne doivent pas être converties.

Le PDF doit afficher les vraies heures (`task_hours` pour les lignes manuelles, `quantity` pour les anciennes). Le calcul du sous-total, des taxes et du total reste natif et utilise la quantité technique de 1 pour un forfait. Pour ne plus afficher « Taux » dans le **PDF**, il faut aussi retirer la colonne « Taux » du design des tâches (`task_columns`) de l'entreprise.

## Fichiers backend effectivement modifiés dans cette branche

- `app/DataMapper/InvoiceItem.php` : `task_hours` et `manual_total` (avec casts).
- `app/Utils/Traits/MakesInvoiceValues.php` : valeur `$task.hours` provenant des heures réelles.
- `app/Utils/HtmlEngine.php` : somme des heures PDF correcte.
- `tests/Unit/CleanLineItemsTest.php` : cas manuel + cas historique.

## Correctifs de sources frontend préparés et validés au niveau des hunks

Les sources des deux interfaces ne font **pas** partie du fork Laravel `nPhenix-IT/invoiceninja`, et le compte lié n'a pas de droit d'écriture sur les dépôts publics amont. Les correctifs source sont donc conservés sous forme de patches, **pas encore intégrés ni compilés** :

- React : `integrations/react/quote-task-hours-manual-total.patch` (7 fichiers).
- Flutter Web et mobile : `integrations/flutter/quote-task-hours-manual-total-part1.patch` (7 fichiers).
- Flutter tableau desktop : `integrations/flutter/quote-task-hours-manual-total-desktop.patch` (1 fichier).

Les 15 fichiers des patches ont été rapprochés des sources amont : tous les hunks correspondent au texte publié dans les sources interrogées lors de l'analyse. **Cela ne constitue pas une compilation, un test visuel ou un test de fonctionnement.**

### Application React

Dans un checkout séparé de `invoiceninja/ui` correspondant à la version utilisée :

```bash
git switch -c feature/quote-manual-task-total
git apply --check /chemin/vers/invoiceninja/integrations/react/quote-task-hours-manual-total.patch
git apply /chemin/vers/invoiceninja/integrations/react/quote-task-hours-manual-total.patch
npm ci
npm run build
```

### Application Flutter

Dans un checkout séparé de `invoiceninja/flutter` correspondant à la version utilisée :

```bash
git switch -c feature/quote-manual-task-total
git apply --check /chemin/vers/invoiceninja/integrations/flutter/quote-task-hours-manual-total-part1.patch
git apply --check /chemin/vers/invoiceninja/integrations/flutter/quote-task-hours-manual-total-desktop.patch
git apply /chemin/vers/invoiceninja/integrations/flutter/quote-task-hours-manual-total-part1.patch
git apply /chemin/vers/invoiceninja/integrations/flutter/quote-task-hours-manual-total-desktop.patch
dart run build_runner build --delete-conflicting-outputs
flutter analyze
flutter test
flutter build web --release
```

Le remplacement des assets `public/main.dart.js` et des autres assets Flutter dans le backend Laravel nécessite une procédure de déploiement versionnée et un test sur un site de préproduction. Ne jamais modifier directement le JavaScript Flutter compilé ni remplacer des assets sur le serveur de production sans validation.

## Critères d'acceptation

1. Les éditeurs **React et Flutter Web** affichent « Service », « Description », « Heures », « Total » ; ils n'affichent plus « Taux » dans les devis.
2. Devis : 24 heures / 1 735 520 FCFA ; après modification à 30 heures, toujours 1 735 520 FCFA.
3. Sauvegarde, réouverture, duplication et révision préservent la valeur des heures, le montant et le type de service.
4. TVA et remises par ligne et au devis, en taxes exclusives et inclusives, restent exactes.
5. PDF, emails, portail client et conversion devis vers facture conservent les montants ; PDF affiche les vraies heures.
6. Un ancien devis de 3 heures × 10 000 FCFA continue à totaliser 30 000 FCFA ; aucune conversion silencieuse sans édition.
7. Les factures existantes, le suivi de tâches et les produits ne sont pas affectés.
8. Tests mobile, desktop et visuels, tests de non-régression, tests de devise XOF et séparateurs.
9. Réaliser les tests sur une base isolée (ne pas modifier la vraie production).

## État de cette Pull Request

Backend adapté et correctifs sources des deux interfaces fournis. **Aucune application frontend n'a été recompilée et les tests d'intégration/visuels n'ont pas été exécutés.** Pull Request en brouillon : ne pas fusionner tant que ce n'est pas validé.
