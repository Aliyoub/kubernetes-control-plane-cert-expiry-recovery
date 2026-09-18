# Expiration des certificats du control plane — Reprise après incident sur un cluster Kubernetes

**Auteur :** Aliyou BINATE

Démonstration pratique d'un incident redouté par toute équipe qui opère
un cluster `kubeadm` sur la durée : les certificats internes du control
plane expirent après un an et ne se renouvellent jamais tout seuls. Le
jour où personne n'y a pensé, le cluster entier cesse de répondre — sans
avertissement préalable. Cette étude de cas provoque volontairement cette
panne sur un cluster réel, la diagnostique à partir des seuls symptômes
observables, puis la corrige.

---

## En bref

`kubeadm` émet par défaut des certificats valables **un an**, sans
renouvellement automatique. À l'expiration, toute communication
authentifiée par certificat client au sein du control plane est
refusée — `kubectl`, kubelet, les webhooks d'authentification, tout.
La réparation passe par `kubeadm certs renew all` puis un redémarrage
des pods statiques du control plane pour qu'ils chargent les nouveaux
certificats — **à condition que l'horloge système soit correcte au
moment du renouvellement**, sous peine de recréer le problème dans
l'autre sens (voir "Décisions techniques clés").

---

## Environnement

- Kubernetes v1.32, cluster à 3 nœuds provisionné avec `kubeadm`
  (1 control-plane + 2 workers), CNI Calico, CRI containerd
- Toutes les opérations s'exécutent en SSH, directement sur chaque nœud
  concerné
- **Horloge des 3 nœuds avancée ensemble** (pas seulement celle du
  control-plane) pour simuler "un an s'est écoulé" — une première
  tentative n'avançant que l'horloge du control-plane a produit une
  panne différente et instructive, détaillée plus bas

---

## Le scénario

1. Relever l'état de référence : cluster sain, certificats valides.
2. Avancer l'horloge système des **3 nœuds** au-delà de la date
   d'expiration des certificats.
3. Constater la panne et la diagnostiquer.
4. Corriger via `kubeadm certs renew all` et redémarrer le control
   plane.
5. Vérifier que le cluster est de nouveau pleinement fonctionnel.

---

## Étape 1 — État de référence

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
sudo kubeadm certs check-expiration
```

**Capture 1**

![État de référence — cluster sain, certificats à 354 jours de leur expiration](screenshots/01-baseline-healthy.png)

Cluster pleinement opérationnel (3 nœuds `Ready`, tous les pods
`kube-system` `Running`), certificats à 354 jours de leur expiration.

---

## Étape 2 — Provoquer l'expiration

Sur **les 3 nœuds** (pas seulement le control-plane — voir "Décisions
techniques clés" pour pourquoi c'est important) :

```bash
sudo systemctl stop vboxadd-service   # empêche la resynchro auto de l'horloge (spécifique à ce lab VirtualBox)
sudo date -s '2027-09-10 12:00:00 UTC'
```

Aucun redémarrage n'a même été nécessaire pour déclencher la panne :
la toute première commande utilisant une **nouvelle** connexion TLS
échoue immédiatement.

```bash
kubectl get nodes
```

**Capture 2**

![Panne déclenchée — rejet x509 immédiat](screenshots/02-failure-onset.png)

Le message est sans ambiguïté : ce n'est pas une panne réseau, c'est un
rejet de certificat, daté et explicite.

---

## Étape 3 — Diagnostic

```bash
sudo kubeadm certs check-expiration
```

**Capture 3**

![Diagnostic — résiduel <invalid>, repli sur la lecture locale des fichiers PKI](screenshots/03-diagnosis.png)

Détail révélateur : `kubeadm` lui-même essaie normalement de lire sa
configuration depuis un `ConfigMap` en cluster — et n'y arrive plus,
pour la même raison que `kubectl`. Il se **rabat silencieusement** sur
la lecture directe des fichiers PKI locaux, ce qui permet quand même
d'obtenir le diagnostic (`<invalid>` au lieu d'un compte à rebours) sans
avoir besoin d'un control plane fonctionnel. Comprendre ce mécanisme de
repli évite de croire, à tort, que `kubeadm certs check-expiration` est
lui aussi hors service.

---

## Étape 4 — Réparation : renouvellement des certificats

```bash
sudo kubeadm certs renew all
```

**Capture 4**

![Certificats renouvelés via kubeadm certs renew all](screenshots/04-renew-certs.png)

Renouveler les fichiers ne suffit pas : les processus du control plane
ont déjà les anciens certificats chargés en mémoire. kubelet ne relit
un pod statique que si son fichier manifeste change (ou disparaît) —
d'où un aller-retour explicite plutôt qu'un simple redémarrage de
service :

```bash
sudo mkdir -p /tmp/manifests-backup
sudo mv /etc/kubernetes/manifests/*.yaml /tmp/manifests-backup/
```

**Capture 4b**

![Manifestes déplacés — /etc/kubernetes/manifests/ vide, les 4 fichiers dans /tmp/manifests-backup/](screenshots/04b-manifests-moved.png)

kubelet surveille en permanence le contenu de
`/etc/kubernetes/manifests/` (un simple dossier, pas une API) : dès
qu'un fichier y disparaît, il en déduit que le pod statique
correspondant doit être arrêté. Le dossier vide ici ne signifie pas
"le control plane a planté" — c'est une action volontaire et
réversible, le fichier existe toujours juste à côté, dans
`/tmp/manifests-backup/`.

Vérifier explicitement que les 4 conteneurs ont disparu avant de les
remettre — un premier essai avec un simple délai fixe s'était avéré
insuffisant pour `etcd` (plus lent à s'arrêter proprement que les
autres) :

```bash
sudo crictl ps | grep -E "etcd|kube-apiserver|kube-controller|kube-scheduler"
# répéter jusqu'à ce que la liste soit vide
```

**Capture 4c**

![Les 4 conteneurs ont disparu — seul calico-kube-controllers (sans rapport) reste](screenshots/04c-containers-gone.png)

Confirmation avant de remettre les manifestes : les 4 anciens
conteneurs ont bien été tués, pas seulement "en cours d'arrêt". Les
remettre trop tôt (avant cette confirmation) est précisément l'erreur
qui, lors d'un premier essai, avait laissé `etcd` tourner encore avec
son ancien certificat.

```bash
sudo mv /tmp/manifests-backup/*.yaml /etc/kubernetes/manifests/
sleep 20
sudo crictl ps | grep -E "etcd|kube-apiserver|kube-controller|kube-scheduler"
```

**Capture 5**

![Les 4 composants du control plane repartis avec de nouveaux conteneurs (ATTEMPT 0)](screenshots/05-restart-control-plane.png)

Quatre nouveaux ID de conteneur, démarrés à l'instant — la preuve que
les certificats renouvelés sont bien ceux désormais chargés.

---

## Décisions techniques clés

<!-- TODO -->

---

## Compétences démontrées

<!-- TODO -->

---

## Reproduire cette démonstration

<!-- TODO -->

---

## Documentation

<!-- TODO -->
