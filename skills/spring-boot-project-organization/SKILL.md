---
name: spring-boot-project-organization
description: Guide la définition de l’architecture d’un projet Java Spring Boot et l’ajout de nouvelles fonctionnalités en appliquant conjointement DDD, architecture hexagonale (ports et adaptateurs) et Clean Architecture. À utiliser pour délimiter les modules métier, modéliser les invariants, répartir les responsabilités entre Domain, Application et Infrastructure, définir les ports et adaptateurs, vérifier les dépendances et prévoir les tests, sans abstractions inutiles ni rupture des contrats et comportements d’un dépôt existant.
---

# Organiser un projet Java Spring Boot

## Démarche

1. **Cadrer.** Architecture cible, ajout de fonctionnalité ou adaptation d’un dépôt existant ? Identifie les besoins métier, les invariants et les interactions externes concernés. Dans un dépôt, pars de ses fichiers et conventions, et identifie ce qu’il faut préserver : contrats publics, validations, sécurité, comportements.
2. **Appliquer les trois approches sans surdimensionner.** Prends DDD, l’architecture hexagonale et Clean Architecture comme cible commune : modèle métier explicite, cœur indépendant et interactions externes par ports et adaptateurs. Ajoute une couche ou une abstraction seulement si elle clarifie une frontière utile, et nomme son coût. Ne duplique pas les concepts communs aux trois approches. Si l’existant s’en écarte, explicite les écarts et propose une adaptation ciblée sans refonte non demandée ni rupture de contrat ou de comportement. Si la cible et les contraintes existantes sont incompatibles, expose le conflit et les compromis ; demande une décision utilisateur si le choix affecte significativement l’architecture ou les contrats, sans dégrader silencieusement les contraintes existantes.
3. **Placer par domaine.** Modules métier d’abord, puis seulement les couches utilisées. Une nouvelle fonctionnalité rejoint le module existant qu’elle concerne et suit ses conventions compatibles avec la cible ; crée un module seulement pour un domaine distinct. Définis ses invariants dans le domaine, son orchestration dans un cas d’utilisation, et ses échanges externes dans les ports et adaptateurs nécessaires.
4. **Illustrer un flux.** Suis une requête de l’adaptateur entrant (par exemple REST) au cas d’utilisation et au domaine ; ajoute un appel sortant seulement s’il est nécessaire. Distingue le flux d’exécution, qui peut aller vers un adaptateur, des dépendances du code, toujours dirigées vers le cœur.
5. **Tester.** Prévois des tests ciblés sur les frontières touchées. Si tu implémentes, exécute les vérifications disponibles et rapporte résultats et limites.
6. **Livrer honnêtement.** Sépare l’existant observé de tes recommandations ; n’invente ni version, ni module, ni test. Termine par les compromis et ce qui reste à décider.

## Les trois approches

Elles constituent ensemble l’architecture cible et se recouvrent sur l’isolation du métier ; les combiner n’exige ni couches dupliquées ni interfaces multipliées.

| Approche | Question principale | Responsabilité |
| --- | --- | --- |
| DDD | Comment modéliser le métier et ses frontières ? | Langage métier, modèles, invariants, limites de cohérence et de contexte. |
| Clean Architecture | Où placer les responsabilités et dans quel sens dépendent-elles ? | Séparer Domain, Application et Infrastructure ; orienter les dépendances vers le cœur. |
| Ports & Adapters (hexagonale) | Comment le cœur dialogue-t-il avec l’extérieur ? | Ports entrants pour ses capacités, ports sortants pour ses besoins, reliés à des adaptateurs. |

- **DDD :** n’utilise que les concepts utiles (entité, value object, invariant, agrégat, bounded context). Un module métier n’est pas automatiquement un bounded context.
- **Clean Architecture :** ce skill simplifie les quatre cercles d’origine en trois couches. Le cœur regroupe Domain et Application. Dépendances : `infrastructure → application → domain`, l’infrastructure pouvant aussi utiliser le domaine ; le cœur ne dépend jamais des adaptateurs.
- **Ports & Adapters :** une classe de cas d’utilisation suffit souvent comme port entrant. Les ports sortants appartiennent au cœur (`application` ou `domain`, sans duplication) ; leurs adaptateurs vivent en `infrastructure`.

