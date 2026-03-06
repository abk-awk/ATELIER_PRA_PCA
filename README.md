------------------------------------------------------------------------------------------------------
ATELIER PRA/PCA
------------------------------------------------------------------------------------------------------
L’idée en 30 secondes : Cet atelier met en œuvre un **mini-PRA** sur **Kubernetes** en déployant une **application Flask** avec une **base SQLite** stockée sur un **volume persistant (PVC pra-data)** et des **sauvegardes automatiques réalisées chaque minute vers un second volume (PVC pra-backup)** via un **CronJob**. L’**image applicative est construite avec Packer** et le **déploiement orchestré avec Ansible**, tandis que Kubernetes assure la gestion des pods et de la disponibilité applicative. Nous observerons la différence entre **disponibilité** (recréation automatique des pods sans perte de données) et **reprise après sinistre** (perte volontaire du volume de données puis restauration depuis les backups), nous mesurerons concrètement les RTO et RPO, et comprendrons les limites d’un PRA local non répliqué. Cet atelier illustre de manière pratique les principes de continuité et de reprise d’activité, ainsi que le rôle respectif des conteneurs, du stockage persistant et des mécanismes de sauvegarde.
  
**Architecture cible :** Ci-dessous, voici l'architecture cible souhaitée.   
  
![Screenshot Actions](Architecture_cible.png)  
  
-------------------------------------------------------------------------------------------------------
Séquence 1 : Codespace de Github
-------------------------------------------------------------------------------------------------------
Objectif : Création d'un Codespace Github  
Difficulté : Très facile (~5 minutes)
-------------------------------------------------------------------------------------------------------
**Faites un Fork de ce projet**. Si besoin, voici une vidéo d'accompagnement pour vous aider à "Forker" un Repository Github : [Forker ce projet](https://youtu.be/p33-7XQ29zQ) 
  
Ensuite depuis l'onglet **[CODE]** de votre nouveau Repository, **ouvrez un Codespace Github**.
  
---------------------------------------------------
Séquence 2 : Création du votre environnement de travail
---------------------------------------------------
Objectif : Créer votre environnement de travail  
Difficulté : Simple (~10 minutes)
---------------------------------------------------
Vous allez dans cette séquence mettre en place un cluster Kubernetes K3d contenant un master et 2 workers, installer les logiciels Packer et Ansible. Depuis le terminal de votre Codespace copier/coller les codes ci-dessous étape par étape :  

