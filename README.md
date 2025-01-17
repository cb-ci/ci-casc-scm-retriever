This repo is about how to enable CasC SCM Retriever on k8s
See
* https://docs.cloudbees.com/docs/cloudbees-ci/latest/casc-oc/bundle-retrieval-scm
* https://docs.cloudbees.com/docs/cloudbees-ci/latest/casc-oc/bundle-retrieval-scm#_scm_configuration
* https://docs.cloudbees.com/docs/cloudbees-ci/latest/casc-oc/bundle-retrieval-scm#_optional_configure_github_webhook_support

# GitHub: SSH Secret and config

## Create SSH secret for CasC SCM retriever in CloudBees Operations Center


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

## Create Webhook (Optional, when webhooks should be enabled)

See 
* https://docs.cloudbees.com/docs/cloudbees-ci/latest/casc-oc/bundle-retrieval-scm#_optional_configure_github_webhook_support


* Create Webhook Secret 

> export WEBHOOK_SECRET=$(echo -n any string | shasum -a 256 | awk '{ print $1 }') 


* In GitHub -> CJoc casc repo -> Setting -> Webhooks
* Create Webhook
 * https://docs.github.com/en/webhooks/using-webhooks/creating-webhooks
 * https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries


> PayLoad URL: https:/${DOMAIN}/cjoc/casc-retriever/retriever-github-webhook/
> Secret: $WEBHOOK_SECRET

* Add k8s secret with Webhook secret

```
cat <<EOF | kubectl  apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: casc-webhook-secret
type: Opaque
stringData:
    casc-webhook-secret: $WEBHOOK_SECRET
EOF
```



## Enable CasC retriever in Helm values
  * Enable  CasC retriever: 
  * `Casc.Retriever.Enabled=true`
  * Set sshConfig: The k8s ssh secret name is required here(That we have created in the steps above)
  * `Casc.Retriever.secrets.sshConfig=casc-ssh-secret`

## SSH AUTH: Helm values section for SCM retriever looks like this:

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
      scmRepo: "git@github.com:cb-ci/casc.git" <- Adjust to your CJOC Casc repo
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
        githubWebhookSecret: casc-cjoc-retriever-webhook-secret

```


# BitBucket: HTTP Basic Auth with App Password token

## Generate an App Password in Bitbucket
see https://support.atlassian.com/bitbucket-cloud/docs/app-passwords/

* Go to your Bitbucket account settings:
  https://bitbucket.org/account/settings/
* Under the Access management section, click App passwords.
* Click the Create app password button.
* Give your app password a meaningful name (e.g., "Git CLI Auth").
* Select the permissions required for your operations. For basic Git usage with a private repository, ensure the following are checked:

 * Repositories:
  * > Read
  * > Write

* (Optional) Account → Read, if you want to access user information.
* Click Create and note down the app password displayed (you won’t see it again).

# Test local first(Optional, not required for CasC SCM retriever)

* Stop and clean the local git cache daemon, it might contain an old token
> git credential-cache exit

> git clone https://<your_bitbucket_username>@bitbucket.org/<workspace>/<repository>.git

* git will ask you now for the app password token
* make a change and try to push to git 
* git push should be successfully 



## create K8s secret with BB username and app pw token

```
kubectl create secret generic casc-retriever-secrets \
    --from-literal=scmUsername=xxxx \
    --from-literal=scmPassword=xxxx 

kubectl get secret casc-retriever-secrets -o yaml 
```

Note: i added the `--from-literal=scmPassword=xxxx without ''` , have not tested if quotes matters or not. it worked for me without the surrounding single quotes
--from-literal=scmPassword=xxxx  vs --from-literal=scmPassword='xxxx'
## Adjust helm values

```
  CasC:
    Enabled: true
    Retriever:
      Enabled: true
      # OperationsCenter.CasC.Retriever.scmRepo -- The url of the repo containing the casc bundle
      scmRepo: <REPO_URL>
      # OperationsCenter.CasC.Retriever.scmBranch -- The branch of the repo containing the casc bundle
      scmBranch: <BRANCH_NAME>
      # OperationsCenter.CasC.Retriever.scmPollingInterval -- How frequently to poll SCM for changes
      # Interval is specified using standard java Durataion format
      # (see https://docs.oracle.com/javase/8/docs/api/java/time/Duration.html#parse-java.lang.CharSequence-)
      scmPollingInterval: "PT1M"
      # OperationsCenter.CasC.Retriever.secrets -- Allows you to customize the key used for each secret value
      secrets:
        # OperationsCenter.CasC.Retriever.secrets.secretName -- Define the name of the object that holds the secrets, defaults to casc-retriever-secrets if not specified
        secretName: casc-retriever-secrets
        # OperationsCenter.CasC.Retriever.secrets.scmUsername -- SCM username to authenticate against the repo
        scmUsername: scmUsername
        # OperationsCenter.CasC.Retriever.secrets.scmPassword -- SCM password / token used in user authentication in the repo
        scmPassword: scmPassword
```

# Git credentials cache (Optional, not required for CasC SCM retriever)


Cache Your Credentials
To avoid entering the app password every time:

Configure Git to cache your credentials:

```
git config --global credential.helper store
```

The next time you push, your credentials will be stored in plain text in ~/.git-credentials.

Alternatively, use the cache helper to store credentials temporarily:
```
git config --global credential.helper cache
```

By default, credentials will be cached for 15 minutes. You can adjust the timeout:

```
git config --global credential.helper 'cache --timeout=3600'

```

