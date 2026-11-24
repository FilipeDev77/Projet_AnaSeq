# Prévision des hospitalisations COVID-19 : démarche, choix et résultats

## 1. Objectif

L'objectif du projet est de comparer plusieurs modèles de séries temporelles pour prévoir le taux hebdomadaire d'hospitalisation COVID-19 aux États-Unis. Nous partons des modèles les plus simples et ajoutons un élément à chaque étape (dynamique à court terme, saisonnalité, variables externes) afin de mesurer ce que chaque ajout apporte réellement.

Les données proviennent des réseaux de surveillance du CDC (COVID-NET, FluSurv-NET, RSV-NET et un réseau combiné). Le taux est exprimé pour 100 000 habitants.

## 2. Nettoyage des données

Le fichier brut contient 124 904 lignes et 12 colonnes. Pour une même semaine, il existe de nombreuses lignes : taux cumulé et taux hebdomadaire, différentes tranches d'âge, sexes, origines et États. Or une série temporelle doit avoir une seule valeur par date.

Nous avons donc gardé uniquement les dates hebdomadaires, le taux hebdomadaire (le taux cumulé repart à zéro à chaque saison), le niveau national pour toute la population (tous âges, sexes, origines et États confondus) et les valeurs observées. Il reste alors 1 417 lignes, soit exactement une valeur par semaine et par réseau, sans doublon. Les autres colonnes devenant constantes, nous n'avons conservé que la date, le réseau et la valeur.

Les séries grippe et RSV présentent des semaines manquantes chaque été, entre mai et début octobre, car ces réseaux ne collectent pas de données hors saison. Une semaine manquante signifie « non mesuré » et non « aucun cas ». Plutôt que de remplir ces trous par 0, par une moyenne ou par interpolation, nous avons préféré ne pas inventer de valeurs. Nous avons donc retenu la série COVID-19, complète depuis le 7 mars 2020 : 345 semaines consécutives, sans valeur manquante, jusqu'au 10 octobre 2026.

## 3. Exploration de la série

La série présente trois caractéristiques. On observe d'abord une saisonnalité annuelle, avec des pics qui reviennent chaque hiver. Ensuite, ces pics diminuent au fil des années, ce qui correspond à une tendance à la baisse, probablement liée à l'immunité acquise. Enfin, la période 2020-2022 est très différente du reste, avec des vagues fortes et irrégulières et un maximum d'environ 36 en janvier 2022 (vague Omicron). On peut parler d'un changement de régime.

Ce dernier point s'est révélé central dans toute la suite de l'analyse.

## 4. Stationnarité et différenciation

Un modèle ARIMA suppose une série stationnaire. L'ACF de la série brute décroît lentement, ce qui indique une forte persistance, et elle remonte autour du lag 52, ce qui confirme la saisonnalité annuelle.

Nous avons utilisé deux tests aux hypothèses opposées. Le test ADF donne une p-value de 0.003 et conclut que la série est stationnaire, tandis que le test KPSS donne une p-value inférieure à 0.01 et conclut le contraire. Ce désaccord s'explique par les changements de niveau : la série revient vers un niveau bas après chaque pic, ce que détecte ADF, mais son niveau général change entre 2020-2022 et la suite, ce que détecte KPSS. Après une différenciation, les deux tests concluent à la stationnarité. Nous avons donc retenu d = 1.

Dans un premier temps, nous avons choisi de ne pas appliquer de transformation logarithmique et de traiter la non-stationnarité uniquement par la différenciation.

## 5. Choix des ordres p et q

Nous avons lu l'ACF et la PACF de la série différenciée, qui correspond à la variation d'une semaine à l'autre. L'ACF vaut environ 0.70 au lag 1 : une hausse a tendance à se prolonger la semaine suivante. La PACF vaut environ -0.40 au lag 2 puis s'arrête : cette hausse finit par ralentir, comme une vague qui approche de son pic. L'ACF devient négative autour des lags 8 à 12, ce qui correspond à une descente qui suit la montée, environ une demi-vague plus tard. Enfin, un pic au lag 52 montre que la saisonnalité est toujours présente après la différenciation.