**Création du cluster K3d**  
```
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```
```
k3d cluster create pra \
  --servers 1 \
  --agents 2
```
**vérification de la création de votre cluster Kubernetes**  
```
kubectl get nodes
```
**Installation du logiciel Packer (création d'images Docker)**  
```
PACKER_VERSION=1.11.2
curl -fsSL -o /tmp/packer.zip \
  "https://releases.hashicorp.com/packer/${PACKER_VERSION}/packer_${PACKER_VERSION}_linux_amd64.zip"
sudo unzip -o /tmp/packer.zip -d /usr/local/bin
rm -f /tmp/packer.zip
```
**Installation du logiciel Ansible**  
```
python3 -m pip install --user ansible kubernetes PyYAML jinja2
export PATH="$HOME/.local/bin:$PATH"
ansible-galaxy collection install kubernetes.core
```
  
---------------------------------------------------
Séquence 3 : Déploiement de l'infrastructure
---------------------------------------------------
Objectif : Déployer l'infrastructure sur le cluster Kubernetes
Difficulté : Facile (~15 minutes)
---------------------------------------------------  
Nous allons à présent déployer notre infrastructure sur Kubernetes. C'est à dire, créér l'image Docker de notre application Flask avec Packer, déposer l'image dans le cluster Kubernetes et enfin déployer l'infratructure avec Ansible (Création du pod, création des PVC et les scripts des sauvegardes aututomatiques).  

**Création de l'image Docker avec Packer**  
```
packer init .
packer build -var "image_tag=1.0" .
docker images | head
```
  
**Import de l'image Docker dans le cluster Kubernetes**  
```
k3d image import pra/flask-sqlite:1.0 -c pra
```
  
**Déploiment de l'infrastructure dans Kubernetes**  
```
ansible-playbook ansible/playbook.yml
```
  
**Forward du port 8080 qui est le port d'exposition de votre application Flask**  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
  
---------------------------------------------------  
**Réccupération de l'URL de votre application Flask**. Votre application Flask est déployée sur le cluster K3d. Pour obtenir votre URL cliquez sur l'onglet **[PORTS]** dans votre Codespace (à coté de Terminal) et rendez public votre port 8080 (Visibilité du port). Ouvrez l'URL dans votre navigateur et c'est terminé.  

**Les routes** à votre disposition sont les suivantes :  
1. https://...**/** affichera dans votre navigateur "Bonjour tout le monde !".
2. https://...**/health** pour voir l'état de santé de votre application.
3. https://...**/add?message=test** pour ajouter un message dans votre base de données SQLite.
4. https://...**/count** pour afficher le nombre de messages stockés dans votre base de données SQLite.
5. https://...**/consultation** pour afficher les messages stockés dans votre base de données.
  
---------------------------------------------------  
### Processus de sauvegarde de la BDD SQLite

Grâce à une tâche CRON déployée par Ansible sur le cluster Kubernetes (un CronJob), toutes les minutes une sauvegarde de la BDD SQLite est faite depuis le PVC pra-data vers le PCV pra-backup dans Kubernetes.  

Pour visualiser les sauvegardes périodiques déposées dans le PVC pra-backup, coller les commandes suivantes dans votre terminal Codespace :  

```
kubectl -n pra run debug-backup \
  --rm -it \
  --image=alpine \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "debug",
      "image": "alpine",
      "command": ["sh"],
      "stdin": true,
      "tty": true,
      "volumeMounts": [{
        "name": "backup",
        "mountPath": "/backup"
      }]
    }],
    "volumes": [{
      "name": "backup",
      "persistentVolumeClaim": {
        "claimName": "pra-backup"
      }
    }]
  }
}'
```
```
ls -lh /backup
```
**Pour sortir du cluster et revenir dans le terminal**
```
exit
```

---------------------------------------------------
Séquence 4 : 💥 Scénarios de crash possibles  
Difficulté : Facile (~30 minutes)
---------------------------------------------------
### 🎬 **Scénario 1 : PCA — Crash du pod**  
Nous allons dans ce scénario **détruire notre Pod Kubernetes**. Ceci simulera par exemple la supression d'un pod accidentellement, ou un pod qui crash, ou un pod redémarré, etc..

**Destruction du pod :** Ci-dessous, la cible de notre scénario   
  
![Screenshot Actions](scenario1.png)  

Nous perdons donc ici notre application mais pas notre base de données puisque celle-ci est déposée dans le PVC pra-data hors du pod.  

Copier/coller le code suivant dans votre terminal Codespace pour détruire votre pod :
```
kubectl -n pra get pods
```
Notez le nom de votre pod qui est différent pour tout le monde.  
Supprimez votre pod (pensez à remplacer <nom-du-pod-flask> par le nom de votre pod).  
Exemple : kubectl -n pra delete pod flask-7c4fd76955-abcde  
```
kubectl -n pra delete pod <nom-du-pod-flask>
```
**Vérification de la suppression de votre pod**
```
kubectl -n pra get pods
```
👉 **Le pod a été reconstruit sous un autre identifiant**.  
Forward du port 8080 du nouveau service  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
Observez le résultat en ligne  
https://...**/consultation** -> Vous n'avez perdu aucun message.
  
👉 Kubernetes gère tout seul : Aucun impact sur les données ou sur votre service (PVC conserve la DB et le pod est reconstruit automatiquement) -> **C'est du PCA**. Tout est automatique et il n'y a aucune rupture de service.
  
---------------------------------------------------
### 🎬 **Scénario 2 : PRA - Perte du PVC pra-data** 
Nous allons dans ce scénario **détruire notre PVC pra-data**. C'est à dire nous allons suprimer la base de données en production. Ceci simulera par exemple la corruption de la BDD SQLite, le disque du node perdu, une erreur humaine, etc. 💥 Impact : IL s'agit ici d'un impact important puisque **la BDD est perdue**.  

**Destruction du PVC pra-data :** Ci-dessous, la cible de notre scénario   
  
![Screenshot Actions](scenario2.png)  

🔥 **PHASE 1 — Simuler le sinistre (perte de la BDD de production)**  
Copier/coller le code suivant dans votre terminal Codespace pour détruire votre base de données :
```
kubectl -n pra scale deployment flask --replicas=0
```
```
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":true}}'
```
```
kubectl -n pra delete job --all
```
```
kubectl -n pra delete pvc pra-data
```
👉 Vous pouvez vérifier votre application en ligne, la base de données est détruite et la service n'est plus accéssible.  

✅ **PHASE 2 — Procédure de restauration**  
Recréer l’infrastructure avec un PVC pra-data vide.  
```
kubectl apply -f k8s/
```
Vérification de votre application en ligne.  
Forward du port 8080 du service pour tester l'application en ligne.  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
https://...**/count** -> =0.  
https://...**/consultation** Vous avez perdu tous vos messages.  

Retaurez votre BDD depuis le PVC Backup.  
```
kubectl apply -f pra/50-job-restore.yaml
```
👉 Vous pouvez vérifier votre application en ligne, **votre base de données a été restaureé** et tous vos messages sont bien présents.  

Relance des CRON de sauvgardes.  
```
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":false}}'
```
👉 Nous n'avons pas perdu de données mais Kubernetes ne gère pas la restauration tout seul. Nous avons du protéger nos données via des sauvegardes régulières (du PVC pra-data vers le PVC pra-backup). -> **C'est du PRA**. Il s'agit d'une stratégie de sauvegarde avec une procédure de restauration.  

---------------------------------------------------
Séquence 5 : Exercices  
Difficulté : Moyenne (~45 minutes)
---------------------------------------------------
**Complétez et documentez ce fichier README.md** pour répondre aux questions des exercices.  
Faites preuve de pédagogie et soyez clair dans vos explications et procedures de travail.  

**Exercice 1 :**  
Quels sont les composants dont la perte entraîne une perte de données ?  
  
➡️ Le PVC pra-backup :
Ce volume contient l’ensemble des sauvegardes historiques. S’il est supprimé en même temps que le volume de production (pra-data), aucune copie de sauvegarde ne reste disponible pour restaurer les données.

➡️ Le nœud physique ou virtuel unique :
Dans notre cas, K3d est exécuté dans un Codespace. Les deux volumes (pra-data et pra-backup) sont donc très probablement stockés sur le même disque physique. En cas de panne matérielle du serveur hôte, les données de production ainsi que les sauvegardes seraient perdues simultanément.

➡️ Les données non encore sauvegardées :
Le CronJob effectue une sauvegarde toutes les minutes. Par conséquent, toutes les données ajoutées entre deux exécutions (par exemple un message créé quelques secondes avant un crash) ne seront pas incluses dans la dernière sauvegarde et seront perdues. *

**Exercice 2 :**  
Expliquez nous pourquoi nous n'avons pas perdu les données lors de la supression du PVC pra-data  
  
La récupération a été possible grâce à la redondance asynchrone mise en place.

➡️ Les données sont stockées dans le volume persistant pra-data et non dans le Pod, ce qui permet de les conserver même si le Pod est recréé.
➡️ Un CronJob effectue une sauvegarde du fichier SQLite toutes les minutes en le copiant du volume pra-data vers le volume de sauvegarde pra-backup.
➡️ Lorsque pra-data a été supprimé, le volume pra-backup est resté intact. Le job 50-job-restore.yaml a alors permis de restaurer la dernière sauvegarde vers un nouveau volume pra-data.*

**Exercice 3 :**  
Quels sont les RTO et RPO de cette solution ?  
  
*RPO : 1 minute
Le CronJob effectuant une sauvegarde chaque minute, la perte maximale de données en cas de panne est limitée à 60 secondes.

RTO : 4 à 8 minutes
C’est le temps estimé pour remettre l’application en service : détection de l’incident, suppression des ressources corrompues, redéploiement de l’application et restauration des données. L’application peut ainsi être opérationnelle en moins de 10 minutes.*

**Exercice 4 :**  
Pourquoi cette solution (cet atelier) ne peux pas être utilisé dans un vrai environnement de production ? Que manque-t-il ?   
  
*La solution présente plusieurs limites. Le stockage est local au cluster, ce qui crée un point de défaillance unique : si le cluster K3d ou le serveur tombe, les données de production et les sauvegardes sont perdues, d’autant plus que pra-data et pra-backup sont stockés au même endroit. De plus, les sauvegardes ne sont pas externalisées, alors qu’un vrai PRA exige un stockage hors site (ex. S3 ou autre région).

Par ailleurs, l’utilisation de SQLite ne permet pas d’assurer une haute disponibilité, contrairement à des bases comme PostgreSQL ou MariaDB. Enfin, la restauration est manuelle et aucun système de monitoring n’alerte en cas d’échec du CronJob, ce qui augmente le risque d’erreur et réduit la fiabilité globale du dispositif*
  
**Exercice 5 :**  
Proposez une archtecture plus robuste.   
  
*L’architecture améliore la résilience et la continuité de service (PRA/PCA) en supprimant les points de défaillance uniques et en automatisant les opérations critiques.

Base de données haute disponibilité
SQLite est remplacée par PostgreSQL répliqué (Primary + Replica). Cela permet une réplication en temps réel et une bascule automatique si le serveur principal tombe.

Stockage distribué
Le stockage local est remplacé par un stockage distribué Kubernetes (ex : Longhorn, Ceph, OpenEBS) afin d’assurer la réplication des données entre plusieurs nœuds et éviter le SPOF.

Sauvegardes hors site
Les backups sont envoyés vers un stockage objet externe (S3, Azure Blob, GCS) via des outils comme Velero ou Restic pour garantir la protection en cas de panne totale du cluster.

Restauration automatisée
Les procédures de restauration sont automatisées via scripts, pipelines ou jobs Kubernetes, permettant une reconstruction rapide de l’infrastructure.

Monitoring et alertes
Un système d’observabilité avec Prometheus, Grafana et Alertmanager permet de surveiller les performances, détecter les incidents et envoyer des alertes.

PRA multi-zone
Un cluster secondaire dans une autre région peut être activé en cas de catastrophe pour assurer la continuité de service.

Résultat

RPO : quelques secondes

RTO : environ 2 à 5 minutes

Architecture hautement disponible, supervisée et tolérante aux pannes.*

<img width="1536" height="1024" alt="ChatGPT Image Mar 6, 2026, 01_26_57 PM" src="https://github.com/user-attachments/assets/0b86c8e9-3a20-4c18-9076-9dd2569821cc" />


---------------------------------------------------
Séquence 6 : Ateliers  
Difficulté : Moyenne (~2 heures)
---------------------------------------------------
### **Atelier 1 : Ajoutez une fonctionnalité à votre application**  
**Ajouter une route GET /status** dans votre application qui affiche en JSON :
* count : nombre d’événements en base
* last_backup_file : nom du dernier backup présent dans /backup
* backup_age_seconds : âge du dernier backup

<img width="811" height="141" alt="image" src="https://github.com/user-attachments/assets/147bfbef-8c4d-46bd-98b1-638097edc081" />


---------------------------------------------------
### **Atelier 2 : Choisir notre point de restauration**  
Aujourd’hui nous restaurons “le dernier backup”. Nous souhaitons **ajouter la capacité de choisir un point de restauration**.

*La restauration actuelle est limitée car elle sélectionne automatiquement la sauvegarde la plus récente. Pour permettre un vrai point de restauration, le Job a été modifié afin d’accepter un nom de fichier de backup en paramètre (BACKUP_FILE). Cela permet de restaurer un état précis de la base selon le besoin du PRA.*  

** Etapes de restauration avec point de restauration connue **

1️⃣ Vérifier que les backups existent

Lister les sauvegardes présentes dans le volume /backup.

```
kubectl exec -it deploy/flask -n pra -- ls /backup
```
Exemple de sortie :
```
app-1772801221.db
app-1772801161.db
app-1772801100.db
```
Chaque fichier correspond à un point de restauration.

2️⃣ Choisir le backup à restaurer

Identifier le fichier correspondant au moment voulu.

Exemple :

```
app-1772801161.db
```
3️⃣ Arrêter temporairement l’application (option recommandé)

Pour éviter les écritures pendant la restauration :

```
kubectl scale deployment flask --replicas=0 -n pra
```
4️⃣ Lancer le Job de restauration

Appliquer le Job Kubernetes qui restaure la base.

```
kubectl apply -f 50-job-restore.yaml 
```
5️⃣ Vérifier que la restauration est terminée
Consulter les logs du Job :

```
kubectl logs job/sqlite-restore -n pra
```
6️⃣ Redémarrer l’application

Relancer l’application pour utiliser la base restaurée.
```
kubectl scale deployment flask --replicas=1 -n pra
```
7️⃣ Vérifier l’état de l’application

Contrôler que l’application fonctionne :
```
kubectl get pods -n pra
```

Voici le cron entier avec choix du point de restauration

```
apiVersion: batch/v1
kind: Job
metadata:
  name: sqlite-restore
  namespace: pra
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: restore
          image: alpine
          command: ["/bin/sh","-c"]
          args:
            - |
              if [ -n "$BACKUP_FILE" ]; then
                FILE="/backup/$BACKUP_FILE"
              else
                FILE=$(ls -t /backup/*.db | head -1)
              fi

              if [ ! -f "$FILE" ]; then
                echo "Erreur : backup introuvable"
                ls -1 /backup
                exit 1
              fi

              cp "$FILE" /data/app.db
              echo "Restauration terminée depuis $FILE"
          env:
            - name: BACKUP_FILE
              value: "app-1772801221.db"
          volumeMounts:
            - name: data
              mountPath: /data
            - name: backup
              mountPath: /backup
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: pra-data
        - name: backup
          persistentVolumeClaim:
            claimName: pra-backup
```

---------------------------------------------------
Evaluation
---------------------------------------------------
Cet atelier PRA PCA, **noté sur 20 points**, est évalué sur la base du barème suivant :  
- Série d'exerices (5 points)
- Atelier N°1 - Ajout d'un fonctionnalité (4 points)
- Atelier N°2 - Choisir son point de restauration (4 points)
- Qualité du Readme (lisibilité, erreur, ...) (3 points)
- Processus travail (quantité de commits, cohérence globale, interventions externes, ...) (4 points) 

