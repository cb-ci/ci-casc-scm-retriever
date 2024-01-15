# Create SSH secret for CasC SCM retriever in CloudBees Operations Center

This repo is about how to enable SSH for Git in the CasC retriever on K8s

See
* https://docs.cloudbees.com/docs/cloudbees-ci/latest/casc-oc/bundle-retrieval-scm
* https://docs.cloudbees.com/docs/cloudbees-ci/latest/casc-oc/bundle-retrieval-scm#_scm_configuration 

## Required files for SSH auth:

*  config 
```
Host github.com
User <USERNAME>
Hostname ssh.github.com
AddKeysToAgent yes
PreferredAuthentications publickey
IdentitiesOnly yes
IdentityFile ~/.ssh/id_rsa
Port 443
```

* id_rsa 

```
-----BEGIN RSA PRIVATE KEY-----
     ####YOUR_PRIVATE KEY HERE####
-----END RSA PRIVATE KEY-----
```

* known_hosts

`ssh-keyscan` was not working for me so I used the known host file that have been created by a real `git clone git@.....`operation

```
[ssh.github.com]:443 ecdsa-sha2-nistp256 ..........
[ssh.github.com]:443 ssh-ed25519 ............
[ssh.github.com]:443 ssh-rsa ..................
ssh.github.com ssh-ed25519 ............
```

* Create all the three files in a directory called `k8s-casc-secret` 
* Then run the following command to create the secret in k8s

```
cd k8s-casc-secret
kubectl delete secret casc-ssh-secret
kubectl create secret generic casc-ssh-secret  --from-file=./
```

* Enable CasC retriever in Helm values
  * Enable  CasC retriever: 
  * `Casc.Retriever.Enabled=true`
  * Set sshConfig: The k8s ssh secret name is required here(That we have created in the steps above)
  * `Casc.Retriever.secrets.sshConfig=casc-ssh-secret`

The overall Helm values ection for SCM retriever looks like this:

```
# Operations Center options
OperationsCenter:
  Enabled: true
  ....
  # Configuration as Code (CasC) for Operations Center.
  CasC:
    # OperationsCenter.CasC.Enabled -- enable or disable CasC for Operations Center.
    Enabled: true
    # OperationsCenter.CasC.ConfigMapName -- the name of the ConfigMap used to configure Operations Center.
    # Note: this property can point to a ConfigMap defined in OperationsCenter.ExtraConfigMaps,
    # or any ConfigMap that exists in the cluster.
    ConfigMapName: oc-casc-bundle
    Retriever:
      Enabled: true
      #scmRepo: "https://github.com/cb-ci/casc.git"
      scmRepo: "git@github.com:cb-ci/casc.git" <- Adjust to youfr CJOC Casc repo
      # OperationsCenter.CasC.Retriever..scmBranch -- The branch of the repo containing the casc bundle
      scmBranch: "master"
      # OperationsCenter.CasC.Retriever.scmBundlePath -- path to a folder within the repo where the bundle.yaml is located
      scmBundlePath: "/casc-operationscenter/bundle" # <- Adjust the Sub path in the CasC Repo where to expect the CjoC bundle 
      # OperationsCenter.CasC.Retriever.ocBundleAutomaticVersion -- if true, the commit hash will replace the version of the bundle from the SCM
      ocBundleAutomaticVersion: "true"
      # OperationsCenter.CasC.Retriever.scmPollingInterval -- How frequently to poll SCM for changes
      # Interval is specified using standard java Durataion format
      # (see https://docs.oracle.com/javase/8/docs/api/java/time/Duration.html#parse-java.lang.CharSequence-)
      scmPollingInterval: "PT1M"
      # OperationsCenter.CasC.Retriever.githubWebhooksEnabled -- Indicates if Github webhook support is activated
      githubWebhooksEnabled: "true"
      ...
      # OperationsCenter.CasC.Retriever.secrets -- Allows you to customize the key used for each secret value
      secrets:
        ...
        sshConfig: casc-ssh-secret # <- For SSH we need to set the k8s secret containing the ssh secret 
        # OperationsCenter.CasC.Retriever.secrets.githubWebhookSecret -- If Github webhooks will be using a secret, set it up here. If not indicated all webhooks from configured repository + branch will be accepted
        githubWebhookSecret: null

```