La PACF coupe après le lag 2, ce qui suggère un ARIMA(2,1,0). L'ACF coupe aussi assez vite, ce qui rend un ARIMA(0,1,2) plausible. Nous avons comparé plusieurs candidats avec l'AIC. L'ARIMA(1,1,0) est nettement moins bon (822 contre environ 775), ce qui montre qu'un seul retard ne suffit pas. L'ARIMA(0,1,2) obtient le meilleur AIC (771.3), devant l'ARIMA(2,1,0) (775.0). Ajouter un troisième paramètre n'apporte presque rien, donc nous avons gardé les deux modèles à deux paramètres.

Les coefficients de l'ARIMA(2,1,0) sont significatifs (0.97 et -0.40) et cohérents avec la PACF. Les résidus ne sont pas autocorrélés (Ljung-Box, p = 0.48), mais ils ne sont pas normaux et leur variance n'est pas constante, car les erreurs sont beaucoup plus grandes pendant les vagues de 2020-2022.

## 6. Protocole d'évaluation

Les données sont découpées dans l'ordre chronologique. Les modèles sont entraînés de mars 2020 à fin septembre 2025 (291 semaines) et évalués sur la dernière saison complète, d'octobre 2025 à octobre 2026 (54 semaines).

Nous avons évalué deux horizons de prévision. Dans le premier, le modèle prévoit les 54 semaines du test en une seule fois, à partir de la fin de l'entraînement, sans voir aucune valeur du test. Cela correspond à un usage de planification. Dans le second, le modèle prévoit uniquement la semaine suivante, chaque semaine, en connaissant toutes les valeurs observées jusque-là. Les paramètres restent ceux estimés sur l'entraînement. Cela correspond à un usage de surveillance. Les erreurs des deux horizons ne sont pas comparables entre elles.

Les erreurs sont mesurées avec la RMSE, qui pénalise fortement les grosses erreurs, et la MAE, qui donne l'erreur moyenne en points de taux. Chaque modèle est comparé à deux baselines : le naïf, qui reprend la dernière valeur connue, et le naïf saisonnier, qui reprend la valeur de la même semaine un an plus tôt.

## 7. Résultats : prévision de toute la saison (54 semaines)

Les baselines obtiennent une RMSE de 1.15 pour le naïf et de 1.23 pour le naïf saisonnier. Le naïf reste bloqué à 1.9, une valeur trop élevée pour presque toute la saison. Le naïf saisonnier a la bonne forme, avec un pic en janvier et un creux en été, mais il recopie un pic de 4.3 alors que celui de cette année ne dépasse pas 2.1.

Les deux ARIMA font mieux, avec une RMSE de 0.76 pour l'ARIMA(2,1,0) et de 0.65 pour l'ARIMA(0,1,2), ce qui confirme le classement de l'AIC. Leurs prévisions sont cependant presque plates. C'est normal : à long terme, le modèle remplace les chocs futurs par zéro, et avec d = 1 le niveau prévu devient constant. Ils gagnent surtout parce que ce niveau constant est proche du niveau moyen du test, pas parce qu'ils anticipent la saison.

Nous avons ensuite ajouté la saisonnalité avec un SARIMA, en gardant la partie (0,1,2) et une période de 52 semaines. Parmi les modèles sans différence saisonnière, les trois candidats ont des AIC très proches, et nous avons gardé le plus simple, SARIMA(0,1,2)(1,0,0)52. Il gagne environ 10 points d'AIC par rapport à l'ARIMA et obtient une RMSE de 0.38 et une MAE de 0.31. C'est le meilleur modèle sur cet horizon : environ 40 % d'erreur en moins que le meilleur ARIMA et deux tiers de moins que le naïf. Sa prévision reste toutefois prudente et ne reproduit pas vraiment le pic de janvier.

Le SARIMA avec une différence saisonnière, SARIMA(0,1,2)(0,1,1)52, est le seul à prévoir correctement le moment du pic. Mais il en surestime fortement l'amplitude (environ 12 au lieu de 2), descend sous zéro au printemps et obtient la pire RMSE (3.09). En reproduisant les variations des hivers précédents, il ajoute au niveau actuel les fortes hausses de 2021 et 2022. Son AIC très bas (681) ne se compare pas aux autres et ne garantit pas une bonne prévision.

