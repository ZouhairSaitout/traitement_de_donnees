# Script de présentation — Prédiction des prix de billets d'avion
**Durée cible : 5 minutes** | *Les indications entre [crochets] sont des repères pour l'orateur, à ne pas lire*

---

## ÉLOÏSE — Introduction, Objectif, Description des données
*Environ 1 min 30*

Bonjour à tous. Imaginez : vous hésitez à acheter un billet d'avion — est-ce que vous achetez maintenant, ou vous attendez deux jours en espérant que le prix baisse ?

C'est exactement la question qu'on s'est posée. Est-ce qu'on peut prédire comment évolue le prix d'un billet — et surtout, est-ce qu'il y a des moments où attendre est réellement payant ?

*[pointer le graphique moyennetousvol]* Ce graphique montre la tendance sur 300 000 vols intérieurs indiens. En gros : plus on se rapproche du départ, plus le prix monte. Mais la courbe est loin d'être une ligne droite — il y a des chutes, des remontées, des effets de seuil visibles. C'est ça qui rend le problème intéressant, et difficile.

Pour rester rigoureux, on a restreint l'analyse aux vols au départ de Bangalore, vers trois destinations représentatives : Mumbai pour les vols courts, Chennai pour les vols moyens, et Delhi pour les longs. On a aussi exclu les vols avec plus d'une escale pour garder des données comparables.

Côté variables : trois variables numériques — le prix, le nombre de jours avant le départ, et la durée du vol — et des variables catégorielles comme la compagnie, la classe, le nombre d'escales.

*[pointer la distribution des prix]* Une chose importante : le prix Economy est très asymétrique. Beaucoup de billets peu chers et quelques valeurs très élevées qui tirent la distribution vers le haut. Ça va directement influencer nos choix de modélisation. Et les jours avant départ sont quasi-uniformément distribués — toutes les fenêtres de réservation sont bien représentées, c'est une bonne nouvelle pour l'analyse.

Je passe la parole à Adnane pour les features.

---

## ADNANE — Handcrafted Features
*Environ 1 min 15*

Avant de modéliser quoi que ce soit, on a construit des variables à la main — et chacune répond à un problème précis.

La première, c'est la **fenêtre de réservation** : on découpe les données en quatre périodes — dernière minute, court terme, moyen terme, et anticipé. C'est surtout utile pour visualiser comment les prix se distribuent selon la période de réservation.

Ensuite, le **prix par heure de vol**. Comparer directement le prix d'un vol de 2h et d'un vol de 20h, c'est pas honnête. Cette normalisation par la durée rend les comparaisons cohérentes.

La transformation la plus importante : le **log du prix**. Le prix brut est très asymétrique — on vient de le voir. Quelques billets très chers écrasent toute la distribution et gonflent les erreurs du modèle. En prenant le logarithme, on symétrise la distribution et on stabilise ces erreurs. C'est une transformation classique en statistique dès qu'on a ce type de données.

Même logique pour le **log des jours avant départ**. La relation entre les jours et le prix n'est pas du tout linéaire : elle décroît très vite au début, puis se stabilise longtemps. Le log capture naturellement cette forme en "courbe" sans avoir besoin d'un modèle complexe.

Et enfin les **hinge features** — ce sont elles qui constituent le cœur du modèle. Je laisse Zouhair vous expliquer comment elles fonctionnent dans la régression.

---

## ZOUHAIR — Régression
*Environ 1 min 45*

Pourquoi une régression linéaire classique ne marche pas ici ? Parce que le prix ne diminue pas de manière uniforme. Longtemps à l'avance, les prix sont quasi-stables. Puis ils explosent dans les derniers jours. Une droite ne peut pas modéliser ça.

On a comparé trois approches. *[pointer le graphique comparaison]* La **régression linéaire simple** donne un R² autour de 0.66 — insuffisant, elle rate complètement la non-linéarité. La **régression polynomiale de degré 2** monte à 0.85-0.91 — c'est mieux, mais elle est trop lisse : elle ne capture pas les changements de dynamique que les données montrent clairement.

Notre modèle final, c'est un **modèle piecewise avec log**. *[pointer la formule]* La formule combine un terme linéaire, un terme log, et trois termes appelés "hinge" : max(0, d − k).

Voilà ce que fait ce max. Tant qu'on est au-delà du seuil k — par exemple J-7 — le terme vaut zéro et le modèle l'ignore totalement. Dès qu'on passe en dessous du seuil, il s'active et ajoute une pente supplémentaire. En d'autres termes : le modèle change de comportement à J-7, à J-15, et à J-30 — sans avoir besoin d'entraîner trois modèles séparés. Un seul modèle, mais avec des "engrenages" qui s'enclenchent au bon moment.

Ces trois seuils ne sont pas arbitraires — ce sont les paliers qu'on observe visuellement dans les données, là où la dynamique change réellement.

*[pointer piecewise_economy]* Résultat : R² entre 0.89 et 0.97 selon la destination. Le modèle suit à la fois la tendance de fond — les prix stables longtemps à l'avance — et les accélérations brutales en fin de fenêtre.


---

## ADNANE — Conclusion
*Environ 30 secondes*

Pour finir, quelques limites à garder en tête. Les données sont limitées aux vols indiens — on ne peut pas transposer directement à d'autres marchés. Il n'y a pas d'information sur la saison — les effets des fêtes et des vacances sont absents. Et on prédit une **tendance médiane**, pas le prix exact de votre billet.

Des pistes d'amélioration : des splines ou du LOESS pour plus de flexibilité, intégrer la saisonnalité, ou du Random Forest pour capturer les interactions non-linéaires.

Mais le message clé qu'on retient : **à moins de dix jours du départ, chaque jour d'attente vous coûte plus cher**. Notre modèle vous le dit clairement. Merci.

---

*Durée estimée : ~5 min à 130 mots/minute (environ 640 mots)*
