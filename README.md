# country-service

Micro-service REST de gestion de pays (Spring Boot), utilisé comme projet fil rouge d'un pipeline **CI/CD** :
**GitHub → Jenkins → Maven → JUnit → SonarQube → Nexus**.

> TP DevOps — ENSI, II3-GL

## Sommaire

1. [Technologies](#technologies)
2. [Structure du projet](#structure-du-projet)
3. [Prérequis](#prérequis)
4. [Lancer l'application](#lancer-lapplication)
5. [API REST](#api-rest)
6. [Tests unitaires](#tests-unitaires)
7. [Analyse de qualité avec SonarQube](#analyse-de-qualité-avec-sonarqube)
8. [Pipeline Jenkins](#pipeline-jenkins)
9. [Stockage des artefacts avec Nexus](#stockage-des-artefacts-avec-nexus)
10. [Dépannage](#dépannage)

## Technologies

| Outil | Usage |
|---|---|
| Java 17 (JDK 21 sur Jenkins) | Langage |
| Spring Boot 3.5.4 | Framework (Web, Data JPA, Data REST) |
| H2 / MySQL | Base de données (H2 en mémoire par défaut) |
| Lombok | Réduction du code répétitif (getters/setters) |
| springdoc-openapi 2.8.6 | Documentation Swagger UI |
| Maven 3.9.x | Build et gestion des dépendances |
| JUnit 5, Mockito, MockMvc | Tests unitaires |
| JaCoCo | Couverture de code |
| SonarQube | Analyse de la qualité du code |
| Jenkins | Intégration continue |
| Nexus Repository | Stockage du jar et des rapports |
| Docker | Exécution de SonarQube et Nexus |

## Structure du projet

```
country-service
├── Jenkinsfile                      # Pipeline CI/CD
├── lombok.config                    # Exclut le code Lombok de la couverture
├── pom.xml
└── src
    ├── main
    │   ├── java/com/countryservice/demo
    │   │   ├── CountryServiceApplication.java
    │   │   ├── beans/Country.java                  # Entité métier
    │   │   ├── controllers/CountryController.java  # Couche API REST
    │   │   ├── services/CountryService.java        # Couche métier
    │   │   └── repositories/CountryRepository.java # Couche persistance
    │   └── resources/application.properties
    └── test
        ├── java/com/countryservice/demo
        │   ├── CountryServiceApplicationTests.java
        │   ├── services/ServiceMockitoTests.java
        │   └── controllers/
        │       ├── ControllerMockitoTests.java
        │       └── ControllerMockMvcTests.java
        └── resources/application.properties        # H2 pour les tests
```

Architecture en couches : `Client → Controller → Service → Repository → Base de données`.

## Prérequis

- JDK 17 ou plus
- Maven **3.9.x** (Maven 4 n'est pas compatible avec le scanner SonarQube)
- Git
- Docker Desktop (pour SonarQube et Nexus)

## Lancer l'application

```bash
git clone https://github.com/fidayahyaoui7/country-service3.git
cd country-service3
mvn clean package
java -jar target/country-service-0.0.1-SNAPSHOT.jar
```

L'application démarre sur `http://localhost:8082`.

- Swagger UI : `http://localhost:8082/swagger-ui/index.html`
- Base H2 en mémoire par défaut. Pour utiliser MySQL, décommentez le bloc MySQL dans `src/main/resources/application.properties` et renseignez votre mot de passe.

## API REST

| Méthode | URL | Description | Codes de réponse |
|---|---|---|---|
| `GET` | `/getcountries` | Liste de tous les pays | 200 |
| `GET` | `/getcountries/{id}` | Pays par identifiant | 200, 404 |
| `GET` | `/getcountries/countryname?name=Tunisia` | Pays par nom | 200, 404 |
| `POST` | `/addcountry` | Ajouter un pays | 201, 409 (id déjà utilisé) |
| `PUT` | `/updatecountry/{id}` | Modifier un pays | 200, 404 |
| `DELETE` | `/deletecountry/{id}` | Supprimer un pays | 200, 404 |

Exemple de corps pour `POST /addcountry` :

```json
{
  "idCountry": 1,
  "name": "Tunisia",
  "capital": "Tunis"
}
```

## Tests unitaires

Les tests couvrent chaque couche avec des mocks :

| Fichier | Objet testé | Outils |
|---|---|---|
| `ServiceMockitoTests` | `CountryService` (repository simulé) | JUnit 5, Mockito |
| `ControllerMockitoTests` | `CountryController` (service simulé) | JUnit 5, Mockito |
| `ControllerMockMvcTests` | Endpoints HTTP sans serveur | MockMvc (`standaloneSetup`), Mockito |
| `CountryServiceApplicationTests` | Démarrage du contexte Spring | `@SpringBootTest` |

```bash
mvn clean test
```

Un rapport de couverture JaCoCo est généré par `mvn clean verify` :
`target/site/jacoco/index.html`.

Pendant les tests, la base H2 est utilisée (`src/test/resources/application.properties`), sans MySQL.

## Analyse de qualité avec SonarQube

1. Démarrer SonarQube :

   ```bash
   docker pull sonarqube
   docker run -d --name sonarqube -p 9000:9000 sonarqube
   ```

   Attendre « SonarQube is operational » (`docker logs -f sonarqube`), puis ouvrir `http://localhost:9000` (`admin` / `admin`, mot de passe à changer à la première connexion).

2. Créer le projet `countryservice3` (Create a local project) et générer un **token**.

3. Lancer l'analyse, sur **une seule ligne** (Windows CMD) :

   ```bash
   mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=countryservice3 -Dsonar.projectName="countryservice3" -Dsonar.host.url=http://localhost:9000 -Dsonar.token=VOTRE_TOKEN
   ```

   Le token ne doit jamais être commité dans le dépôt.

4. Consulter le tableau de bord et le **Quality Gate** (Passed / Failed) sur `http://localhost:9000/dashboard?id=countryservice3`.

## Pipeline Jenkins

```mermaid
flowchart LR
    A[Poste de dev] -->|git push| B[GitHub]
    B -->|webhook| C[Jenkins]
    C --> D[Maven : compile + package]
    D --> E[JUnit : tests]
    E --> F[SonarQube : analyse]
    F --> G[Nexus : jar + rapports]
```

Le pipeline est décrit dans le fichier `Jenkinsfile` :

| Étape | Action |
|---|---|
| Checkout code | Récupère le code depuis GitHub |
| Compile, test, package | `mvn clean install` + publication des résultats JUnit |
| SonarQube Analysis | Analyse avec le plugin Maven Sonar (rapport JaCoCo inclus) |
| display message | Message de contrôle |
| Deploy jar to Nexus | `mvn deploy` (désactivée tant que `NEXUS_READY = 'false'`) |
| Store reports in Nexus | Envoie les rapports JaCoCo et Surefire (même condition) |

### Configuration Jenkins nécessaire

- Plugins : Git, Maven Integration, SonarQube Scanner, Pipeline Stage View.
- **Tools** : JDK nommé `JDK21`, Maven nommé `M2_HOME` (3.9.x).
- **System → SonarQube servers** : nom `MySonarQubeServer`, URL `http://localhost:9000`, token enregistré dans un identifiant *Secret text*.
- **Credentials** : `nexus-creds` (utilisateur et mot de passe Nexus).
- Job de type **Pipeline script from SCM** pointant sur ce dépôt (branche `main`, chemin `Jenkinsfile`).
- Déclenchement automatique : webhook GitHub (`/github-webhook/`) exposé avec ngrok, et l'option *GitHub hook trigger for GITScm polling*.

## Stockage des artefacts avec Nexus

```bash
docker run -d --name nexus -p 8081:8081 sonatype/nexus3
docker exec nexus cat /nexus-data/admin.password
```

Ouvrir `http://localhost:8081` (utilisateur `admin`), puis :

1. Créer un dépôt **raw (hosted)** nommé `reports` pour les rapports.
2. Utiliser les dépôts Maven existants `maven-snapshots` et `maven-releases` (déclarés dans `distributionManagement` du `pom.xml`).
3. Passer `NEXUS_READY` à `'true'` dans le `Jenkinsfile`, puis relancer le pipeline.

Résultat attendu dans Nexus (*Browse*) : le jar dans `maven-snapshots`, et les archives `jacoco-report.zip` et `surefire-reports.zip` dans `reports/countryservice3/<numéro du build>/`.

## Dépannage

| Problème | Cause probable | Solution |
|---|---|---|
| `No plugin found for prefix 'sonar'` | Plugin Sonar non déclaré dans le settings Maven | Utiliser le nom complet `org.sonarsource.scanner.maven:sonar-maven-plugin:sonar` |
| `Unknown lifecycle phase "\"` | Commande multi-lignes copiée dans CMD | Mettre la commande sur une seule ligne, sans `\` |
| `Not authorized` (Sonar) | Token invalide | Générer un nouveau token |
| `Port 8082 was already in use` | Une autre instance tourne | Arrêter l'ancienne instance |
| Les tests échouent à cause de MySQL | Base MySQL absente | Utiliser H2 (`src/test/resources`) |
| `localhost:9000` inaccessible depuis Jenkins (WSL) | Réseau WSL / Docker | Utiliser `http://host.docker.internal:9000` |
| Nexus inaccessible | Démarrage lent ou mauvais port | Attendre 2 à 5 min, vérifier `docker ps` |

## Auteur

Fida Yahyaoui — [github.com/fidayahyaoui7](https://github.com/fidayahyaoui7)
