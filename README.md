# Elastic Agent Buildpack

Buildpack Scalingo pour installer Elastic Agent en mode colocalisé avec une autre application à monitorer.

Fork du [buildpack `elastic-beats-buildpack`](https://github.com/SocialGouv/elastic-beats-buildpack), pour résoudre un
dépassement de la limite de taille d'image Scalingo (2048 Mo) (cf. [issue #2](https://github.com/SocialGouv/elastic-beats-buildpack/issues/2)).

Le buildpack original extrait l'intégralité du tarball Linux d'Elastic Agent (~1,7 Go pour la version 9.4.x), incluant des composants serveur non nécessaires
à un agent de collecte standard (~550 Mo : `cloudbeat`, `fleet-server`, `apm-server`…). Combiné à d'autres buildpacks, l'image finale peut dépasser la limite
Scalingo :

```
!   <app-name> image exceeds the limit of 2048MB - (2269MB)
```

Ce fork utilise à la place les **images OCI officielles d'Elastic** (`elastic-agent-slim`, `elastic-agent`, `elastic-agent-complete`) via `crane export`, ce qui
permet de sélectionner précisément les composants installés via le concept de **flavor** et de rester aligné sur la définition officielle d'Elastic.

## Variables d'environnement

### `ELASTIC_AGENT_VERSION`

Version d'Elastic Agent à installer.

Exemple : `9.4.5`

### `ELASTIC_AGENT_LOG_LEVEL`

Niveau de log de l'Elastic Agent et de ses sous-composants (metricbeat, filebeat, etc.) sur stdout.

En mode `container`, l'Agent écrit tous ses logs sur stdout, y compris les rapports de métriques internes
de libbeat (`"Total metrics"`), ce qui peut générer du bruit dans les consoles de logs des plateformes PaaS.

Ce paramètre est propagé à `elastic-agent.yml` au démarrage et s'applique à tous les sous-composants gérés par l'Agent.

| Valeur               | Description                                                                 |
|----------------------|-----------------------------------------------------------------------------|
| `warning` *(défaut)* | Seuls les warnings et erreurs sont affichés. Recommandé en production.      |
| `info`               | Logs détaillés. Utile pour diagnostiquer un problème de démarrage ou Fleet. |
| `debug`              | Logs très verbeux. À n'utiliser que ponctuellement.                         |

### `ELASTIC_AGENT_FLAVOR`

Flavor d'Elastic Agent à utiliser. Correspond aux distributions officielles publiées par Elastic pour les environnements conteneurisés.

| Valeur             | Image OCI utilisée        | Description                                                                                     |
|--------------------|---------------------------|-------------------------------------------------------------------------------------------------|
| `basic` *(défaut)* | `elastic-agent-slim`      | Flavor Basic : composants de collecte uniquement. Recommandé pour réduire la taille de l'image. |
| `servers`          | `elastic-agent`           | Flavor Servers : inclut les composants serveur (fleet-server, apm-server, cloudbeat, etc.).     |
| `complete`         | `elastic-agent-complete`  | Flavor Complete : tous les composants, y compris les intégrations supplémentaires.              |

#### Impact sur la taille

Avec Elastic Agent 9.4.x :
- `basic` : ~1,1 Go (économie d'environ 550 Mo par rapport au flavor Servers)
- `servers` : ~1,7 Go
- `complete` : plus volumineux encore

Le flavor `basic` est recommandé pour les cas d'usage de collecte de métriques via Fleet.

## Utilisation

Dans le fichier `.buildpacks` de l'application, ajouter ce buildpack **avant** le buildpack à monitorer :

```
https://github.com/France-Travail/elastic-agent-buildpack
```

## Configuration Fleet

Pour que l'Elastic Agent collecte des métriques et les envoie dans Kibana, il faut configurer Fleet. Voici le guide complet.

### 1. Créer l'Agent Policy

Dans Kibana : **Management → Fleet → onglet Agent policies → Create agent policy**

- Nom : `<app>-scalingo-<env>` (adapter selon l'environnement : `prod`, `staging`…)
- Namespace : `default`
- Désactiver **Collect system logs and metrics** si l'option est proposée (Fleet ajouterait sinon l'intégration System automatiquement).

Si la policy existe déjà avec `system-1` :
- Fleet → Agent policies → `<app>-scalingo-<env>` → onglet Integrations
- Sur `system-1` : **Actions … → Delete integration**
- Cette intégration supervise le conteneur Scalingo, pas l'application — elle n'est pas nécessaire ici.

### 2. Ajouter l'intégration de l'application à monitorer

Fleet → Agent policies → `<app>-scalingo-<env>` → onglet Integrations → **Add integration**

Rechercher l'intégration correspondant à l'application et la configurer selon ses propres instructions.

- Nom de l'integration policy : `<app>-<service>`
- Rattacher à l'Agent Policy : `<app>-scalingo-<env>`

> Elastic recommande **Metrics (Elastic Agent)** pour les dashboards modernes des intégrations.
> Ne pas activer simultanément Metrics (Elastic Agent) et Metrics (Stack Monitoring).

### 3. Vérifier le monitoring de l'Elastic Agent lui-même

Fleet → Agent policies → `<app>-scalingo-<env>` → onglet **Settings** → section **Agent monitoring**

Activer (au moins pendant la mise en place) :
- **Agent logs** : ON
- **Agent metrics** : ON

C'est indépendant de `system-1`. Fleet crée automatiquement l'intégration nécessaire au monitoring de l'Elastic Agent.

Architecture finale de la policy :
```
System integration        supprimée
Elastic Agent monitoring  conservé (logs + metrics)
<app> integration         conservée
```

### 4. Récupérer FLEET_URL

Kibana : **Management → Fleet → onglet Settings → Fleet Server hosts**

Sur Elastic Cloud, l'URL est normalement déjà renseignée automatiquement. Copier l'URL du Fleet Server.

Exemple : `https://xxxxxxxx.fleet.eu-west-1.aws.elastic-cloud.com:443`

Dans Scalingo : `FLEET_URL=<URL copiée>`

> Pas besoin de créer un nouveau Fleet Server : celui déjà utilisé dans le déploiement Elastic Cloud convient.

### 5. Créer/récupérer le token d'enrollment

Kibana : **Management → Fleet → onglet Enrollment tokens → Create enrollment token**

- Nom : `<app>-scalingo-<env>-enrollment`
- Agent policy : `<app>-scalingo-<env>`

Une fois créé, utiliser l'icône **Show token** et copier le secret.

> Fleet crée déjà automatiquement un enrollment token à la création d'une Agent Policy.
> Avoir un token explicitement nommé pour ce déploiement est plus propre.

Dans Scalingo : `FLEET_ENROLLMENT_TOKEN=<secret>`

### 6. Générer un ID stable pour l'Agent

À faire une seule fois localement :

```bash
uuidgen | tr '[:upper:]' '[:lower:]'
```

Dans Scalingo : `ELASTIC_AGENT_ID=<UUID>`

Un ID stable évite qu'un redéploiement Scalingo crée à chaque fois une nouvelle identité Fleet.

> ⚠️ Cet ID doit être **unique par instance**. Chaque environnement (prod, staging, perf…) doit avoir son propre UUID.

### 7. Générer le replacement token

À faire localement :

```bash
openssl rand -hex 32
```

Dans Scalingo : `FLEET_REPLACE_TOKEN=<secret généré>`

La combinaison `ELASTIC_AGENT_ID` + `FLEET_REPLACE_TOKEN` permet à une nouvelle instance de remplacer proprement l'ancienne après recréation du conteneur
Scalingo (mécanisme documenté par Elastic).

### 8. Variables Scalingo complètes

```bash
# Buildpack
ELASTIC_AGENT_VERSION=9.4.5
ELASTIC_AGENT_FLAVOR=basic

# Fleet enrollment
FLEET_ENROLL=1
FLEET_URL=https://<fleet-server>
FLEET_ENROLLMENT_TOKEN=<token Fleet>

# Identité stable (à générer une fois, cf. étapes 6 et 7)
ELASTIC_AGENT_ID=<UUID fixe>
FLEET_REPLACE_TOKEN=<secret fixe>

# Tags (pour filtrer dans Fleet)
ELASTIC_AGENT_TAGS=scalingo,<app>
```

> `STATE_PATH` n'est pas une variable Scalingo : elle est passée directement dans le `Procfile` de l'application via
> `STATE_PATH="/app/data/elastic-agent-state"`. Le répertoire doit être créé avant le lancement de l'agent.

### 9. Contrôler l'enrollment après déploiement

Kibana : **Management → Fleet → onglet Agents**

L'Agent doit apparaître avec :
- Policy : `<app>-scalingo-<env>`
- Status : **Healthy**

Si l'Agent apparaît mais reste **Unhealthy**, consulter son onglet **Logs**.

### 10. Vérifier que les métriques arrivent

Une fois l'Agent Healthy :

Kibana → **Integrations → onglet Installed integrations → `<app>` → Assets**

Les dashboards fournis avec l'intégration sont disponibles. Elastic les installe automatiquement lorsque l'intégration est ajoutée à une Agent Policy.
