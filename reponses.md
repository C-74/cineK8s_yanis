# Examen CinéK8s — BELLAHCENE Yanis

## Partie 1
**Q1.1**  
Il lit la propriété "movie.url", et pour surcharger avec la variable d'environnement, il faudra mettre MOVIE_URL
___
**Q1.2**
- a : 422 UNPROCESSABLE_ENTITY
- b : 409 CONFLICT
- c : 503 SERVICE_UNAVAILABLE  
___
**Q1.3**

Ligne yaml : include: readinessState,movie

Explication : si la dépendance movie est mise dans la liveness, sa chute fait redémarrer les pods ticket en boucle. En readiness, le pod est juste retiré du Service sans restart et revient seul quand movie revient.
___
**Q1.4**

| Endpoint | Probe(s) Kubernetes qui l'utilisent | Conséquence d'un **échec** de la probe                  |
|----------|-------------------------------------|---------------------------------------------------------|
| `/actuator/health/liveness` | startupProbe et livenessProbe       | Le kubelet redémarre le conteneur                       |
| `/actuator/health/readiness` | readinessProbe                      | Le Pod est retiré des endpoints du Service sans restart |

`graceful` laisse finir les requêtes en cours avant d'arrêter le conteneur, pour éviter les coupures pendant un rolling update.
___
## Partie 2
```
c74@LEGION-C74:~$ curl -s -X POST localhost:8086/api/tickets   -H 'Content-T
ype: application/json'   -d '{"movieId":2,"seats":3}' | jq
{
  "id": 1,
  "movieId": 2,
  "movieTitle": "Le Seigneur des Pods",
  "seats": 3,
  "total": 36.00,
  "createdAt": "2026-10-08T10:30:15.594449992Z"
}
c74@LEGION-C74:~$ curl -s localhost:8086/actuator/health/readiness | jq
{
  "status": "UP",
  "components": {
    "movie": {
      "status": "UP"
    },
    "readinessState": {
      "status": "UP"
    }
  }
}
```
**Q2.1**  
Pour changer le port sans modifier le fichier. Spring lit SERVER_PORT à la place de server.port. Et c'est grâce au relaxed binding et la surcharge avec la variable d'environnement qui permet cela
___
**Q2.2**  
C'est normal car il attend la dépendance "movie" qui n'est pas encore démarrée. Le pod reste Running sans restart et passera UP seul quand movie répondra.
___
## Partie 3

**Q3.1**
On copie pom.xml avant src/ pour profiter du cache Docker. Les dépendances changent rarement, donc la couche dependency:go-offline est réutilisée. Si on modifie une ligne java, seul le COPY src + package est rejoué, pas le téléchargement Maven.
___
**Q3.2**
"-Xmx512m" fixe une taille rigide : trop grande → OOMKilled, trop petite → mémoire gaspillée. "-XX:MaxRAMPercentage=75" calcule le heap à 75% de la mémoire vue par le conteneur, donc ça s'adapte aux requests/limits K8s et laisse 25% pour le non-heap (threads, metaspace, OS).
___
**Q3.3**
Kubernetes n'attend pas, il n'y a pas de depends_on. Si ticket démarre avant movie, sa readiness (movie DOWN) échoue → Pods 0/1 Running mais hors Service, sans trafic et sans restart (liveness reste UP). Dès que movie est UP, ticket passe 1/1 tout seul.
___
## Partie 4
```
c74@LEGION-C74:~/Kubernetes$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
movie-57ccc8b45f-q5lb8    1/1     Running   0          2m53s
movie-57ccc8b45f-vbkcq    1/1     Running   0          2m53s
ticket-794545579f-jwzjt   1/1     Running   0          2m53s
ticket-794545579f-ldlld   1/1     Running   0          2m53s
c74@LEGION-C74:~/Kubernetes$ kubectl get endpoints movie ticket
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME     ENDPOINTS                         AGE
movie    10.244.0.7:8080,10.244.0.8:8080   52s
ticket   10.244.0.6:8080,10.244.0.9:8080   52s
c74@LEGION-C74:~/Kubernetes$ kubectl exec deploy/ticket -- wget -qO- http://movie:8080/api/movies/whoami
{"hostname":"movie-57ccc8b45f-q5lb8","environment":"kubernetes"}c74@LEGION-C74:~/Kubernetes$ kubectl exec deploy/ticket -- wget -qO- http://localhost:8080/actuator/health/readiness
{"status":"UP","components":{"movie":{"status":"UP"},"readinessState":{"status":"UP"}}}c74@LEGION-C74:~/Kubernetes$ kubectl port-forward svc/ticket 8082:8080 &
curl -s -X POST localhost:8082/api/tickets -H 'Content-Type: application/json' \
  -d '{"movieId":2,"seats":2}' | jq
kill %1
[1] 12126
{
  "id": 2,
  "movieId": 2,
  "movieTitle": "Le Seigneur des Pods",
  "seats": 2,
  "total": 24.00,
  "createdAt": "2026-10-08T11:06:16.830956480Z"
}
```

