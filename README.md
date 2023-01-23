# RWX K8s PVC volumes with Compute instance

- This is a solution for creating RWX support for OVH K8s clusters, as OVH does not support RWX volumes yet.
- OVH suggests to use their NAS-HA solution in the meantime
  - This is expensive (minimum amount of GB)
  - Not well integrated: You have to whitelist the IPs yourself and you have to manually whitelist IPs when Nodescalling happens. This causes production downtime in the worst case!

# Prerequisites

Your OVH K8s cluster must be connected to a ovh public cloud private network

# 1) Compute Instance: Ubuntu + NFS Server

- Create an Ubuntu compute instance in the OVH public cloud project connected to the same private network as your kubernetes cluster

- Install NFS Server on your machine
```shell
sudo apt-get install nfs-kernel-server
```

- Create directory for our NFS storage
```shell
sudo mkdir -p /srv/nfs
sudo chown nobody:nogroup /srv/nfs
sudo chmod 0777 /srv/nfs
```

- Allow ONLY your Private Network subnet access to the NFS Server:
- IMPORTANT: Replace '1.2.3.4/16' with your own subnet!
```shell
sudo mv /etc/exports /etc/exports.bak
echo '/srv/nfs 1.2.3.4/16(rw,sync,no_subtree_check)' | sudo tee /etc/exports
```

- Restart the NFS Service
```shell
sudo systemctl restart nfs-kernel-server
```

# 2) Setting up the Kubernetes part to use the NFS Server

- Install NFS CSI driver
```shell
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm repo update
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs --namespace kube-system --set kubeletDir=/var/lib/kubelet
```

- Check if everything is running
```shell
kubectl get pods -nkube-system
kubectl get csidrivers
```

- Add StorageClass type / Connect NFS server (see file storageClass.yaml)
- IMPORTANT: Replace '1.2.3.4' with the private network IP of your Compute Instance in the storageClass.yaml file!
```shell
kubectl apply -f storageClass.yaml
```

# 3) Short test

- Short test: Start 30 hello-world Pods with an example PVC
```shell
kubectl apply -f testRwxPvc.yaml
```

- Check if all 30 Pods are in Running state

- Clean up:
```shell
kubectl delete namespace rwxtest
```

# 4) Backup

- Please do not forget to activate instance backup when using this in production
