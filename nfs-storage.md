# k8s 에서 nfs-storage 시도

* 아직 성공 못했습니다.
* virtualbox+vagrant 기반의 k8s 에서 시도하는 것입니다. 이 환경에서는 변변한 storageclass를 쓸 수가 없어서..

아직 성공 못했지만 했었던 것들을 적어 놓습니다. 성공하면 다시 깔끔하게 정리해야죠.
```
# k8s-head 에서
sudo apt install nfs-kernel-server
sudo mkdir -p /mnt/nfs_share
sudo chown -R nobody:nogroup /mnt/nfs_share/
sudo chmod 777 /mnt/nfs_share/
sudo cat >> /etc/exports <<EOF
/mnt/nfs_share  192.128.205.0/24(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
EOF
sudo exportfs -rav

# 모든 시스템 (k8s-head, k8s-node-1, k8s-node-2)
for i in k8s-head k8s-node-1 k8s-node-2 ; do \
  ssh vagrant@$i sudo apt install nfs-common ; \
  ssh vagrant@$i sudo mkdir -p /mnt/nfs ; \
  ssh vagrant@$i sudo mount -t nfs 192.128.205.10:/mnt/nfs_share  /mnt/nfs ; \
  done      
암호 열심히 넣고 Y 엔터치고...



# k8s-head에서
git clone https://exxsyseng@bitbucket.org/exxsyseng/nfs-provisioning.git    # 뭔가 미심쩍은데, 원본이 어디에 있을 듯.
# 원본으로 추정되는 위치: https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client/deploy
cd nfs-provisioning
kubectl create -f rbac.yaml
kubectl get clusterrole,clusterrolebinding,role,rolebinding | grep nfs  # 확인

cat > nfs-storageclass.yaml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: skmta.net/nfs
parameters:
  archiveOnDelete: "false"
EOF

kubectl create -f nfs-storageclass.yaml
```
