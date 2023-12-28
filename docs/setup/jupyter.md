# JyputerHub installation
## Kubernetes storage:
- Reference: https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner#with-helm
```bash
sudo mkdir /mnt/nfs

helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=172.16.0.5 \
    --set nfs.path=/mnt/nfs-ehn1/

kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

- Check:
```bash
kubectl get storageclass
```

- Change the ReclaimPolicy
```bash
kubectl get storageclass nfs-client -o yaml > storage-config.yaml
```
- Edit a `reclaimPolicy` field to `Retain`, and update the config
```bash
kubectl replace -f storage-config.yaml --force
```

## Create a shared storage
- Create a config file
```bash
nano pvc-shared-folder.yaml
```
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jupyterhub-shared-volume
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```
- Create PVC for the shared folder
```bash
kubectl apply -f pvc-shared-folder.yaml --namespace=hub
```

## Install JupyterHub Helm chart
- Config file:
  - `client_secret`: `client_token` of OAuth Plugin in Moodle
  - Change the IP and port appropriately
  - In `Authenticator/admin_users`, list of accounts which have the admin permission.
```yaml
hub:
  revisionHistoryLimit:
  config:
    GenericOAuthenticator:
      client_id: JupyterHub
      client_secret: 92745f3e63c073b4170700e773a5ccdadd821f2f811ea597
      oauth_callback_url: http://192.168.33.5/hub/oauth_callback
      authorize_url: http://192.168.33.5:8888/moodle/local/oauth/login.php?client_id=jupyterhub&response_type=code
      token_url: http://192.168.33.5:8888/moodle/local/oauth/token.php
      userdata_url: http://192.168.33.5:8888/moodle/local/oauth/user_info.php
      #userdata_method: "GET"
      scope:
        - user_info
    Authenticator:
      admin_users:
        - admin
    JupyterHub:
      authenticator_class: generic-oauth
  networkPolicy:
    enabled: false

# singleuser relates to the configuration of KubeSpawner which runs in the hub
# pod, and its spawning of user pods such as jupyter-myusername.
singleuser:
  allowPrivilegeEscalation: false
  storage:
    type: dynamic
    static:
      pvcName:
      subPath: "{username}"
    capacity: 10Gi
    homeMountPath: /home/jovyan
    dynamic:
      storageClass:
      pvcNameTemplate: claim-{username}{servername}
      volumeNameTemplate: volume-{username}{servername}
      storageAccessModes: [ReadWriteOnce]
    extraVolumes:
      - name: jupyterhub-shared
        persistentVolumeClaim:
          claimName: jupyterhub-shared-volume
    extraVolumeMounts:
      - name: jupyterhub-shared
        mountPath: /home/jovyan/shared

  image:
    name: tlkh/ai-container
    tag: "05.23"
  startTimeout: 3600
  cpu:
    limit: 4
    guarantee: 1
  memory:
    limit: 4G
    guarantee: 1G
  extraResource:
    limits: {"nvidia.com/gpu": "1"}
    guarantees: {"nvidia.com/gpu": "1"}
  cmd: jupyterhub-singleuser
  profileList: []
  networkPolicy:
    enabled: false

scheduling:
  userScheduler:
    enabled: false

cull:
  enabled: true
  users: true # --cull-users
  adminUsers: true # --cull-admin-users
  timeout: 3600 # --timeout
  every: 600 # --cull-every
  concurrency: 10 # --concurrency
  maxAge: 0 # --max-age
```

- Install JupyterHub using Helm chart
```bash
helm repo add jupyterhub https://hub.jupyter.org/helm-chart/
helm repo update
helm upgrade --cleanup-on-fail   --install hub jupyterhub/jupyterhub   --namespace hub   --create-namespace   --values config.yaml --version 2.0
```

- The output should be
```bash
### Post-installation checklist

  - Verify that created Pods enter a Running state:

      kubectl --namespace=hub get pod

    If a pod is stuck with a Pending or ContainerCreating status, diagnose with:

      kubectl --namespace=hub describe pod <name of pod>

    If a pod keeps restarting, diagnose with:

      kubectl --namespace=hub logs --previous <name of pod>

  - Verify an external IP is provided for the k8s Service proxy-public.

      kubectl --namespace=hub get service proxy-public

    If the external ip remains <pending>, diagnose with:

      kubectl --namespace=hub describe service proxy-public

  - Verify web based access:

    You have not configured a k8s Ingress resource so you need to access the k8s
    Service proxy-public directly.

    If your computer is outside the k8s cluster, you can port-forward traffic to
    the k8s Service proxy-public with kubectl to access it from your
    computer.

      kubectl --namespace=hub port-forward service/proxy-public 8080:http

    Try insecure HTTP access: http://localhost:8080
```

- Config `apache2`
  - Create a new file
    ```bash
    sudo vi /etc/apache2/sites-available/jupyterhub.conf
    ```

  - Check port of JupyterHub
    ```bash
    kubectl get services -n hub
    ```
  - The output should be
    ```bash
    aimc@aimc-en1:~$ kubectl get services -n hub
    NAME           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
    hub            ClusterIP      10.110.122.85    <none>        8081/TCP       3d4h
    proxy-api      ClusterIP      10.103.58.117    <none>        8001/TCP       3d4h
    proxy-public   LoadBalancer   10.109.161.235   <pending>     80:30669/TCP   3d4h
    ```
  
  - Copy content and change the appropriate port (port displayed at the `proxy-public` service)
    ```bash
    <VirtualHost *:*>
        ProxyPreserveHost On
    
        RewriteEngine on
        RewriteCond %{HTTP:UPGRADE} ^WebSocket$ [NC]
        RewriteCond %{HTTP:CONNECTION} ^Upgrade$ [NC]
        RewriteRule .* ws://192.168.33.5:30669%{REQUEST_URI} [P] 

        ProxyPass / http://192.168.33.5:30669/
        ProxyPassReverse / http://192.168.33.5:30669/
        ProxyRequests on
        RequestHeader set X-Forwarded-Proto "http"

        ServerName localhost
    </VirtualHost>
    ```
  - Enable site
    ```bash
    sudo a2ensite jupyterhub.conf
    ```
  
  - Restart `apache2`
    ```bash
    sudo systemctl restart apache2
    ```
