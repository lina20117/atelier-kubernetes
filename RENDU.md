# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :** EL HADDAJI Lina 
**Groupe :** RS3

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
chaque nœud Kubernetes est un conteneur Docker basé sur l’image rancher/k3s. docker ps montre aussi :

k3d-lab-serverlb (image k3d-proxy), le load balancer qui reçoit le port 8081 ;
éventuellement k3d-lab-tools.

Ce sont donc des « machines » simulées par des conteneurs.
2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   >Tous sont dans le namespace `kube-system` :
- `coredns` : DNS interne (résolution des noms de Services) ;
- `local-path-provisioner` : création des volumes persistants ;
- `metrics-server` : métriques CPU/RAM (`kubectl top`) ;
- `traefik` : Ingress Controller ;
- `svclb-traefik-xxx` : un pod par nœud (DaemonSet) qui expose Traefik ;
- `helm-install-traefik(-crd)` : jobs d'installation, en état `Completed`.
Ce sont les composants système installés par défaut par k3s.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   >L'ancien pod passe en `Terminating` et un nouveau pod apparaît immédiatement (`Pending` → `ContainerCreating` → `Running`). Le nom a changé : le préfixe `web-<hash>` est conservé, mais le suffixe aléatoire est nouveau. C'est un pod différent, pas le même pod redémarré.
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   >Le Deployment pilote un ReplicaSet dont l'état désiré est 1 réplica. Sa boucle de contrôle constate 0 pod au lieu de 1 et en recrée un automatiquement : c'est l'auto-réparation. Kubernetes compare en permanence l'état réel à l'état désiré et corrige l'écart. Les pods sont jetables et interchangeables ; c'est le Service qui fournit une adresse stable vers eux.

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   >La page ne s'affiche plus. `/tmp/web.log` indique une perte de connexion avec le pod (`lost connection to pod` / `pod not found`). Le `port-forward` s'attache à un pod précis au moment de son lancement, même quand on cible le Service. Quand ce pod est supprimé, le tunnel meurt avec lui et ne bascule pas vers le nouveau pod.
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   >Même lancé sur `svc/web`, le `port-forward` ne passe pas vraiment par le Service : au démarrage, kubectl choisit *un pod précis* derrière le Service et ouvre le tunnel directement vers lui. Quand ce pod est supprimé, le tunnel n'a plus de destination et s'arrête (`/tmp/web.log` : `lost connection to pod` / `pod not found`). Il ne bascule pas automatiquement sur le nouveau pod recréé par le ReplicaSet : il faut relancer le tunnel. C'est pour ça qu'en production on utilise un Ingress plutôt qu'un port-forward.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   >`kubectl create` est impératif : il ordonne de créer une ressource et échoue si elle existe déjà (`AlreadyExists`). Il ne sert donc qu'une fois et ne permet pas de mettre à jour.
   > `kubectl apply -f` est déclaratif : il envoie l'état désiré décrit dans un fichier YAML. Kubernetes crée la ressource si elle n'existe pas, sinon il calcule l'écart et ne modifie que ce qui a changé. La commande est idempotente (on peut la relancer sans risque), et comme les fichiers sont versionnés dans Git, on a un historique, des revues de code et un déploiement reproductible.

8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   >Le `selector` (ici `app: site`) indique au Service quels pods cibler : tous les pods qui portent ces labels deviennent ses endpoints, et le trafic est réparti entre eux. Le lien se fait par les labels, pas par les noms, ce qui permet au Service de suivre automatiquement les pods recréés ou ajoutés.
   > S'il ne correspond à aucun pod, le Service existe toujours (il garde son IP et son nom DNS) mais n'a aucun endpoint : `kubectl get endpoints site` affiche `<none>`, aucune requête n'aboutit (connexion refusée ou timeout) et le port-forward échoue. Kubernetes n'affiche aucune erreur : il faut vérifier les endpoints pour s'en rendre compte.


## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   NAME                   READY   STATUS    RESTARTS   AGE   IP          NODE               NOMINATED NODE   READINESS GATES
site-f58fdbd6c-cnz2w   1/1     Running   0          22s   10.42.0.5   k3d-lab-server-0   <none>           <none>
site-f58fdbd6c-frllx   1/1     Running   0          6s    10.42.0.6   k3d-lab-server-0   <none>           <none>
site-f58fdbd6c-lt6z8   1/1     Running   0          14s   10.42.1.7   k3d-lab-agent-0    <none>           <none>

   ```
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
   Le nouveau pod `site-7bf67d9cf9-qz94k` était en `ImagePullBackOff` (`0/1`), après une première phase `ErrImagePull`. La cause apparaît avec `kubectl describe pod <nom-du-pod>`, dans la section *Events* : `Failed to pull image "nginx:9.99"` (`not found`). Le tag `9.99` n'existe pas sur Docker Hub, donc le kubelet ne peut pas télécharger l'image et réessaie avec un délai croissant. `kubectl logs` ne donne rien, car le conteneur n'a jamais démarré.
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    Oui. Pendant la panne, `kubectl get pods` montrait le pod cassé en `0/1 ImagePullBackOff` et les 3 anciens pods (`site-f58fdbd6c-*`) toujours en `1/1 Running`. Avec la stratégie RollingUpdate, Kubernetes crée d'abord un nouveau pod et attend qu'il soit prêt (readinessProbe) avant de supprimer un ancien. Comme le nouveau ne devient jamais prêt, le rollout reste bloqué et le Service continue d'envoyer le trafic uniquement aux 3 anciens pods. `kubectl rollout undo deploy/site` a ramené la révision précédente (`successfully rolled out`), et `kubectl diff -f k8s/site/` n'a renvoyé aucun écart : le cluster et le YAML sont de nouveau alignés. Un `kubectl apply -f k8s/site/` a ensuite resynchronisé l'annotation `last-applied-configuration`, que le `undo` impératif n'avait pas mise à jour.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    > https://upgraded-space-winner-qwqxw65vjgghx7xq-8585.app.github.dev/
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    > Oui : `kubectl get deploy mario -o yaml` affiche bien le Deployment créé depuis Portainer. Portainer et `kubectl` parlent tous les deux à l'API Server (`kube-apiserver`), le composant du control plane qui sert de point d'entrée unique au cluster. Il authentifie et valide les requêtes, puis enregistre l'état dans la base du cluster (etcd, ou SQLite dans k3s). Portainer n'est qu'un client graphique de cette API, avec les droits d'un ServiceAccount, puisqu'il tourne dans le cluster. Le Deployment est donc un objet identique à ceux créés avec kubectl. La seule différence, ce sont des labels et annotations `io.portainer.kubernetes.application.*` qu'il ajoute pour suivre ses applications.


## Bonus (facultatif)

14. Bonus réalisés et remarques :
    
    > - **A (Ingress)** : Ingress `site` (`ingressClassName: traefik`, chemin `/` → Service `site:80`). `curl localhost:8081` renvoie bien la page personnalisée. Contrairement au port-forward, l'Ingress passe par les endpoints du Service, mis à jour en continu, donc le site reste accessible quand des pods sont supprimés.
    > - **B (Probes et ressources)** : `readinessProbe` HTTP sur `/` (port 80), requests `50m` CPU / `32Mi` et limits `100m` / `64Mi` (QoS `Burstable`). La probe a contribué à la résilience pendant la panne `nginx:9.99` : un pod n'est ajouté au Service qu'une fois prêt. Avec une probe sur `/nope`, nginx répond 404 : les pods restent `Running` mais `0/1`, sont retirés des endpoints et ne reçoivent plus de trafic (sans redémarrage, contrairement à une livenessProbe).
    > - Ajout d'une `topologySpreadConstraint` (`kubernetes.io/hostname`, `maxSkew: 1`) pour forcer la répartition des réplicas sur plusieurs nœuds.
---

**Score obtenu avec `./scripts/check.sh` :** .18 / 18.
