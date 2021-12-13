# external vault

1. **Enable Kubernetes Vault Authentication**
    
    ```bash
    $ vault auth enable kubernetes
    ```
    
    if you want enable auth mothod kubernets with spesific path  for another k8s cluster 
    
    ```bash
    $ vault auth enable -path=cluster2/ kubernetes
    ```
    
2.  **Install Vault agent integration with external vault server**
    
    ```bash
    helm install vault hashicorp/vault --set injector.externalVaultAddr="http://167.172.7.218:8200"
    ```
    
    install vault agent integration with spesific auth path 
    
    ```bash
    helm install vault hashicorp/vault --set injector.externalVaultAddr="http://167.172.7.218:8200" --set injector.authPath="auth/cluster2"
    ```
    
    *injector.externalVaultAddr = vault server address
    injector.authPath = custom path for auth method kubernets you have*
    
    You will able to see vault-agent-injector pod running in your default namespace:
    
    ```bash
    ➜  kubectl get pod                        
    NAME                                   READY   STATUS    RESTARTS   AGE
    vault-agent-injector-98d87bf9c-md455   1/1     Running   0          75m
    ```
    
3. **Create a service account vault_auth and enable RBAC using clusterRoleBinding to grant access cluster-wide**
    
    create service account
    
    ```bash
    kubectl create serviceaccount vault-auth
    ```
    
    create ClusterRoleBinding
    
    ```yaml
    cat vault-auth-service-account.yam
    ---
    apiVersion: [rbac.authorization.k8s.io/v1](http://rbac.authorization.k8s.io/v1)
    kind: ClusterRoleBinding
    metadata:
    	name: role-tokenreview-binding
    	namespace: default
    roleRef:
    	apiGroup: [rbac.authorization.k8s.io](http://rbac.authorization.k8s.io/)
    	kind: ClusterRole
    	name: system:auth-delegator
    subjects:
    	- kind: ServiceAccount
    		name: vault-auth
    		namespace: default
    ```
    
    ```bash
    kubectl apply -f vault-auth-service-account.yaml
    ```
    
    Get service account value created earlier
    
    ```bash
    export VAULT_SA_NAME=$(kubectl get sa vault-auth -o jsonpath="{.secrets[*]['name']}")
    ```
    
    Get the JWT_TOKEN value to access TokenReview API
    
    ```bash
    export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data.token}" | base64 --decode; echo)
    ```
    
    Get the Kubernetes certificate value
    
    ```bash
    export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)
    ```
    
    Get the Kubernetes server address
    
    ```bash
    export KUBE_HOST=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.server}')
    ```
    
    Get the issuer value
    
    ```bash
    kubectl proxy &curl --silent http://127.0.0.1:8001/.well-known/openid-configuration | jq -r .issuer
    # Kill the background proxy process when you're done
    kill %%
    ```
    
    ```bash
    export ISSUER="https://kubernetes.default.svc.cluster.local"
    ```
    
    Configure vault auth on /config endpoint using the above values.
    
    ```bash
    kubectl --context do-sgp1-my-lab -n coba exec -it vault-has-0 -- vault write auth/kubernetes/config \
    token_reviewer_jwt=$SA_JWT_TOKEN \
    kubernetes_host=$KUBE_HOST \                                                                      
    kubernetes_ca_cert=$SA_CA_CRT \
    issuer=$ISSUER
    ```
    
