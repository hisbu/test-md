# Install Hashicorp Vault on Kubernetes

## Prerequisites

- Helm v3
- Kubernetes Cluster
- Kubectl

### Install Vault

- add the Hashicorp helm repository
    
    ```bash
    helm repo add hashicorp https://helm.releases.hashicorp.com
    ```
    
- check that you have access to the chart
    
    ```bash
    helm search repo hashicorp/vault
    ```
    
- Use `helm install` to install the latest release of the Vault Helm chart
    
    ```bash
    helm install vault hashicorp/vault -n vault   
    ```
    
- Or install a specific version of the chart.
    
    ![Untitled](Install%20Hashicorp%20Vault%20on%20Kubernetes%206fddaec7be87437fa4d0064c5125dbf5/Untitled.png)
    
- Override default setting
    
    ```bash
    $ helm install vault hashicorp/vault \    
    		--namespace vault \    
    		--set "server.ha.enabled=true" \    
    		--set "server.ha.replicas=3" \
    ```
    
    > *ha replias depends on available node worker*
    > 
- Alternatively, specify the desired configuration in a file, `override-ha-values.yml`.
    
    ```yaml
    # Vault Helm Chart Value Overrides
    global:
      enabled: true
      tlsDisable: true
    
    injector:
      enabled: true
      image:
        repository: "hashicorp/vault-k8s"
        tag: "latest"
      affinity: ""
    server:
      auditStorage:
        enabled: false
      standalone:
        enabled: false
      image:
        repository: "hashicorp/vault"
        tag: "latest"
      affinity: ""
      readinessProbe:
        enabled: true
        path: "/v1/sys/health?standbyok=true&sealedcode=204&uninitcode=204"
      ha:
        enabled: true
        replicas: 3
        raft:
          enabled: true
          setNodeId: true
          config: |
            ui = true
    
            listener "tcp" {
              tls_disable = true
              address = "[::]:8200"
              cluster_address = "[::]:8201"
            }
    
            storage "raft" {
              path = "/vault/data"
            }
    
            service_registration "kubernetes" {}
        config: |
          ui = true
    
          listener "tcp" {
            tls_disable = true
            address = "[::]:8200"
            cluster_address = "[::]:8201"
          }
    
          service_registration "kubernetes" {}
    
    ui:
      enabled: true
      externalPort: 8200
    ```
    
- Override the default configuration with the values read from the `override-values.yml` file.
    
    ```bash
    $ helm install vault hashicorp/vault \    
    		--namespace vault \    
    		-f override-ha-values.yml
    ```
    

### Initialize and unseal Vault

After the Vault Helm chart is installed in `standalone` or `ha` mode one of the Vault servers need to be [initialized](https://www.vaultproject.io/docs/commands/operator/init). The initialization generates the credentials necessary to [unseal](https://www.vaultproject.io/docs/concepts/seal#why) all the Vault servers.

- CLI initialize and unseal
    
    View all the Vault pods in the current namespace:
    
    ```bash
    $ kubectl get pods -l app.kubernetes.io/name=vault
    NAME                                    READY   STATUS    RESTARTS   AGE
    vault-0                                 0/1     Running   0          1m49s
    vault-1                                 0/1     Running   0          1m49s
    vault-2                                 0/1     Running   0          1m49s
    ```
    
- Initialize one Vault server with the default number of key shares and default key threshold:
    
    ```bash
    $ kubectl exec -ti vault-0 -- vault operator init
    Unseal Key 1: MBFSDepD9E6whREc6Dj+k3pMaKJ6cCnCUWcySJQymObb
    Unseal Key 2: zQj4v22k9ixegS+94HJwmIaWLBL3nZHe1i+b/wHz25fr
    Unseal Key 3: 7dbPPeeGGW3SmeBFFo04peCKkXFuuyKc8b2DuntA4VU5
    
    Initial Root Token: s.zJNwZlRrqISjyBHFMiEca6GF
    ##..
    ```
    
    > *The output displays the key shares and initial root key generated.*
    > 
- Unseal the Vault server with the key shares until the key threshold is met:
    
    ```bash
    ## Unseal the first vault server until it reaches the key threshold
    $ kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 1
    $ kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 2
    $ kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 3
    
    ##example
    $ kubectl exec -ti vault-0 -- vault operator unseal MBFSDepD9E6whREc6Dj+k3pMaKJ6cCnCUWcySJQymObb
    ```
    
    > Finally, join the remaining pods to the Raft cluster and unseal them. The pods will need to communicate directly so we'll configure the pods to use the internal service provided by the Helm chart:
    > 
    
    ```bash
    $ kubectl exec -ti vault-1 -- vault operator raft join http://vault-0.vault-internal:8200
    $ kubectl exec -ti vault-1 -- vault operator unseal # .... unseal key vault-0-1
    $ kubectl exec -ti vault-1 -- vault operator unseal # .... unseal key vault-0-1
    $ kubectl exec -ti vault-1 -- vault operator unseal # .... unseal key vault-0-1
    
    $ kubectl exec -ti vault-1 -- vault operator raft join http://vault-0.vault-internal:8200
    $ kubectl exec -ti vault-1 -- vault operator unseal # .... unseal key vault-0-1
    $ kubectl exec -ti vault-1 -- vault operator unseal # .... unseal key vault-0-1
    $ kubectl exec -ti vault-1 -- vault operator unseal # .... unseal key vault-0-1
    ```
    

source: [https://www.vaultproject.io/docs/platform/k8s/helm/run](https://www.vaultproject.io/docs/platform/k8s/helm/run)