**Q4.1** 
kubectl apply -f k8s/ traite par ordre alphabétique. Les préfixes 00-, 10-, 20- forcent : Namespace d'abord, puis ConfigMaps, puis Deployments/Services. Sinon les Pods arriveraient avant le Namespace ou avant la ConfigMap → CreateContainerConfigError / namespace inexistant.
___
**Q4.2**
C'est la startupProbe (GET /liveness, toutes les 2s, 30 essais). Non, pas une anomalie : Spring met 10-40s à démarrer, la startup suspend liveness/readiness pendant ce temps. Le 0/1 = démarrage en cours.
___
**Q4.3**
Avec Always, kubelet irait toujours chercher sur Docker Hub, alors que tes images sont locales à Minikube (minikube image load). Résultat : ErrImagePull / ImagePullBackOff. IfNotPresent dit d'utiliser l'image du nœud si elle existe.
___
## Partie 5
```
c74@LEGION-C74:~/Kubernetes$ curl -s http://cinema.local/api/movies | jq '.[].title'
"Pod Fiction"
"Le Seigneur des Pods"
"Docker Wars"
"Rollback to the Future"
c74@LEGION-C74:~/Kubernetes$ curl -s -X POST http://cinema.local/api/tickets -H 'Content-Type: application/json' -d '{"movieId":3,"seats":10}' | jq
{
  "id": 1,
  "movieId": 3,
  "movieTitle": "Docker Wars",
  "seats": 10,
  "total": 90.00
}
c74@LEGION-C74:~/Kubernetes$ for i in $(seq 1 6); do curl -s http://cinema.local/api/movies/whoami | jq -r .hostname; done
movie-57ccc8b45f-vbkcq
movie-57ccc8b45f-q5lb8
movie-57ccc8b45f-q5lb8
movie-57ccc8b45f-vbkcq
movie-57ccc8b45f-vbkcq
movie-57ccc8b45f-q5lb8
c74@LEGION-C74:~/Kubernetes$ curl -s -o /dev/null -w "%{http_code}\n" http://cinema.local/actuator/health
404
```
**Q5.1**
Deux pods distincts ont répondu (movie-57ccc8b45f-vbkcq et movie-57ccc8b45f-q5lb8). C'est le Service movie (ClusterIP) qui répartit en round-robin entre ses endpoints.
___
**Q5.2**
Avec pathType: Exact sur /api/movies, seul /api/movies pile matcherait. GET /api/movies/1 tomberait en 404 car pas de règle exacte. Prefix route /api/movies + tout le sous-arbre (/api/movies/1, /whoami).
___
**Q5.3**
On obtient le code HTTP 404, et c'est souhaitable car l'Ingress n'expose que /api/movies et /api/tickets. /actuator/health n'a pas de règle → 404 du controller nginx, donc pas d'exposition inutile des endpoints internes.
___
## Partie 6
Prédictions Q6.1 :
- (a) READY 0/1, RESTARTS 0
- (b) endpoints ticket vide
- (c) 503 via Ingress
- (d) liveness UP
___
**Q6.1**
1. movie à 0 → MovieClient.ping() échoue → Health movie DOWN.
2. readiness ticket (readinessState,movie) passe DOWN.
3. kubelet retire les Pods ticket des endpoints → ticket vide → Ingress/Service répond 503.
4. liveness reste UP → pas de restart → RESTARTS 0. Au scale 2, tout repasse UP seul.
___
**Q6.2**

| # | Statut observé | Commande de diagnostic | Cause exacte | Correction apportée |
|---|----------------|------------------------|--------------|---------------------|
| 1 | `ErrImagePull` / `ImagePullBackOff` | `kubectl describe pod -l app=ticket-debug` (Events) | `imagePullPolicy: Always` : cherche l'image sur Docker Hub alors qu'elle est locale à Minikube | `imagePullPolicy: IfNotPresent` |
| 2 | `CreateContainerConfigError` | `kubectl describe pod -l app=ticket-debug` (Events) + `kubectl get cm` | `configMapRef: ticket-configmap` n'existe pas, la vraie s'appelle `ticket-config` | `name: ticket-config` |
| 3 | `Running` mais `0/1` | `kubectl describe pod -l app=ticket-debug` (Events : `Readiness probe failed`) | `readinessProbe` sur `port: 8081` alors que l'app écoute sur `8080` | `port: 8080` (ou `port: http`) |

**Q6.3**
La ConfigMap est injectée en variables d'environnement, donc lues uniquement au démarrage du conteneur. Modifier la ConfigMap ne redémarre rien seul : il faut `kubectl rollout restart deploy/movie` pour recréer les Pods avec les nouvelles valeurs.
___
## Partie 7

**Q7.1**
`http://movie:8080` se résout via CoreDNS vers la ClusterIP du Service movie. Ensuite kube-proxy redirige vers un seul Pod prêt (un des endpoints). Le ticket ne connaît donc jamais les IP des Pods directement.
___
**Q7.2**
Les tickets sont stockés dans une liste en mémoire, propre à chaque Pod ticket. Selon le Pod touché via le Service, la liste varie. En cas de `kubectl delete pod`, son contenu est perdu. Pour un vrai partage, il faut une base externe (Postgres/Redis) commune à tous les réplicas.
___
**Q7.3**
Avec un Deployment, le ReplicaSet recrée automatiquement un Pod de remplacement et la donnée movie revient (données statiques au démarrage). Avec un Pod nu, rien ne le recrée : le service est définitivement perdu.
