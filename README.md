Day 0 Data Protection with Kasten K10, Policy-as-Code guardrails, and GitOps
============================================================================

This is an example project meant to demonstrate the capabilities of [Kasten K10](https://www.kasten.io) to enable automated protection of Kubernetes applications on Day 0. 

As a Kubernetes-native solution, Kasten K10 is built on custom resources that extend the Kubernetes API. It can be easily integrated with other solutions common in a cloud native stack, such as policy-as-code solutions like Kyverno or Open Policy Agent, and continuous delivery solutions, like Argo or Flux.

In this project we will:

  - Use Kasten K10 **Policy Presets** to provide standardized, specialist-validated SLAs for teams to consume
  - Use Kyverno **ClusterPolicies** to enforce all new namespaces having a valid `dataprotection` label to prevent workloads from being deployed inadvertently without backup
  - Use Kyverno **ClusterPolicies** to automatically generate a Kasten K10 **Policy** based on the **Policy Preset** associated with the `dataprotection` label
  - Use Argo to deploy our stateful [Sock Shop](https://microservices-demo.github.io/) application from this Git repository
  - Use Argo **Pre-Sync Waves** to automate running your Kyverno-generated Kasten K10 **Policy** prior to each code update being pushed into production
  - Use Kasten K10 to save the day when someone gets a little sloppy with their database maintenance queries

# Prerequisites

This assumes you have already:

  - [Installed Kasten K10](https://docs.kasten.io/latest/install/index.html) on your cluster in the `kasten-io` namespace
  - [Created three Policy Presets](https://docs.kasten.io/latest/api/policypresets.html#create-a-policypreset) named `gold`, `silver`, and `bronze`.
  
    NOTE: You can configure each preset based on your own needs
  - Forked and then cloned this GitHub repo
  - Install K10 Blueprints to ensure data consistency when performing volume snapshot based backup of your critical data services in the SockShop app:

    ```bash
    kubectl apply -f blueprints -n kasten-io
    ```

# Part 1 - Infrastructure

First we need to install the additional tools we'll be using, Kyverno and Argo.

1. Install Kyverno:

    ```bash
    helm repo add kyverno https://kyverno.github.io/kyverno/
    helm repo update
    helm install kyverno kyverno/kyverno --namespace kyverno --create-namespace
    helm install kyverno-policies kyverno/kyverno-policies --namespace kyverno
    ```

1. Install Argo & start port-forward to access UI:

    ```bash
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    # Return Argo 'admin' password
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
    kubectl port-forward svc/argocd-server -n argocd 9090:443
    ```

1. Log into Argo at https://localhost:9090 using `admin` and the `argocd-initial-admin-secret` password value returned from the previous step.

1. Apply the custom Kyverno ClusterPolicies to enforce namespace `dataprotection` labeling and generate K10 policies based on the associated label:

    ```bash
    kubectl apply -f kyverno/kyvernorbac.yaml
    kubectl apply -f kyverno/enforce-data-protection-by-preset-label.yaml
    kubectl apply -f kyverno/generate-preset-backup-policy-ns.yaml
    ```

# Part 2 - Initial SockShop Deployment

1. In the Argo web UI, add a new application:

1. Perform a manual sync to attempt to deploy the application.

    Note that...

1. Uncomment the `dataprotection: gold` label in `base/create-ns.yaml`

1. Save the file and then commit and push the change to your repo.

1. Perform a manual sync and note the application now deploys successfully and a new Policy was automatically created in K10.

1. Get your SockShop EXTERNAL IP (LoadBalancer):

    ```bash
    kubectl get service front-end -o wide -n sockshop
    ```

1. Open your browser to the SockShop IP and verify you see multiple different socks available for sale.

# Part 3 - Implement Pre-sync Wave Backup

1. Move the following files into the `base/` directory:

    - `argocd-commits/presync-service-account.yaml` # K8s ServiceAccount to be used by the presync Job
    - `argocd-commits/presync-sa-crb.yaml` # ClusterRoleBinding to provide ServiceAccount permissions
    - `argocd-commits/k10-presync-job.yaml` # Job to create K10 RunAction to perform backup

1. Commit and push the changes to your repo.

1. In Argo, perform a manual sync.

# Part 4 - Get Inebriated & Perform DB Maintenance

1. Move the following files into the `base/` directory:

    - `argocd-commits/unwanted-stock-cm.yaml` # ConfigMap defining unwanted sock colors to be removed from the product catalog
    - `argocd-commits/catalogue-db-maint-job.yaml` # Argo Sync Job to update catalogue-db MySQL instance to remove socks of unwanted colors, except OOPS! I forgot the WHERE clause in my query. I wonder what that will do.

2. Commit and push the changes to your repo.

3. In Argo, perform a manual sync.

4. Open the SockShop catalog in your browser and absolutely panic when you realize you've deleted all of them.

5. Open up K10 and see the Argo automagically ran your app's policy before committing any of your changes. 

# Part 5 - Call the Fixer

1. In K10, under Applications > Sockshop, click Restore.
   
2. Select the most recent RestorePoint

3. Perform an in-place restore or granularly restore just the catalog-db PVC.

4. Once the restore completes, return to the SockShop and verify your socks are available once again.

# Part 6 - Second Time is a Charm

1. Edit `base/catalogue-db-maint-job.yaml` to uncomment the WHERE clause as shown below:

    ```sql
    cat <<EOF > catalogue-db-update.sql
    USE socksdb;
    SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0;
    DELETE sock
    FROM sock
    JOIN sock_tag ON sock.sock_id=sock_tag.sock_id
    JOIN tag ON tag.tag_id=sock_tag.tag_id; # <--- Remove ; from end of this line!
    WHERE tag.name in ${COLORS}; # <--- Remove comment # on this line! 
    SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS;
    EOF
    ```

2. Commit and push the changes to your repo.

3. In Argo, perform a manual sync.

4. Once the sync completes, return to SockShop and note the only `green` and `brown` socks have been removed from the catalog.

5. *Promise yourself that you'll stop testing things in production.*

