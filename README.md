# Using Openshift Gitops to manage Secrets with SOPS

## Prerequisites
+ Openshift 4 Cluster with Gitops operator installed
+ oc tool locally installed
+ Kustomize CLI locally installed
+ SOPS CLI locally installed
+ Age file encryption tool installed

## How to use
+ Generate a new key with the age tool: \
`$ age-keygen -o age.agekey`

+ Create a secret in the Gitops operator project: \
`$ cat age.agekey | oc create secret generic sops-age --namespace=openshift-gitops --from-file=key.txt=/dev/stdin`

+ Edit the ArgoCD instance inside the Gitops operator: \
```
repo:
  resources:
    limits:
      cpu: '1'
      memory: 1Gi
    requests:
      cpu: 250m
      memory: 256Mi
  # ADD THE FOLLOWING TO THE MANIFEST
  env:
   - name: XDG_CONFIG_HOME
     value: /.config
   - name: SOPS_AGE_KEY_FILE
     value: /.config/sops/age/keys.txt
  volumes:
   - name: custom-tools
     emptyDir: {}
   - name: sops-age
     secret:
       secretName: sops-age
  initContainers:
   - name: install-ksops
     image: viaductoss/ksops:v3.0.2
     command: ["/bin/sh", "-c"]
     args:
     - 'echo "Installing KSOPS..."; cp ksops /custom-tools/; cp $GOPATH/bin/kustomize /custom-tools/; echo "Done.";'
     volumeMounts:
       - mountPath: /custom-tools
         name: custom-tools
  volumeMounts:
   - mountPath: /usr/local/bin/kustomize
     name: custom-tools
     subPath: kustomize
   - mountPath: /.config/kustomize/plugin/viaduct.ai/v1/ksops/ksops
     name: custom-tools
     subPath: ksops
   - mountPath: /.config/sops/age/keys.txt
     name: sops-age
     subPath: keys.txt
  kustomizeBuildOptions: --enable-alpha-plugins
```

+ Wait for a new openshift-gitops-repo-server pod to be created and started: \
`$ oc get pod -n openshift-gitops -w | grep openshift-gitops-repo-server`

+  Create the SOPS configuration file: \
```
$ cat <<EOF > .sops.yaml
creation_rules:
  - path_regex: apps/.*\.sops\.ya?ml
    encrypted_regex: "^(data|stringData)$"
    # Use the public key found in the agekey file we generated in the first step of this guide:
    age: age1...
EOF
```

+ Create the local Kubernetes Secret: \
```
cat <<EOF > secret.sops.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  #FILL THE FOLLOWING FIELDS
  username: 
  password: 
EOF
```
+ Encrypt with SOPS CLI: \
`sops --age=age1... --config <(echo '') --encrypt --in-place secret.sops.yaml`

+ Define KSOPS kustomize Generator: \
```
cat <<EOF > secret-generator.yaml
apiVersion: viaduct.ai/v1
kind: ksops
metadata:
  # Specify a name
  name: example-secret-generator
files:
  - ./secret.sops.yaml
EOF
```

+ Create the kustomization manifest: \
```
cat <<EOF > kustomization.yaml
generators:
  - ./secret-generator.yaml
EOF
```

+ Push everything to the git repository you will use during the creation of the application on ArgoCD.
