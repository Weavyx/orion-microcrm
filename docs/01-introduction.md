# 1. Introduction

## 1.1 Contexte du projet

Orion développe MicroCRM, une application de gestion de la relation client (CRM) simplifiée, utilisée par ses départements technique et commercial pour gérer des personnes et des organisations. Jusqu'ici, les déploiements étaient faits à la main, avec des retards et des erreurs. Maria, CTO d'Orion, demande d'industrialiser la chaîne de livraison de l'application.

Le dépôt reçu (`OpenClassrooms-Student-Center/P7-FSJA`, deux commits) contient :

- `back/` : une API REST Spring Boot 3.2.5 exposée par Spring Data REST (Java 17, Gradle 8.7 par le wrapper), avec une base HSQLDB embarquée en mémoire et 2 tests ;
- `front/` : une application Angular 17.3, testée avec Karma et Jasmine (8 tests) ;
- un `Dockerfile` à plusieurs étapes, qui produit une image pour le front (servi par Caddy), une image pour le back et une image tout-en-un de démonstration ;
- aucune intégration continue, aucun fichier Docker Compose, aucune release.

Commandes lancées en local le 14/11/2026 (Java 21, Node 24). Celles du back et des tests viennent du README ; pour le front, le README ne donne que `npm install` et `npx @angular/cli serve`, et le build a été lancé avec `npm ci` et `npm run build` :

| Commande | Résultat | Cause et contournement |
| --- | --- | --- |
| `./gradlew build` (dans `back/`) | échec : `Permission denied` | `gradlew` est commité sans droit d'exécution ; `bash gradlew build` réussit (2 tests sur 2). Correction suivie dans l'[issue #12](https://github.com/Weavyx/orion-microcrm/issues/12) |
| `java -jar build/libs/microcrm-0.0.1-SNAPSHOT.jar` | l'API démarre sur le port 8080 | les données sont perdues à chaque redémarrage (base en mémoire) |
| `npm ci` puis `npm run build` (dans `front/`) | succès, sortie dans `dist/microcrm` | |
| `npm test` (dans `front/`) | 8 tests sur 8, mais la commande ne rend jamais la main et ouvre une fenêtre Chrome | `ng test` reste en mode surveillance, qu'aucune option du projet ne désactive ; `npx ng test --no-watch --browsers=ChromeHeadless` s'arrête avec un code de sortie 0 (lancé avec `CHROME_BIN=/usr/bin/google-chrome`) |

L'analyse complète (constats, veille des versions, bonnes pratiques) est dans l'[annexe A](annexe-a-analyse-initiale.md).

## 1.2 Objectifs de l'industrialisation

La demande de Maria porte sur :

- l'automatisation de la construction, des tests et du déploiement avec GitHub Actions ;
- la conteneurisation du front et du back, orchestrée avec Docker Compose, en vérifiant que l'application démarre ;
- l'analyse de la qualité et de la sécurité du code avec SonarQube Cloud ;
- les plans de test périodique, de sécurité et de sauvegarde ;
- la création automatique des releases GitHub avec leurs artefacts (JAR et build Angular) ;
- la centralisation des logs avec une stack ELK, la mesure de la performance de la livraison par les métriques DORA et des KPI, et un plan de mise à jour ;
- la documentation de toutes les étapes.

Le but affiché est « d'assurer à l'avenir une livraison continue, de réduire la dette technique et de garantir la stabilité du système », avec des recommandations que l'équipe puisse appliquer dès les prochaines itérations.

Périmètre : le code reçu n'est pas réécrit. Pour chaque défaut relevé, trois questions se posent dans l'ordre : est-ce demandé ? Sinon, est-ce que cela bloque quelque chose de demandé ? Sinon, est-ce à la fois peu coûteux et utile pour la suite du projet ? Une réponse positive entraîne la correction ; sinon, le défaut devient une recommandation documentée. Par exemple, `gradlew` non exécutable bloque la CI, la construction des images et la release : il est corrigé. La migration d'Angular et de Spring Boot n'est ni demandée ni bloquante, et elle est risquée avec 10 tests : elle est décrite dans le plan de mise à jour (section 8). La dette technique se réduit ainsi par des recommandations priorisées et par la CI, qui rendra ces migrations plus sûres.

## 1.3 Technologies principales

| Composant | Version du projet | Support au 14/11/2026 |
| --- | --- | --- |
| Java | 17 (cible de compilation) | maintenu |
| Spring Boot | 3.2.5 | hors support |
| Gradle | 8.7 (wrapper) | maintenance seulement |
| Base de données | HSQLDB embarquée, en mémoire | non évalué |
| Angular | 17.3 | hors support |
| Node.js | non déclaré ; Angular 17.3 accepte les versions 18 et 20 | 18 et 20 hors support |
| Tests du front | Karma 6.4.3 et Jasmine | Karma déprécié |
| Images Docker | images finales `alpine:3.19` | hors support depuis le 01/11/2025 |

Outils ajoutés par l'industrialisation : GitHub Actions (intégration et déploiement continus), SonarQube Cloud (analyse statique de la qualité et de la sécurité), Docker Compose (lancement de l'application), semantic-release (versions et releases), stack ELK (logs centralisés).

Détail des versions et sources : [annexe A](annexe-a-analyse-initiale.md). Migration recommandée : section 8, plan de mise à jour.

## 1.4 Présentation rapide du pipeline CI/CD

À compléter à la fin de la partie 1, une fois le pipeline en place.
