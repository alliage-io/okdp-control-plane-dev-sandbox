# OKDP Dev Sandbox (Minimalist)

Environnement de développement minimaliste pour OKDP (Open Kubernetes Data Platform).
Ce sandbox permet de lancer un cluster Kubernetes (Kind) local avec les composants essentiels (Flux, Kubocd, Kubauth) pour développer les frontends et backends de la plateforme.

## 1. Prérequis

*   [Docker](https://docs.docker.com/get-docker/) (Runtime)
*   [Kind](https://kind.sigs.k8s.io/) (Kubernetes in Docker)
    *   *Validé avec: `v0.27.0`*
*   [Kubectl](https://kubernetes.io/docs/tasks/tools/)
    *   *Validé avec: `v1.32.2`*
*   [Flux CLI](https://fluxcd.io/flux/installation/)
    *   *Validé avec: `v2.6.1`*
*   `kubocd` (Optionnel, pour le build des packages)

## 2. Démarrage Rapide

### A. Création du Cluster

1.  Créer et lancer le cluster :

    ```bash
    cat <<EOF | kind create cluster --name okdp-dev --config=-
    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    nodes:
    - role: control-plane
      extraPortMappings:
      - containerPort: 30080
        hostPort: 80
      - containerPort: 30443
        hostPort: 443
      - containerPort: 30053
        hostPort: 53
        protocol: UDP
    EOF
    ```

### B. Installation de l'Infrastructure (Flux & Kubocd)

1.  Installer **FluxCD** (Mode standalone) :

    ```bash
    flux install
    ```

2.  Installer **Kubocd** (Controller & CRDs) :

    ```bash
    kubectl apply -f clusters/dev/flux/kubocd.yaml
    ```

3.  Appliquer le **Context** et les **Releases** :

    ```bash
    # Applique le contexte
    kubectl apply -f clusters/dev/default-context.yaml

    # Déploie les dépendances Système (Ingress, Cert-Manager, DNS-Server, Vault, External Secrets)
    kubectl apply -f manifests/infrastructure

    # Attendre que les dépendances Système soient prêtes (inclut coredns-patch pour la résolution DNS interne
    # et metrics-server qui alimente kubectl top + le panel Resource usage du Console)
    kubectl wait --for=jsonpath='{.status.phase}'=READY release/cert-manager release/ingress-nginx release/dns-server release/vault release/external-secrets release/coredns-patch release/metrics-server -n kubocd-system --timeout=240s


    # Créer le namespace système
    kubectl create namespace okdp-system

    # Une fois le système prêt, lance Kubauth
    kubectl apply -f manifests/platform/kubauth.yaml

    # Attendre que Kubauth soit prêt (Release & CRDs)
    kubectl wait --for=jsonpath='{.status.phase}'=READY release/kubauth -n okdp-system --timeout=120s
    kubectl wait --for=condition=established --timeout=60s crd/oidcclients.kubauth.kubotal.io


    # Déploie le Spark Operator (cluster-wide, surveille tous les namespaces)
    kubectl apply -f manifests/platform/spark-operator.yaml
    kubectl wait --for=jsonpath='{.status.phase}'=READY release/spark-operator -n okdp-system --timeout=180s

    # Créer les clients OIDC
    kubectl apply -f manifests/platform/oidc-client.yaml
    kubectl apply -f manifests/platform/oidc-client-spark.yaml

    ```

### C. Initialisation de l'Identité (User Admin)

Une fois Kubauth lancé, vous devez créer l'utilisateur administrateur et son groupe :

1.  Créer l'utilisateur `useradmin` et le groupe `admins` :

    ```bash
    kubectl apply -f manifests/platform/user-admin.yaml
    ```

2.  **Identifiants par défaut :**

    *   **User** : `useradmin`
    *   **Password** : `password`

    > ⚠️ **Note :** Le mot de passe est hashé dans le manifest (`passwordHash`).

## 3. Accès Local

Le cluster expose l'Ingress Controller sur les ports locaux `80` et `443` (mappés sur les NodePorts `30080` et `30443` du container Kind).

### Configuration DNS


Le cluster expose un serveur DNS de développement sur le port NodePort `30053` (UDP).

Pour configurer votre résolveur local (recommandé pour ne pas toucher à `/etc/hosts`), suivez le guide :

👉 **[Voir la documentation DNS](./docs/dns-configuration.md)**

### Certificats SSL

Pour éviter les avertissements de sécurité (et les erreurs de blocage dans l'UI), vous pouvez faire confiance à l'autorité de certification du sandbox.

**Option 1 : Installer le certificat CA (Recommandé)**

Récupérez le certificat CA et ajoutez-le à votre trousseau système ou navigateur :

```bash
kubectl get secret default-issuer -n cert-manager -o jsonpath='{.data.ca\.crt}' | base64 -d > okdp-dev-ca.crt
```

**Option 2 : Ignorer les avertissements**
Accédez manuellement à `https://kubauth.okdp.dev-sandbox` et acceptez le risque (Obligatoire si vous n'installez pas le certificat).

### Vault (Gestion des secrets)

Vault est déployé avec un Ingress et accessible via :

*   **URL** : `https://vault.okdp.dev-sandbox`
*   **Mode** : Dev (token root par défaut)

## 4. Développement Local
 
Ce sandbox fournit l'infrastructure de base (Kubernetes + OIDC) nécessaire pour développer sur les autres composants de la plateforme.
 
*   **Backend (`okdp-server-new`)** :
 
    Si ce n'est pas déjà fait, clonez le dépôt :
    `https://github.com/kubotal/okdp-server-new.git`
 
    ```bash
    cd okdp-server-new
    
    # Configurer l'accès au cluster (Kind)
    kind get kubeconfig --name okdp-dev > ~/.kube/okdp-dev-config
    export KUBECONFIG=~/.kube/okdp-dev-config

    # Lancer le serveur
    go run cmd/server/main.go
    ```
    > Le serveur écoutera sur `http://localhost:8093`.

*   **Frontend (`okdp-ui-new`)** :
 
    Si ce n'est pas déjà fait, clonez le dépôt :
    `https://github.com/kubotal/okdp-ui-new.git`
 
    ```bash
    cd okdp-ui-new
    npm install
    npm start
    ```
    > L'application sera accessible sur `http://localhost:4200`.

    💡 **Connexion :** Utilisez l'utilisateur `useradmin` / `password` (créé à l'étape 2.C).
