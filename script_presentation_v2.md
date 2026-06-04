# Script de présentation — Prédiction des prix de billets d'avion
**Durée cible : 5 minutes** | *Les indications entre [crochets] sont des repères pour l'orateur, à ne pas lire*

---

## 🎤 ÉLOÏSE — Introduction, Objectif, Description des données
*Environ 1 min 30*

Bonjour à tous. Imaginez : vous hésitez à acheter un billet d'avion — est-ce que vous achetez maintenant, ou vous attendez deux jours en espérant que le prix baisse ?

C'est exactement la question qu'on s'est posée. Est-ce qu'on peut prédire comment évolue le prix d'un billet — et surtout, est-ce qu'il y a des moments où attendre est réellement payant ?

*[pointer le graphique moyennetousvol]* Ce graphique montre la tendance sur 300 000 vols intérieurs indiens. La première chose qu'on voit : le prix moyen passe d'environ 4 500 INR à J-50 à près de 15 000 INR à J-0 — soit un triplement en moins de deux mois. Mais la courbe n'est pas une ligne droite. On voit des paliers, des chutes brutales, des remontées ponctuelles. C'est ça qui rend le problème intéressant — et difficile.

Pour rester rigoureux, on a restreint l'analyse aux vols au départ de Bangalore, vers trois destinations représentatives : Mumbai pour les vols courts, Chennai pour les vols moyens, et Delhi pour les longs. On a aussi exclu les vols avec plus d'une escale pour garder des données comparables.

Côté variables : trois variables numériques — le prix, le nombre de jours avant le départ, et la durée du vol — et des variables catégorielles comme la compagnie, la classe, le nombre d'escales.

*[pointer la distribution des prix — histogramme bleu]* Une chose importante : le prix Economy est très asymétrique. La grande majorité des billets se concentre entre 3 000 et 8 000 INR, mais quelques billets très chers tirent la distribution vers le haut. Ce déséquilibre va directement influencer nos choix de modélisation — j'en repasse la main à Adnane dans un instant.

*[pointer le violin plot]* Ce graphique le confirme de façon encore plus claire. En classe Economy, la médiane des billets de dernière minute est à environ 10 000 INR, contre à peine 4 000 INR pour les billets achetés plus de 30 jours à l'avance. En Business, l'effet est encore plus marqué : la dispersion est énorme dans la fenêtre dernière minute — ce sont exactement ces seuils que notre modèle va chercher à capturer.

Enfin, les jours avant départ sont quasi-uniformément distribués sur toutes les fenêtres — toutes les périodes de réservation sont bien représentées dans nos données, c'est une bonne nouvelle pour la robustesse de l'analyse.

Je passe la parole à Adnane pour les features.

---

## 🎤 ADNANE — Handcrafted Features
*Environ 1 min 15*

Merci Éloïse. Avant de modéliser quoi que ce soit, on a construit des variables à la main — et chacune répond à un problème précis.

La première, c'est la **fenêtre de réservation** : on découpe les données en quatre périodes — dernière minute, court terme, moyen terme, et anticipé. C'est la variable qu'on vient de voir dans le violin plot : elle nous permet de visualiser comment les prix se distribuent selon la période de réservation et de vérifier visuellement où sont les ruptures de comportement.

Ensuite, le **prix par heure de vol**. Comparer directement le prix d'un vol Bangalore-Mumbai d'1h30 et d'un vol Bangalore-Delhi de 3h, c'est comparer des choses incomparables. Cette normalisation par la durée rend les comparaisons cohérentes entre destinations.

La transformation la plus importante : le **log du prix**. Le prix brut est très asymétrique — on vient de le voir. Ces quelques billets très chers gonflent les erreurs du modèle et rendent les résidus hétéroscédastiques. En prenant le logarithme, on symétrise la distribution, on stabilise la variance des résidus, et on change l'interprétation du modèle : au lieu de prédire des effets additifs en INR, on prédit des effets multiplicatifs — ce qui est bien plus cohérent avec la réalité du marché aérien.

*[pointer la distribution log(Prix) — histogramme cyan]* Le résultat est visible ici : après transformation log, la distribution est nettement plus symétrique et concentrée.

Même logique pour le **log des jours avant départ**. La relation entre les jours et le prix n'est pas du tout linéaire : l'effet d'un jour de plus est énorme à J-3, mais quasi nul à J-45. Le log compresse naturellement les grandes valeurs et étire les petites — ce qui capture exactement cette forme sans avoir besoin d'un modèle complexe.

Et enfin les **hinge features** — ce sont elles qui constituent le cœur du modèle. Ces seuils J-7, J-15 et J-30, on ne les a pas choisis arbitrairement : ce sont les paliers qu'on observe visuellement dans les données — les endroits où la dynamique de prix change réellement, comme le montre le violin plot. Je laisse Zouhair vous expliquer comment elles fonctionnent dans la régression.

