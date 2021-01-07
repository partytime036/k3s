
# 12/30災難篇，查看k3s細項

原因:coredns一直處於重啟狀態，名稱無法解析，癱瘓整個k3s


```bash=
bigred@LCS81:~$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
mdp2021   1/1     Running   1          19h
nna       1/1     Running   0          54m
adm100    1/1     Running   0          54m
wka02     1/1     Running   0          54m
wka01     1/1     Running   0          54m
wka03     1/1     Running   0          54m
ds101     1/1     Running   0          54m
rma       1/1     Running   0          54m
bigred@LCS81:~$ kubectl get nodes kube-system
Error from server (NotFound): nodes "kube-system" not found
bigred@LCS81:~$ kubectl get pods kube-system
Error from server (NotFound): pods "kube-system" not found
bigred@LCS81:~$ kubectl get pod kube-system
Error from server (NotFound): pods "kube-system" not found
bigred@LCS81:~$ kubectl get kubectl get pod -n kube-system
error: the server doesn't have a resource type "kubectl"
bigred@LCS81:~$ kubectl get pod -n kube-system
NAME                                     READY   STATUS      RESTARTS   AGE
helm-install-traefik-ntt9d               0/1     Completed   0          4d12h
svclb-traefik-clh7n                      2/2     Running     14         4d12h
svclb-traefik-58dgc                      2/2     Running     16         4d12h
metrics-server-7b4f8b595-l7wsm           1/1     Running     7          4d12h
svclb-traefik-wkv9g                      2/2     Running     12         4d12h
traefik-5dd496474-jgjp7                  1/1     Running     8          4d12h
local-path-provisioner-7ff9579c6-b4n4f   1/1     Running     11         4d12h
svclb-traefik-tr5t4                      2/2     Running     14         4d12h
svclb-traefik-ttrc7                      2/2     Running     14         4d12h
svclb-traefik-dhnz9                      2/2     Running     14         4d12h
coredns-66c464876b-rst8q                 1/1     Running     18         4d12h
bigred@LCS81:~$ kubectl get all -n kube-system
NAME                                         READY   STATUS      RESTARTS   AGE
pod/helm-install-traefik-ntt9d               0/1     Completed   0          4d13h
pod/svclb-traefik-clh7n                      2/2     Running     14         4d13h
pod/svclb-traefik-58dgc                      2/2     Running     16         4d13h
pod/metrics-server-7b4f8b595-l7wsm           1/1     Running     7          4d13h
pod/svclb-traefik-wkv9g                      2/2     Running     12         4d13h
pod/traefik-5dd496474-jgjp7                  1/1     Running     8          4d13h
pod/local-path-provisioner-7ff9579c6-b4n4f   1/1     Running     11         4d13h
pod/svclb-traefik-tr5t4                      2/2     Running     14         4d12h
pod/svclb-traefik-ttrc7                      2/2     Running     14         4d12h
pod/svclb-traefik-dhnz9                      2/2     Running     14         4d12h
pod/coredns-66c464876b-rst8q                 1/1     Running     19         4d13h

NAME                         TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
service/kube-dns             ClusterIP      172.20.0.10    <none>          53/UDP,53/TCP,9153/TCP       4d13h
service/metrics-server       ClusterIP      172.20.0.67    <none>          443/TCP                      4d13h
service/traefik-prometheus   ClusterIP      172.20.0.122   <none>          9100/TCP                     4d13h
service/traefik              LoadBalancer   172.20.0.233   120.96.143.81   80:30557/TCP,443:30760/TCP   4d13h

NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/svclb-traefik   6         6         6       6            6           <none>          4d13h

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/metrics-server           1/1     1            1           4d13h
deployment.apps/traefik                  1/1     1            1           4d13h
deployment.apps/local-path-provisioner   1/1     1            1           4d13h
deployment.apps/coredns                  1/1     1            1           4d13h

NAME                                               DESIRED   CURRENT   READY   AGE
replicaset.apps/metrics-server-7b4f8b595           1         1         1       4d13h
replicaset.apps/traefik-5dd496474                  1         1         1       4d13h
replicaset.apps/local-path-provisioner-7ff9579c6   1         1         1       4d13h
replicaset.apps/coredns-66c464876b                 1         1         1       4d13h

NAME                             COMPLETIONS   DURATION   AGE
job.batch/helm-install-traefik   1/1           20s        4d13h
bigred@LCS81:~$
```

************************************************
k3s安裝完之後會有traefik、coredns、svclb、local-path-provisioner、metrics-server

kubectl get all -n kube-system看細項

replicaset就是controller負責照顧的pod包括有coredns

*************************************************
先前為了要解決使用名稱互PING問題我們把每個POD的中的resolv.conf的DNS(開機時搜尋的位子)貼到MASTER中的resolv.conf導致整個系統重啟時DNS的混亂
## 解決方案

```bash=
bigred@LCS91:~$ sudo nano /etc/resolv.conf
```
```bash=
search default.svc.dt.io svc.dt.io dt.io mcu.edu.tw hdp.default.svc.dt.io
nameserver 172.30.0.10
options ndots:5


#search mcu.edu.tw
#nameserver 120.96.81.1
#nameserver 120.96.80.1
#nameserver 168.95.1.1
nameserver 8.8.8.8
```
************************
下午發現DNS還是不太穩定
************************
## 再次解決
解決方式:把主機nameserver往上修改使其高於
POD的nameserver
```bash=
search default.svc.dt.io svc.dt.io dt.io mcu.edu.tw hdp.default.svc.dt.io
nameserver 8.8.8.8
nameserver 172.30.0.10
options ndots:5


#search mcu.edu.tw
#nameserver 120.96.81.1
#nameserver 120.96.80.1
#nameserver 168.95.1.1
```
### 12/31 實測後狀況暫時正常 持續觀察




