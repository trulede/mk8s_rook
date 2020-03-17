### MicroK8s with privileged=true

Add --allow-privileged=true

    sudo vim /var/snap/microk8s/current/args/kube-apiserver
    microk8s.stop
    microk8s.start



## Rook NFS

https://rook.io/docs/rook/master/nfs.html

    git clone https://github.com/rook/rook.git
    cd rook/cluster/examples/kubernetes/nfs


### Setup

    apt install nfs-common


### NFS Operator and Share

    $ k create -f operator.yaml
    $ k -n rook-nfs-system get pod
    NAME                                    READY   STATUS    RESTARTS   AGE
    rook-nfs-operator-65548dcccf-dmqzp      1/1     Running   0          39s
    rook-nfs-provisioner-6b97f4d658-j9mq2   1/1     Running   0          38s

    $ k create -f nfs.yaml
    $ k -n rook-nfs get nfsservers.nfs.rook.io
    NAME       AGE
    rook-nfs   49s
    $ k -n rook-nfs get pod
    NAME         READY   STATUS    RESTARTS   AGE
    rook-nfs-0   1/1     Running   0          51s


### NFS Storage Class

    $ k create -f sc.yaml
    storageclass.storage.k8s.io/rook-nfs-share1 created
    $ k get sc
    NAME                          PROVISIONER               RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
    microk8s-hostpath (default)   microk8s.io/hostpath      Delete          Immediate           false                  186d
    rook-nfs-share1               rook.io/nfs-provisioner   Delete          Immediate           false                  7s


### NFS PVC

    $ k create -f pvc.yaml
    persistentvolumeclaim/rook-nfs-pv-claim created
    $ k get pvc
    NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
    rook-nfs-pv-claim   Bound    pvc-368a9ccc-440f-48b1-859f-d52832681ce4   1Mi        RWX            rook-nfs-share1   11s



## Rook Ceph

### Clear out previously generated keys.

Each time the operator is installed, it generates keys, so if starting again
you need to clear out the previous keys. Same with your Block device, need
to zero it out, or recreate.

    sudo rm -rf /var/lib/rook/


### The basics

    k create -f common.yaml
    k create -f operator.yaml
    k -n rook-ceph get pods

### Now the cluster, single node.

    k create -f cluster.yaml
    k -n rook-ceph get pods


### Now Ceph FS

https://rook.io/docs/rook/v1.2/ceph-filesystem.html

    k create -f filesystem.yaml

you want an OSD pod running

    $ k -n rook-ceph get pod
    NAME                                                      READY   STATUS      RESTARTS   AGE
    ...
    rook-ceph-osd-0-5f6b8549d9-skm2k                          1/1     Running     0          5m37s

    k create -f toolbox.yaml
    kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash
    $ ceph status


### Now the storage class and a pvc

    k create -f storageclass.yaml
    k create -f pvc.yaml
    k get pvc
    NAME                                                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
    cephfs-pvc                                                  Bound    pvc-5689434f-1f99-4a97-9f52-7a76f9326b72   1Gi        RWX            csi-cephfs          5s

    k describe pvc cephfs-pvc
    Name:          cephfs-pvc
    Namespace:     default
    StorageClass:  csi-cephfs
    Status:        Bound
    Volume:        pvc-5689434f-1f99-4a97-9f52-7a76f9326b72
    Labels:        <none>
    Annotations:   pv.kubernetes.io/bind-completed: yes
                   pv.kubernetes.io/bound-by-controller: yes
                   volume.beta.kubernetes.io/storage-provisioner: rook-ceph.cephfs.csi.ceph.com
    Finalizers:    [kubernetes.io/pvc-protection]
    Capacity:      1Gi
    Access Modes:  RWX
    VolumeMode:    Filesystem
    Mounted By:    <none>
    Events:
      Type    Reason                 Age                From                                                                                                             Message
      ----    ------                 ----               ----                                                                                                             -------
      Normal  ExternalProvisioning   30s (x2 over 30s)  persistentvolume-controller                                                                                      waiting for a volume to be created, either by external provisioner "rook-ceph.cephfs.csi.ceph.com" or manually created by system administrator
      Normal  Provisioning           30s                rook-ceph.cephfs.csi.ceph.com_csi-cephfsplugin-provisioner-d77bb49c6-qn5gk_56a0212d-3662-4462-9002-53b9043525db  External provisioner is provisioning volume for claim "default/cephfs-pvc"
      Normal  ProvisioningSucceeded  27s                rook-ceph.cephfs.csi.ceph.com_csi-cephfsplugin-provisioner-d77bb49c6-qn5gk_56a0212d-3662-4462-9002-53b9043525db  Successfully provisioned volume pvc-5689434f-1f99-4a97-9f52-7a76f9326b72