---

## 🎤 ZOUHAIR — Régression
*Environ 1 min 45*

Merci Adnane. Commençons par poser le problème. Une régression linéaire classique cherche à ajuster une droite sur les données — elle suppose que chaque jour supplémentaire avant le départ a exactement le même effet sur le prix. Or on vient de voir que ce n'est pas du tout le cas : longtemps à l'avance, les prix sont stables. Puis ils explosent dans les derniers jours. Une droite ne peut pas capturer ça.

*[pointer le graphique comparaison_modeles]* On a comparé trois approches. La **régression linéaire simple** donne un R² autour de 0.66 — elle rate complètement la non-linéarité, et on le voit clairement : la droite en rouge passe à côté des données sur toute la fenêtre proche du départ. La **régression polynomiale de degré 2** monte à 0.85-0.91 — c'est mieux, mais le polynôme est une courbe globalement lisse et symétrique, il surestime les prix à J-40 et sous-estime l'accélération brutale à J-5.

Notre modèle final résout les deux problèmes à la fois. C'est un **modèle piecewise avec log** — autrement dit, une régression par morceaux. L'idée est simple : au lieu d'imposer une seule relation sur tout le domaine, on découpe le temps en segments et on autorise le modèle à changer de pente à chaque frontière. Concrètement, un seul modèle mais avec des "engrenages" qui s'enclenchent aux bons moments.

*[pointer la formule]* La formule combine quatre ingrédients. Un terme linéaire `d` pour la tendance de fond. Un terme `log(d)` pour capturer la décroissance rapide au début — on vient d'en parler. Et trois termes hinge : `max(0, d − 7)`, `max(0, d − 15)`, `max(0, d − 30)`.

Voilà ce que fait ce max. Prenons `max(0, d − 7)` : tant qu'on est à plus de 7 jours du départ, ce terme vaut zéro et le modèle l'ignore totalement. Dès qu'on passe sous J-7, il s'active et ajoute une pente supplémentaire — le modèle "voit" qu'on entre dans une nouvelle dynamique. En d'autres termes, le coefficient associé à cette hinge mesure le changement de pente qui se produit à J-7. C'est exactement ça, une régression par morceaux : pas trois modèles séparés, mais un seul modèle avec des ruptures structurelles aux bons endroits.

On pourrait se demander : si le modèle change de pente à J-7, est-ce qu'il y a un saut dans la courbe — deux segments qui ne se rejoignent pas ? Non, et c'est justement l'élégance de la hinge. À J-7 exactement, `max(0, 7 − 7)` vaut zéro. Juste avant, il vaut aussi zéro. Juste après, il commence à croître depuis zéro. Les deux morceaux se raccordent parfaitement au seuil — pas de discontinuité, pas de saut. Ce qu'il y a, c'est un **changement de pente** — une inflexion, pas une rupture. La courbe reste continue, elle change juste de vitesse. C'est pour ça qu'on voit sur le graphique une courbe qui s'accélère progressivement, et non des segments disjoints.

Important : on modélise le **log du prix**, pas le prix brut. Ça veut dire que les coefficients s'interprètent comme des effets multiplicatifs — et pour obtenir le prix réel, on repasse par l'exponentielle à la fin.

*[pointer piecewise_economy]* Résultat : R² entre 0.89 et 0.93 selon la destination — Chennai à 0.929, Delhi à 0.907, Mumbai à 0.892. Mumbai est légèrement moins bien ajusté, probablement parce que c'est un marché plus concurrentiel avec plus de variabilité dans les prix à court terme. Le modèle suit à la fois la tendance de fond stable longtemps à l'avance et les accélérations brutales en fin de fenêtre.

Je repasse la parole à Adnane pour conclure.

---

## 🎤 ADNANE — Conclusion
*Environ 30 secondes*

Pour finir, quelques limites à garder en tête, par ordre d'importance. La plus critique : il n'y a **aucune information sur la saisonnalité** — les effets des fêtes et des périodes de vacances sont complètement absents du dataset, alors qu'ils expliquent probablement une large part de la variabilité résiduelle. Ensuite, les données sont limitées aux vols indiens — on ne peut pas transposer directement à d'autres marchés. Enfin, on prédit une **tendance médiane** — pas le prix exact de votre billet. On a choisi la médiane et non la moyenne justement pour être robuste à ces quelques billets très chers qui tirent la distribution vers le haut.

Des pistes d'amélioration : des splines ou du LOESS pour plus de flexibilité dans la forme de la courbe, intégrer la saisonnalité, ou du Random Forest pour capturer les interactions non-linéaires entre compagnie, classe et fenêtre de réservation.

Mais le message clé qu'on retient : **à partir de J-7, chaque jour d'attente vous coûte significativement plus cher**. Notre modèle a la réponse. Merci.

---

*Durée estimée : ~5 min à 130 mots/minute (environ 750 mots)*
