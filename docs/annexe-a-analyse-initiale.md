# Annexe A. Analyse initiale (état au 14/11/2026)

Cette annexe répond à la première étape de la mission : analyser le projet reçu et réaliser une veille technologique ciblée. C'est une photographie de l'état du projet, faite le 14/11/2026 (Dockerfile lu en détail le 20/11/2026, dépôt inchangé entre-temps) ; elle n'est pas mise à jour. La suite donnée à chaque constat est décrite dans la section indiquée.

Colonne « Vérification » : « lu » signifie constaté en lisant le fichier, « exécuté » signifie constaté en lançant la commande ou l'application.

## A.1 Constats sur le code reçu

La numérotation d'origine est conservée ; les numéros C2, C3, C5 et C11 ont été reclassés en contraintes (A.3) ou en veille (A.4).

| # | Emplacement | Constat | Vérification | Suite donnée |
| --- | --- | --- | --- | --- |
| C1 | `front/karma.conf.js` | Trois navigateurs déclarés (`ChromeHeadlessNoSandbox`, `ChromeHeadless`, `Chrome` avec fenêtre), mode surveillance et `restartOnFileChange: true` : `npm test` ne rend jamais la main | exécuté | CI : options `--no-watch --browsers=ChromeHeadless` en ligne de commande, fichier inchangé (section 2) |
| C4 | `back/build.gradle` | `spring-boot-starter-data-jpa` déclaré deux fois ; Gradle ignore le doublon | lu ; build réussi | recommandation (section 9) |
| C6 | `front/src/app/config.ts` | `API_BASE_URL = "http://localhost:8080"` en dur : l'URL est fixée au build, et l'API doit être joignable sur `localhost:8080` depuis le navigateur | lu | conteneurisation et déploiement (section 3) |
| C7 | back | Base HSQLDB en mémoire : les données ne survivent pas à un redémarrage | exécuté (personne créée, absente après redémarrage) | sauvegarde (section 7) |
| C8 | `front/angular.json` | La configuration des tests référence `src/favicon.ico` ; le fichier est `favicon.png` | lu ; aucun effet observé | recommandation (section 9) |
| C9 | `back/bin/` | Sortie de compilation commitée (probablement par un IDE) ; `bin/` absent de `back/.gitignore` | lu ; non recréé par Gradle | recommandation (section 9) |
| C10 | `InitialDataFixture.java` | `@Autowired` sur des champs `final` initialisés à `null` ; fonctionne quand même | lu ; données de démonstration créées | recommandation (section 9) |
| C12 | `Dockerfile:1` | `FROM node` sans tag : désigne au 14/11/2026 Node 26, qu'Angular 17.3 ne prend pas en charge | lu ; Docker Hub | conteneurisation (section 3) |
| C13 | `back/gradlew` | Mode `100644` dans l'index git : `./gradlew` échoue sur tout clone neuf (poste du développeur, runner de CI, `docker build` à la ligne 16 du Dockerfile, release) | exécuté | correction suivie dans l'[issue #12](https://github.com/Weavyx/orion-microcrm/issues/12) |
| C14 | `front/src/index.html` | Bulma 1.0.0 chargé depuis jsDelivr sans attribut `integrity` (SRI) | lu | sécurité (section 5) |
| C15 | `organization-details.component.spec.ts`, `person-details.component.spec.ts` | Même `describe('IndividualDetailsComponent')` dans les deux fichiers : dans un rapport, on ne sait pas quel composant a échoué | lu | recommandation (section 9) |
| C16 | `front/package.json`, lockfile | `jasmine-core ~5.1.0` déclaré, mais Karma exécute la version 4.6.0 (copie installée par `karma-jasmine` 5.1.0) | lu ; écran de Karma | mise à jour (section 8) |
| C17 | `SpringDataRestCustomization.java` | CORS ouvert à toutes les origines (GET, POST, PATCH, DELETE) et aucune authentification | lu ; `POST` accepté sans identifiant | sécurité (section 5) |
| C18 | `Dockerfile:40` | Cible `back` : `EXPOSE 4200` alors que l'API écoute sur 8080 (4200 est le port de `ng serve`) ; sans effet sur le fonctionnement, mais trompeur | lu | conteneurisation (section 3) |
| C19 | `misc/docker/supervisor.ini:24` | `make_main_rpcinterface.` se termine par un point : pourrait empêcher supervisord de démarrer (image `standalone`) | lu ; effet non vérifié | conteneurisation (section 3), à observer |
| C20 | `Dockerfile:34,42` | Nom du JAR écrit en dur avec la version (`microcrm-0.0.1-SNAPSHOT.jar`) ; `build/libs` contient aussi `microcrm-0.0.1-SNAPSHOT-plain.jar` | lu | conteneurisation (section 3) et release (section 2) |
| C21 | `Dockerfile:44-54` | Cible `standalone` : fusionne les systèmes de fichiers des images `front` et `back` (les fichiers communs sont écrasés) et lance deux processus avec supervisord | lu ; écrasement déduit | recommandation ; le compose prévu utilisera les cibles `front` et `back` (section 3) |
| C22 | `Dockerfile:18,23,32,36` | Images finales `alpine:3.19` (hors support depuis le 01/11/2025), Caddy et JRE installés par `apk` au lieu d'images officielles ; aucune instruction `USER` (exécution en root) ; `apk add` sans `--no-cache` | lu | conteneurisation (section 3) |
| C23 | `Dockerfile:3-8,10,16,36` | `COPY ./front` avant `npm ci` (tout changement de code relance l'installation des dépendances) ; tests lancés pendant le `docker build` ; build en JDK 17, exécution en JRE 21 | lu | conteneurisation (section 3) |
| C24 | `.dockerignore` | N'exclut ni `.git`, ni `misc/screenshots`, ni `back/bin` : contexte de build alourdi, `back/bin` copié dans l'étape de build | lu | conteneurisation (section 3) |
| C25 | `README.md` | Ne mentionne ni la base de données, ni Gradle 8.7, ni de version de Node ; `./gradlew build` échoue tel quel | lu ; exécuté | README complet (fin de projet) |

## A.2 Avertissements à l'installation et au démarrage

| # | Source | Constat | Suite donnée |
| --- | --- | --- | --- |
| A1 | `npm audit` | 90 vulnérabilités (7 faibles, 27 moyennes, 51 élevées, 5 critiques), surtout dans l'outillage de développement (webpack, webpack-dev-server, ws) ; avec `--omit=dev` : 8 (4 moyennes, 4 élevées), toutes dans `@angular/*`, avec Angular 22 pour seul correctif. `npm audit fix` non lancé | sécurité (section 5) |
| A2 | `npm ci` sous npm 11.19 | Scripts d'installation non approuvés (esbuild deux fois, nice-napi), probablement non exécutés : le comportement de l'installation dépend de la version de npm | CI : version de Node, donc de npm, fixée (section 2) |
| A3 | démarrage du back | `spring.jpa.open-in-view is enabled by default` | recommandation (section 9) |

## A.3 Contraintes pour la CI/CD

- Tests du front : Chrome est nécessaire (en local, `CHROME_BIN` a été fixé explicitement) ; commande `ng test --no-watch --browsers=ChromeHeadless`. L'option sans bac à sable (`--no-sandbox`) sur un runner reste à vérifier ; le lanceur `ChromeHeadlessNoSandbox` existe déjà dans la configuration.
- `gradlew` : à rendre exécutable avant la CI et avant l'adaptation du Dockerfile (C13).
- Node.js : Angular 17.3 n'accepte que les versions 18.13 et suivantes de la ligne 18, ou 20.9 et suivantes de la ligne 20, toutes deux hors support. Node 24 a construit et testé le front sans avertissement en local. La version retenue doit être fixée à un seul endroit, lu par la CI, le Dockerfile et les développeurs.
- Rapports : aucun rapport de tests au format JUnit côté front, aucun rapport de couverture (JaCoCo côté back, lcov côté front) ; à ajouter si la couverture est envoyée à SonarQube Cloud.
- Health check : pas de Spring Boot Actuator dans le back.
- URL de l'API fixée au build du front (C6).
- Ports : back 8080 ; front 80 et 443 (Caddy) ; 4200 sert seulement au serveur de développement d'Angular.

## A.4 Veille des versions

Sources : angular.dev (versions et compatibilité), endoflife.date (dates de support), README de Karma, Docker Hub, pages GitHub des projets.

| Composant | Version du projet | État au 14/11/2026 | Versions maintenues |
| --- | --- | --- | --- |
| Angular | 17.3 | hors support (versions 2 à 19 non supportées) | 22 active ; 21 LTS jusqu'en 06/2027 ; 20 LTS jusqu'au 28/11/2026 |
| Node.js pour Angular 17.3 | non déclaré | 18 (fin le 30/04/2025) et 20 (fin le 30/04/2026) hors support | 22 (jusqu'au 30/04/2027), 24 (jusqu'au 30/04/2028), 26 (jusqu'au 30/04/2029) |
| Spring Boot | 3.2.5 | hors support (support libre fini le 31/12/2024, commercial le 31/12/2025) ; toute la ligne 3.x est hors support libre | 4.0 (jusqu'au 31/12/2026), 4.1 (jusqu'au 31/07/2027) |
| Hibernate ORM | 6.4.4 | hors support depuis le 06/08/2024 | 7.4 |
| Tomcat | 10.1.20 | branche maintenue, 42 versions correctives de retard | 10.1.62 |
| Java | cible 17 ; poste en 21 | maintenus (Temurin 17 jusqu'au 31/10/2027 ; 21 jusqu'au 31/12/2029) | |
| Gradle | 8.7 | branche 8 en maintenance seulement | 8.14.7 ; 9.9.1 active |
| Karma | 6.4.3 | déprécié | Web Test Runner, jasmine-browser-runner, Jest, Vitest |
| spring-boot-data-fixtures | 0.2.0 | maintenance non établie | |
| Bulma | 1.0.0 | 4 versions correctives de retard | 1.0.4 |
| Image `alpine` | 3.19 | hors support depuis le 01/11/2025 | 3.22 à 3.24 |
| Image `node` (sans tag) | | désigne Node 26 | |
| Image `gradle:jdk17` | | désigne Gradle 9.9.1 (sans effet : le wrapper impose 8.7) | |

Conséquence : deux migrations majeures sont à décrire dans le plan de mise à jour (Angular 17 vers 21 ou 22, Spring Boot 3.2 vers 4.x), sans être réalisées dans cette mission.

## A.5 Bonnes pratiques CI/CD retenues

Sources : documentation GitHub (Java avec Gradle, Node.js, usage sécurisé de GitHub Actions), dépôt `gradle/actions`, guide des tests d'Angular ; pratique 9 : article de blog (source secondaire). Leur application est justifiée dans la section 2.

1. Mêmes commandes en CI qu'en local.
2. Versions de Java et de Node fixées (`actions/setup-java`, `actions/setup-node`).
3. `npm ci` plutôt que `npm install` (installation exacte du lockfile).
4. Cache des dépendances (`cache: npm`, cache de `setup-gradle`) : il réduit le temps de build, suivi comme KPI.
5. `gradle/actions/setup-gradle` : vérifie l'authenticité de `gradle-wrapper.jar` depuis sa version 4.
6. Actions tierces épinglées par SHA plutôt que par tag (contrepartie : lisibilité et mises à jour).
7. `GITHUB_TOKEN` en lecture seule par défaut, droits ajoutés job par job.
8. Secrets jamais en clair ; un secret affiché dans un log est changé.
9. Un job par composant (back, front) ; les filtres de chemins sont possibles, mais inutiles à cette taille de projet.
10. `ng test` détecte la variable `CI=true` selon le guide d'Angular 22 ; rien ne le garantit pour Angular 17 : les options explicites sont gardées.
11. Durée maximale par job (`timeout-minutes`, 360 minutes par défaut) : limite les dégâts d'une commande qui ne s'arrête pas, sans remplacer `--no-watch`.

## A.6 Exemple open source analysé

`jhipster/jhipster-sample-app`, fichier `.github/workflows/github-actions.yml` : workflow d'une application de même pile (Spring Boot et Angular). Pratiques écartées : `chmod +x mvnw` dans la CI (contournement répété à chaque exécution au lieu de corriger le dépôt), actions dans des versions anciennes, ni cache ni `permissions`, `npm install` au lieu de `npm ci`. Limite : un seul exemple a été analysé.
