# Compte rendu TP PetitPanier
Systèmes utilisés : Windows, Docker Desktop, Kind, Kubernetes, Git

## Palier 1
1.1: Le Client est l'interface de ligne de commande (CLI) sur ton poste, le Server est le moteur Docker daemon qui exécute les conteneurs. Si le Server est absent, Docker n'est pas lancé.
1.2: 8080 est le port de la machine hôte (ton PC) et 80 est le port interne du conteneur.
1.3: Une image est un modèle figé en lecture seule (le patron), tandis qu'un conteneur est une instance active et modifiable de cette image.

## Palier 2
2.2: Non, EXPOSE 8080 sert uniquement de documentation. Il faut obligatoirement mapper les ports avec l'option -p lors du lancement.
2.3: Le "." à la fin indique le contexte de build (dossier courant). On utilise -p 8080:8080 car l'image non privilégiée écoute par défaut sur 8080.
2.4: Le conteneur en cours d'exécution tournait encore avec l'ancienne image 1.0. Il a fallu supprimer l'ancien conteneur et relancer un conteneur basé sur la version 2.0.

## Palier 3
3.2: Up indique que le conteneur tourne, Up (healthy) signifie qu'il a passé avec succès le test de santé (healthcheck). C'est le bloc healthcheck du fichier qui produit cela.
3.3: La seconde échoue car "localhost" dans un conteneur désigne le conteneur lui-même et non la machine hôte. "app" représente le nom du service défini dans compose.yaml servant de nom d'hôte DNS interne.

## Palier 4
4.1: Kind load est nécessaire car le cluster Kubernetes tourne dans un conteneur isolé et ne voit pas directement les images locales tant qu'on ne les y charge pas. En entreprise, on utilise un registre distant.
4.2: Les labels servent à identifier les objets. Le selector du Service utilise le label pour cibler les Pods. Si le selector était différent, le Service ne trouverait aucun Pod et les ENDPOINTS seraient vides.
4.3: L'adresse dans ENDPOINTS est l'IP interne du Pod dans le cluster. Le port-forward est nécessaire pour créer un pont réseau entre ton PC (localhost:8080) et le Service Kubernetes.

## Palier 5
5.1: Sur un cluster local à un seul nœud, les 3 Pods tournent sur le même nœud physique. Sur un cluster multi-nœuds, cela garantit la haute disponibilité.
5.2: C'est le Deployment qui a recréé automatiquement un nouveau Pod pour maintenir l'état désiré (3 réplicas).
5.3: "web" correspond au nom du conteneur ciblé dans le déploiement. rollout history affiche 2 révisions.

## Palier 6 tableau de diagnostic
| Panne | Symptôme | Hypothèse | Commande de preuve et résultat | Correction |
|---|---|---|---|---|
| promo-1 | ImagePullBackOff / ErrImagePull | Le nom ou le tag de l'image est erroné ou absent du cluster. | kubectl describe pod promo-1-... | Corriger le nom ou tag de l'image dans le manifeste. |
| promo-2 | CrashLoopBackOff | L'application plante au démarrage à cause d'une commande invalide. | kubectl logs promo-2-... --previous | Corriger la commande ou la variable d'environnement. |
| promo-3 | Pods Running mais site inaccessible | Le selector du Service ne correspond pas aux labels des Pods. | kubectl get endpoints promo-3 | Aligner le selector du Service avec les labels du Deployment. |

6.2 (--previous) : Permet de récupérer les logs de l'instance précédente du conteneur qui vient d'échouer en boucle.
6.3 (pourquoi Running mais inaccessible) : Les Pods fonctionnaient mais le Service ne trouvait aucun endpoint à cause d'une erreur de correspondance de labels.

## Palier 7
| Étape | Commande | Si échec |
|---|---|---|
| build | docker build -t petitpanier:2.0 . | L'image ne peut pas être générée (erreur de syntaxe ou fichier manquant). |
| test | docker run --rm ... | L'application ne répond pas correctement, le code n'est pas validé. |
| publish | docker push ... (ou kind load) | L'image ne peut pas être envoyée sur le registre distant. |
| deploy | kubectl apply -f k8s/ | Les manifestes échouent à s'appliquer, l'ancienne version reste en service. |

7.1 : Un pipeline tague l'image avec un ID de commit unique pour garantir la traçabilité et l'immutabilité, contrairement à "latest" qui change tout le temps.
Workflow n8n : Webhook GitLab (déclencheur) -> If/Else (statut = failed) -> Action Slack (#petitpanier-ops). Les secrets sont stockés dans les variables sécurisées de n8n.