Le SARIMAX avec des termes de Fourier décrit la saison par des sinus et des cosinus utilisés comme variables externes. Le meilleur AIC est obtenu avec trois paires de termes, mais la RMSE est de 2.92. La plupart des coefficients ne sont pas significatifs, car le modèle doit trouver une seule forme de saison pour des hivers très différents. Ce modèle a pratiquement le même AIC que le meilleur SARIMA, mais une erreur huit fois plus grande sur le test. Un bon ajustement sur l'entraînement ne garantit donc pas une bonne prévision.

Ces échecs ont une cause commune : la saisonnalité est apprise en points, alors que les pics diminuent d'année en année. Nous avons donc testé deux approches multiplicatives, où la saison est exprimée en proportion du niveau : Holt-Winters avec une saison multiplicative, et un SARIMA appliqué au logarithme de la série. Résultats à compléter.

## 8. Résultats : prévision à une semaine

À une semaine, le naïf obtient une RMSE de 0.16 et une MAE de 0.11, et le naïf saisonnier une RMSE de 1.18 et une MAE de 1.11.

Le naïf obtient le meilleur résultat, et aucun modèle ne fait mieux que lui. Cela s'explique par la nature de la série. Sur la période de test, le taux change de quelques dixièmes par semaine, donc la dernière valeur observée contient presque toute l'information utile. Les modèles ARIMA reviennent à prendre cette dernière valeur et à y ajouter une correction basée sur les variations et les erreurs récentes. Mais l'ampleur de cette correction a été estimée en grande partie sur les vagues de 2020-2022, beaucoup plus fortes. Sur la période récente, plus calme, la correction est trop forte et ajoute de l'erreur, notamment au moment des retournements. Les termes saisonniers, appris sur des hivers plus marqués, n'apportent pas non plus d'information utile à cet horizon.

Résultats détaillés des modèles à compléter.

## 9. Comparaison des deux horizons

Le meilleur modèle dépend de l'horizon de prévision. Pour anticiper toute la saison, les modèles saisonniers sont nettement meilleurs que les baselines, et le SARIMA(0,1,2)(1,0,0)52 réduit l'erreur des deux tiers par rapport au naïf. Pour prévoir la semaine suivante, la dernière valeur observée reste la meilleure prévision, et les modèles plus complexes n'apportent rien.

Ces deux résultats correspondent à deux usages différents. La prévision de la saison peut servir à planifier les ressources hospitalières pour l'hiver, tandis que la prévision à une semaine sert à surveiller l'évolution de l'épidémie au jour le jour.

## 10. Conclusion

La saisonnalité est indispensable pour prévoir la saison à l'avance : un ARIMA seul produit une prévision plate, alors que le SARIMA(0,1,2)(1,0,0)52 est le meilleur modèle sur 54 semaines. À une semaine, la persistance de la série rend le naïf très difficile à battre.

Le principal enseignement est que l'amplitude de la saison a fortement diminué depuis les vagues de 2020-2022. Plusieurs modèles échouent pour cette raison : le naïf saisonnier, le SARIMA avec différence saisonnière et le modèle avec termes de Fourier surestiment l'hiver 2025-2026, et les modèles à une semaine surréagissent aux variations récentes. Nous retenons aussi que l'AIC sert à choisir entre des modèles comparables, mais que seule l'évaluation sur des données de test permet de juger la qualité d'une prévision.

Cette analyse a plusieurs limites. L'évaluation repose sur une seule saison de test, et l'hiver 2025-2026 a été plutôt doux, ce qui avantage les modèles prudents. Les modèles sans transformation ont des résidus dont la variance n'est pas constante et peuvent produire des valeurs négatives. Enfin, en comparant beaucoup de modèles sur la même période de test, on risque de choisir un modèle qui fonctionne bien sur cette saison en particulier. Il serait préférable de choisir le modèle sur une saison de validation (2024-2025), puis de l'évaluer une seule fois sur le test.

Plusieurs pistes permettraient d'améliorer les résultats : entraîner les modèles uniquement sur la période après 2022, utiliser une saison différente avant et après la pandémie dans le SARIMAX, ou combiner les prévisions de plusieurs modèles.
