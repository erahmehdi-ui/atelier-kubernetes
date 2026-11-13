# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :**
**Groupe :**

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > Il y a 3 nœuds. k3d-lab-server-0 a le rôle control-plane : il héberge l’API Server, le scheduler, le controller manager et etcd, donc il décide et pilote. k3d-lab-agent-0 et k3d-lab-agent-1 sont les workers, qui exécutent les pods.
2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   >docker ps montre un conteneur par nœud, plus k3d-lab-serverlb (le load balancer, 8081 vers 80). Un nœud k3d est un conteneur Docker qui exécute k3s, une distribution Kubernetes légère. Le cluster multi-machines est donc simulé sur une seule VM.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   >Le create deployment crée un Deployment web, qui crée un ReplicaSet, qui crée un Pod. Le expose ajoute un Service web de type ClusterIP sur le port 80.
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   >Le pod est supprimé (Terminating) et un nouveau pod apparaît immédiatement, avec un nom différent (nouveau suffixe aléatoire). Le ReplicaSet maintient le nombre de réplicas demandé (1), et en recrée un dès qu’il en manque. C’est l’auto-réparation.

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   >Non, le message n’a pas survécu. Le fichier modifié était dans la couche inscriptible du conteneur, qui disparaît avec lui. Le nouveau pod repart de l’image d’origine : les conteneurs sont éphémères.
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   >port-forward svc/web se rattache en réalité à un pod précis, choisi au lancement. Ce pod ayant été supprimé, le tunnel est coupé (voir /tmp/web.log). Il faut le relancer pour qu’il se rattache au nouveau pod.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   >create est impératif : il crée l’objet et échoue s’il existe déjà. apply est déclaratif : on décrit l’état désiré dans un fichier YAML, et Kubernetes crée l’objet ou le met à jour selon le diff. apply est rejouable et versionnable dans Git, donc adapté à la production.
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   >Le selector désigne, par leurs labels, les pods qui reçoivent le trafic du Service. S’il ne correspond à aucun pod, le Service n’a aucun endpoint (kubectl get endpoints site affiche <none>). Il existe, mais les requêtes n’aboutissent pas.

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```
NAME                   READY   STATUS    RESTARTS   AGE   IP           NODE               NOMINATED NODE   READINESS GATES
site-75775484b-7mtvn   1/1     Running   0          15m   10.42.0.6    k3d-lab-agent-1    <none>           <none>
site-75775484b-hv5r5   1/1     Running   0          24s   10.42.2.5    k3d-lab-agent-0    <none>           <none>
site-75775484b-vdgvl   1/1     Running   0          24s   10.42.1.10   k3d-lab-server-0   <none>           <none>

Le scheduler a réparti les 3 réplicas sur 3 nœuds différents : k3d-lab-agent-1, k3d-lab-agent-0 et k3d-lab-server-0. Le control plane héberge donc aussi un pod, car k3s le laisse planifiable.
   ```
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    >Le nouveau pod (site-fc576997-jjxbr) était en ErrImagePull, puis ImagePullBackOff, en 0/1. La commande kubectl get events --field-selector type=Warning (ou kubectl describe pod, section Events) donne la cause exacte : Failed to pull image "nginx:9.99": ... docker.io/library/nginx:9.99: not found. Le tag n’existe pas.
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    >Oui, le site est resté accessible. Pendant la panne, les 3 pods de la version saine restaient 1/1 Running : le rolling update ne supprime un ancien pod que lorsqu’un nouveau est prêt. Le nouveau pod n’étant jamais prêt, le trafic a continué vers l’ancienne version. Le kubectl rollout undo a ensuite supprimé le pod en erreur et restauré la version précédente.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    >https://zany-fiesta-r4q557w5rr5xfpxrx-8585.app.github.dev/
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    > Oui, `kubectl get deploy mario -o yaml` retrouve le Deployment créé via Portainer. Il est de même nature que ceux créés en ligne de commande (`apiVersion: apps/v1`, `kind: Deployment`), avec éventuellement quelques annotations ou labels ajoutés par Portainer. Portainer et `kubectl` sont deux clients de la même **API Server** du control plane : toute action passe par elle, et l'état du cluster est stocké dans etcd.

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    >

---

**Score obtenu avec `./scripts/check.sh` :** ... / ...
