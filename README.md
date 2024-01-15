# Create SSH secret for CasC SCM retriever in Cjoc 

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

