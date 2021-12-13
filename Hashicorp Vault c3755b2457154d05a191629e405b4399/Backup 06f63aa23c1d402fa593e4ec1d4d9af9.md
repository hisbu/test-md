# Backup

### Setup Policy and Authentication

> This is mostly stolen from adfinis-sygroup/vault-raft-backup-agent#approle-authentication
> 

Create a minimal policy for our snapshot agent to perform the backup job.

```bash
echo 'path "sys/storage/raft/snapshot" {
   capabilities = ["read"]
}' | vault policy write snapshot -
```

> The approle auth method allows machines or apps to authenticate with Vault-defined roles.
> 

[AppRole](https://www.vaultproject.io/docs/auth/approle) auth method is perfectly suited for the snapshot agent to authenticate with our vault cluster. Notes `role-id` and `secret-id`, you will need them later.

```bash
vault auth enable approle
vault write auth/approle/role/snapshot-agent token_ttl=2h token_policies=snapshot
vault read auth/approle/role/snapshot-agent/role-id -format=json | jq -r .data.role_id
vault write -f auth/approle/role/snapshot-agent/secret-id -format=json | jq -r .data.secret_id
```

### 

### Prepare Secrets

Let’s save all our sensitive information as [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/). We will use them later.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-snapshot-agent-token
type: Opaque
data:
  # we use gotmpl here
  # you can replace them with base64-encoded value
  VAULT_APPROLE_ROLE_ID: {{ .Values.approle.secretId | b64enc | quote }}
  VAULT_APPROLE_SECRET_ID: {{ .Values.approle.secretId | b64enc | quote }}
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-snapshot-s3
type: Opaque
data:
  # we use gotmpl here
  # you can replace them with base64-encoded value
  AWS_ACCESS_KEY_ID: {{ .Values.backup.accessKeyId | b64enc | quote }}
  AWS_SECRET_ACCESS_KEY: {{ .Values.backup.secretAccesskey | b64enc | quote }}
  AWS_DEFAULT_REGION: {{ .Values.backup.region | b64enc | quote }}
```

### 

### The CronJob

Let’s create the [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) that actually does the work.

We configure `VAULT_ADDR` environment variable to `http://vault-active.vault.svc.cluster.local:8200`. Using `vault-active` [Service](https://kubernetes.io/docs/concepts/services-networking/service/) can make sure the snapshot request is made against the `leader` node, assuming you have enabled [Service Registration](https://www.vaultproject.io/docs/configuration/service-registration/kubernetes#kubernetes-service-registration), which is the [default](https://github.com/hashicorp/vault-helm/blob/f67b844d3027b981d12a56957f5fbcbf85ec5adc/values.yaml#L601). The exact url may vary depending on your vault helm chart deployment release name and targer namespace, [learn more](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/).

> I may have over-engineered the cronjob by using multiple containers 
to perform a simple backup and upload task. The intention is to avoid 
building custom images and I don’t want to maintain yet another image.
> 

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: vault-snapshot-cronjob
spec:
  # schedule: "@every 12h"
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          volumes:
          - name: share
            emptyDir: {}
          containers:
          - name: snapshot
            image: vault:1.7.2
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            args:
            - -ec
            - |
              export VAULT_TOKEN=$(vault write auth/approle/login role_id=$VAULT_APPROLE_ROLE_ID secret_id=$VAULT_APPROLE_SECRET_ID -format=json | grep -Eo '"client_token"[^,]*' | grep -Eo '[^:]*$' | sed 's/"//g');
              vault operator raft snapshot save /share/vault-raft.snap; 
              ls /share
            envFrom:
            - secretRef:
                name: vault-snapshot-agent-token
            env:
            - name: VAULT_ADDR
              value: "http://vault-ha-active.vault.svc.cluster.local:8200"
            volumeMounts:
            - mountPath: /share
              name: share

          - name: upload
            image: amazon/aws-cli:2.2.14
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            args:
            - -ec
            - |
              until [ -f /share/vault-raft.snap ]; do sleep 5; done;
              aws s3 cp /share/vault-raft.snap s3://xti-datavault/vault_xti_raft_$(date +"%Y%m%d_%H%M%S").snap;
              aws s3 ls s3://xti-datavault          
            env:
            - name: AWS_ACCESS_KEY_ID
              value: "****"
            - name: AWS_SECRET_ACCESS_KEY
              value: "****"
            volumeMounts:
            - mountPath: /share
              name: share
          restartPolicy: Never
```

*source:* 

- [*https://michaellin.me/backup-vault-with-raft-storage-on-kubernetes/*](https://michaellin.me/backup-vault-with-raft-storage-on-kubernetes/)
- [https://stackoverflow.com/questions/57971193/how-to-read-and-parse-json-in-shell-scripting-without-using-json-tool-and-jq-too](https://stackoverflow.com/questions/57971193/how-to-read-and-parse-json-in-shell-scripting-without-using-json-tool-and-jq-too)