## Structure indicative

Arborescence illustrative : crée seulement les packages utiles, sans dossier vide ni branche anticipée. Un module logique peut être un simple package ; une application simple peut rester plus plate.

```text
<package-racine>/
├── Application.java
└── <module-metier>/
    ├── domain/
    │   ├── model/
    │   ├── service/          # si une règle ne relève d’aucun modèle
    │   └── exception/
    ├── application/
    │   ├── command/          # si l’entrée du cas d’utilisation est composite
    │   ├── usecase/
    │   ├── port/in/          # si une interface entrante est justifiée
    │   ├── port/out/
    │   └── exception/
    └── infrastructure/
        ├── persistence/      # entity, mapper, repository, adapter
        ├── client/           # si un système externe est appelé
        ├── configuration/    # @Bean des cas d’utilisation, transactions
        └── web/
            └── dto/          # si l’entrée ou la sortie HTTP n’est pas un type simple
```

- `domain` : modèle et invariants ; dépend seulement du JDK (ni Spring, ni JPA, ni HTTP).
- `application` : cas d’utilisation et ports sortants ; dépend uniquement de `domain` et du JDK, ni de Spring, ni de l’infrastructure, ni de HTTP ou JPA.
- `infrastructure` : contrôleurs, DTO, persistance (entités JPA comprises), clients, configuration et traduction des erreurs techniques.
- Entre modules : contrats explicites, pas d’accès aux détails internes.

## Spécificités Spring Boot

- `@SpringBootApplication` se place dans le package racine pour que tous les modules soient analysés.
- Les cas d’utilisation ne portent aucune annotation : ils sont déclarés par `@Bean` dans `infrastructure/configuration`. Adaptateurs et mappers peuvent être des `@Component` en infrastructure.
- Place une transaction autour des cas d’utilisation qui doivent être atomiques : charger puis enregistrer via un port ne l’est pas à lui seul. Applique-la depuis l’infrastructure, par exemple par un décorateur utilisant `TransactionTemplate` déclaré dans la configuration ; une exception non vérifiée l’annule. Une transaction n’empêche pas les mises à jour concurrentes : prévois `@Version` ou un verrou si le risque existe.
- Aucun type Spring ni JPA dans les signatures des ports (`Pageable`, `Page`, `ResponseEntity`, entités JPA) : définis des types propres au cœur.
- Le cœur ne lit ni `ApplicationContext`, ni `@Value`, ni le contexte de sécurité : configuration et identité de l’utilisateur lui arrivent en paramètre ou par un port.
- Bean Validation (`@Valid`) sur les DTO web ; les invariants métier restent dans le domaine.
- Les exceptions du cœur sont traduites en réponses HTTP dans l’adaptateur web (par exemple `@RestControllerAdvice`).
- ArchUnit ou Spring Modulith peuvent vérifier les dépendances si le risque le justifie ; ne les ajoute pas sans demande.

## Tests

Stratégie par couche, sans affirmer leur existence dans un dépôt ni imposer d’outil :

- domaine : tests unitaires des invariants, sans Spring ;
- application : scénarios et erreurs, avec doublures des ports, instanciés sans conteneur Spring ;
- adaptateurs : contrats web, validations et traduction d’erreurs ; intégration de la persistance et des clients avec les technologies réelles ;
- assemblage : quelques tests du câblage et des parcours critiques ; règles d’architecture si le risque le justifie.

## Exemple facultatif

[exemples-spring-boot.md](exemples-spring-boot.md) montre un exemple générique commenté. Il ne remplace pas les principes ci-dessus et n’impose ni domaine métier, ni arborescence exacte.