4. **Injecting secrets into application**
    
    Now we need that application should be able to read the vault secrets using a vault-agent as a sidecar container. Hashicorp vault provides 
    two methods to use vault-agent integration as sidecar to read secrets:
    
    - the vault.hashicorp.com/agent-inject-secret annotation, or
    - a configuration map containing Vault Agent configuration files.(Note: Only one of these methods can be used at a time).
    
    We have configured our application to access secrets using vault.hashicorp.com/agent-inject-secret annotation.
    
    First create a vault secret path to store all credentials.
    
    ```bash
    vault secrets enable -path=secrets kv
    ```
    
    Put secrets into secrets/ pat
    
    ```bash
    $ vault kv put /secrets/myapp username="appuser" password="password"
    Key                Value
    ---                -----
    created_time       2021-12-10T03:01:51.500762307Z
    custom_metadata    <nil>
    deletion_time      n/a
    destroyed          false
    version            1
    
    $ vault kv get /secrets/myapp
    ======= Metadata =======
    Key                Value
    ---                -----
    created_time       2021-12-09T07:23:16.728980378Z
    custom_metadata    <nil>
    deletion_time      n/a
    destroyed          false
    version            1
    
    ====== Data ======
    Key         Value
    ---         -----
    password    password
    username    appuser
    ```
    
    Create vault policy to read secrets from defined path
    
    ```bash
    $ cat > app-read-policy.hcl <<EOF
    path "/secrets/data/myapp" {
      capabilities = ["read"]
    }
    EOF
    ```
    
    ```bash
    vault policy write app-read-policy app-read-policy.hcl
    ```
    
    And then create service account in desired namespace
    
    ```bash
    $ kubectl create ns myapp-ns                                                                            
    namespace/myapp-ns created
    $ kubectl create serviceaccount myapp-sa -n myapp-ns
    serviceaccount/myapp-sa created
    ```
    
    Last vault role to bound service account and policy for authentication.
    
    ```bash
    vault write auth/cluster2/role/myapp-role \                         
    	bound_service_account_names=myapp-sa \
    	bound_service_account_namespaces=myapp-ns \
    	policies=app-read-policy \
    	ttl=24h
    ```
    
    ***Note**: if the application is running in a default/different namespace, create your service account in respective namespace and change namespace value in above vault role definition.*
    
    Create deployment manifest
    
    ```yaml
    cat deployment-myapp.yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: myapp
      labels:
        app: vault-agent-injector-demo
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: vault-agent-injector-demo
      template:
        metadata:
          labels:
            app: vault-agent-injector-demo
        spec:
          serviceAccountName: myapp-sa
          containers:
            - name: myapp
              image: nginx
    ```
    
    ```yaml
    kubectl apply -f deployment-myapp.yaml -n myapp-ns
    ```
    
    Get list of pods running n myapp-ns namespace
    
    ```bash
    ➜  kubectl get pod -n myapp-ns                          
    NAME                     READY   STATUS    RESTARTS   AGE
    myapp-7b9c987fc8-79v6n   1/1     Running   0          9s
    ```
    
    Currently there is no annotation applied to pod for vault injector and if we look for /vault/secrets volume in pod we won’t get anything
    
    ```bash
    ➜  kubectl exec myapp-7b9c987fc8-79v6n -n myapp-ns -- ls /vault/secrets
    ls: cannot access '/vault/secrets': No such file or directory
    command terminated with exit code 2
    ```
    
    Let’s apply the vault annotation and apply the same to our running application myapp
    
    ```yaml
    $ cat myapp-vault-agent-patch.yaml
    ---
    spec:
      template:
        metadata:
          annotations:
            vault.hashicorp.com/agent-inject: 'true'    
            vault.hashicorp.com/role: [vault auth kubernetes role you have]
            vault.hashicorp.com/agent-inject-secret-[file name for secret you will save]: [secret path you want to get]
            vault.hashicorp.com/agent-inject-template-myapp.json: |
              {{ with secret "secrets/myapp" -}}
              {
                "username": "{{ .Data.data.username }}",
                "password": "{{ .Data.data.password }}"
              }
              {{ end -}}
    ```
    
    ```yaml
    $ cat myapp-vault-agent-patch.yaml
    ---
    spec:
      template:
        metadata:
          annotations:
            vault.hashicorp.com/agent-inject: 'true'    
            vault.hashicorp.com/role: 'myapp-role'    
            vault.hashicorp.com/agent-inject-secret-myapp.json: 'secrets/myapp'
            vault.hashicorp.com/agent-inject-template-myapp.json: |
              {{ with secret "secrets/myapp" -}}
              {
                "username": "{{ .Data.data.username }}",
                "password": "{{ .Data.data.password }}"
              }
              {{ end -}}
    ```
    
    ```bash
    $ kubectl patch deployment myapp --patch "$(cat myapp-vault-agent-patch.yaml)" -n myapp-ns
    ```
    
    ```bash
    $ kubectl get pod -n myapp-ns
    NAME                     READY   STATUS    RESTARTS   AGE
    myapp-7d554b975f-pz9kp   2/2     Running   0          8s
    ```
    
    Let’s run the command again to get the content from /vault/secrets path:
    
    ```bash
    ➜  external-vault kubectl exec myapp-7d554b975f-pz9kp -n myapp-ns --container myapp -- cat /vault/secrets/myapp.json
    {
      "username": "appuser",
      "password": "password"
    }
    ```
    
    source:
    
    [https://blogs.halodoc.io/kubernetes-and-vault-integration/](https://blogs.halodoc.io/kubernetes-and-vault-integration/)
    
    [https://learn.hashicorp.com/tutorials/vault/kubernetes-external-vault](https://learn.hashicorp.com/tutorials/vault/kubernetes-external-vault)
    
    [https://www.vaultproject.io/docs/auth/kubernetes#discovering-the-service-account-issuer](https://www.vaultproject.io/docs/auth/kubernetes#discovering-the-service-account-issuer)