# Configuring RBAC

Walk through a series of steps, testing how RBAC is configured and changing the
configuration.

1. Accessing the API server from inside a pod.

    Start an nginx container in the cluster.

    ```sh
    kubectl run nginx --image=nginx:latest
    ```

    Get a shell inside the running container.

    ```sh
    kubectl get pods | grep "nginx"
    kubectl exec -it <pod_id_from_previous_command> bash
    ```

    Test the accessibility of the Kubernetes API with some curl commands from
    inside a container in the cluster.

    `curl` isn't available in this container by default, so install it.

    ```sh
    apt-get update && apt-get install -y curl
    curl -ik https://kubernetes/version
    ```

    Every container in a cluster is populated with a token that can be used for
    authenticating to the API server:

    ```sh
    cat /var/run/secrets/kubernetes.io/serviceaccount/token
    ```

    Using that token, check the version of the API server (the equivalent of
    `kubectl version`).

    ```sh
    curl -ik -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes/version
    ```

    Now, something that requires more permissions, listing the pods running in
    the default namespace:

    ```sh
    curl -ik -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes/api/v1/namespaces/default/pods
    ```

2. Enable RBAC. At this point, if you installed your Kubernetes cluster using
   some other mechanism you may find some, or all, of this configuration
   already present.

    * On v1.5.x, RBAC is v1alpha1, or v1.6.x, RBAC is v1beta1. Make sure the
      value of `--runtime-config` matches the version of your Kubernetes
      cluster.

    ```sh
    sudo bash
    cp /etc/kubernetes/manifests/kube-apiserver.json ~
    vi /etc/kubernetes/manifests/kube-apiserver.json
    ```

    Add configuration for RBAC.

    ```
    "--authorization-mode=RBAC",
    "--runtime-config=rbac.authorization.k8s.io/v1alpha1",
    "--authorization-rbac-super-user=admin"
    ```

    If these parameters are not just right, the API server will not come back up.
    `kubectl` commands will respond with an error, and `docker ps | grep api` will
    have no results.

    Sometimes, even with correct parameters, the API server does not come back up.
    In those cases, restarting kubelet appears to fix the problem:

    ```sh
    systemctl restart kubelet
    ```

3. Basic RBAC related commands for `kubectl`. Note, these commands will not
   give errors if RBAC isn't configured.

    ```sh
    kubectl get roles
    kubectl get rolebindings
    kubectl get clusterroles
    kubectl get clusterrolebindings
    kubectl get clusterrole system:node -o yaml
    ```

4. See what it looks like when an improperly authenticated user connects to the
   API.

    * Note: In some installations of 1.6, the API server uses 443 for the secure
      port on the master instead of 6443 in the examples below.

    Run this command on the cluster master.

    ```
    curl -kv -H "Authorization: Bearer not-a-real-token" https://localhost:6443/api/v1/nodes
    ```

5. Add users with tokens so there is someone to bind roles to.

    In the `/etc/kubernetes/manifests/kube-apiserver.json`, there is a
    configuration option, `--token-auth-file` that specifies a file containing
    a list of user to authenticate. In production it is normal to use some
    other mechanism for registering users with Kubernetes, but for simplicity
    of demonstration, this guide uses the token file for managing users.

    Add these lines to `/etc/kubernetes/pki/tokens.csv`:

    ```
    gjsecret,goodjacob,1,"group1,group2"
    bjsecret,badjacob,2,group3
    ```

    Changes to the token file don't take effect automatically, so restart the API
    server:

    ```sh
    docker kill $(docker ps --filter name=kube-apiserver -q)
    ```

6. Test API connections again.

    Authentication is working:

    ```
    kubectl get nodes --v=10
    kubectl --insecure-skip-tls-verify=true --server=https://127.0.0.1:6443 --token=gjsecret get pods --v=10
    curl -ki -H "authorization: bearer gjsecret" https://localhost:6443/version
    ```

    But we are still not authorized for everything:

    ```sh
    curl -ki -H "Authorization: Bearer gjsecret" https://localhost:6443/api/v1/namespaces/default/pods
    ```

7. Add a role. Put this in a new file `pod-reader-role.yml`.

    ```
    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1alpha1
    metadata:
      namespace: default
      name: pod-reader-role
    rules:
      - apiGroups: [""] # The API group "" indicates the core API Group.
        resources: ["pods"]
        verbs: ["get", "watch", "list"]
    ```

    ```sh
    kubectl apply -f pod-reader-role.yml
    kubectl get roles
    ```

8. Add a role binding. Put this in a new file `pod-reader-rolebinding.yml`.

    ```
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1alpha1
    metadata:
      name: read-pods-rolebinding
      namespace: default
    subjects:
      - kind: User # May be "User", "Group" or "ServiceAccount"
        name: goodjacob
    roleRef:
      kind: Role
      name: pod-reader-role
      apiGroup: rbac.authorization.k8s.io
    ```

    ```sh
    kubectl apply -f pod-reader-rolebinding.yml
    kubectl get rolebindings
    ```

    You can confirm the configuration of the role binding:

    ```sh
    kubectl get rolebindings read-pods-rolebinding -o yaml
    ```

9. And now, the user that was denied before, is working:

    ```sh
    curl -ki -H "Authorization: Bearer gjsecret" https://localhost:6443/api/v1/namespaces/default/pods
    ```

    ```sh
    curl -ki -H "Authorization: Bearer bjsecret" https://localhost:6443/api/v1/namespaces/default/pods
    ```

10. Back at the beginning, we showed how code inside a pod can access the API:

    ```sh
    kubectl get pods | grep "nginx"
    kubectl exec -it <pod_id_from_previous_command> bash
    curl -ik -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes/version
    curl -ik -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes/api/v1/namespaces/default/pods
    ```

11. Add a role binding. Put this in a new file `default-pod-reader-rolebinding.yml`.

    ```
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1alpha1
    metadata:
      name: default-read-pods-rolebinding
      namespace: default
    subjects:
      - kind: ServiceAccount # May be "User", "Group" or "ServiceAccount"
        name: default
    roleRef:
      kind: Role
      name: pod-reader-role
      apiGroup: rbac.authorization.k8s.io
    ```

    ```sh
    kubectl apply -f default-pod-reader-rolebinding.yml
    ```

