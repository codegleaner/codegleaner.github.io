---
layout: post
title: "Installing GitLab Runner by GitLab GUI"
tags: troubleshooting
---

GitLab GUI 提供很多工具，讓我們點一下就自動裝到K8s。
裝完 Helm Tiller 後 (才能解鎖其他工具)，就在裝 GitLab Runner 踩雷了。

![](../../../assets/gitlab-runner/1.png)
GitLab 幫你裝的通通東西會放在 ```gitlab-managed-apps``` namespace，可以知道 GitLab Runner 叫作 install-runner。
```
kubectl get all -n gitlab-managed-apps
```
![](../../../assets/gitlab-runner/2.png)

去檢查 install-runner 的 log 吧~ 發現是 helm upgrade runner runner/gitlab-runner... 這個地方會出錯。

```
kubectl logs pod/install-runner -n gitlab-managed-apps
```
![](../../../assets/gitlab-runner/3.png)


因為 pod status 是 error，沒辦法```kubectl exec```，那就找出這個 pod 的 container 在做什麼，也就是取得```COMMAND_SCRIPT```。
```
kubectl describe pod/install-runner -n gitlab-managed-apps
```
![](../../../assets/gitlab-runner/4.png)

```
kubectl get pod/install-runner -n gitlab-managed-apps -o jsonpath="{.spec.containers[*].env[2].value}"
```

並紀錄爛掉的那行：
```
helm upgrade runner runner/gitlab-runner --install --reset-values --tls --tls-ca-cert /data/helm/runner/config/ca.pem --tls-cert /data/helm/runner/config/cert.pem --tls-key /data/helm/runner/config/key.pem --version 0.13.1 --set rbac.create\=true,rbac.enabled\=true --namespace gitlab-managed-apps -f /data/helm/runner/config/values.yaml
```

### 進入同樣的環境，才能知道為什麼那行會爛掉
做一點手腳，複製原來的環境存成 debug-runner.yml，並把執行```COMMAND_SCRIPT```出問題的那行```helm upgrade <release name> <chart path>```刪掉，再執行時 pod status 就變成 running。
```
kubectl get pod/install-runner -n gitlab-managed-apps -o yaml > debug-runner.yml
kubectl apply -f debug-runner.yml
```
![](../../../assets/gitlab-runner/5.png)

我們就可以進去 debug-runner pod 的 container 了。
```
sudo kubectl exec -it debug-runner -c helm -n gitlab-managed-apps -- sh
```

接著，我們思考一下 **failed to download runner/gitlab-runner** 是什麼意思，它要下載的這個 chartpath 的 ```runner``` 是指 repo，於是來檢查 ```runner``` repo 在不在？ 結果不在，那就幫它加上去。
```
helm repo list
```
![](../../../assets/gitlab-runner/6.png)

```
helm repo add runner https://charts.gitlab.io
```
![](../../../assets/gitlab-runner/7.png)

在確定這個 charts.gitlab.io 服務本身沒問題後，試著 ping charts.gitlab.io，發現遇到的問題是 DNS 查不到。

### 修改 DNS 設定，使得能加 https://charts.gitlab.io repo

debug-runner pod 裡面的 /etc/resolv.conf  的 nameserver 是 10.96.0.10，這個 cluster 只有 CoreDNS 當作 DNS Server，我們可以透過修改它的 configmap 解決這個問題。

![](../../../assets/gitlab-runner/8.png)

```
sudo kubectl edit cm coredns -n kube-system
```
![](../../../assets/gitlab-runner/9.png)

發現它的 DNS resolver 預設會 forward 到本地機器的 /etc/resolv.conf，所以我們到本地 /etc/resolv.conf 裡面加上```nameserver 8.8.8.8```，而不在 configmap 修改。
為了清掉 CoreDNS cache，我們把 coredns pod 刪掉，隨即 coredns deployment 會自動重啟新的 coredns pod。

```
kubectl get pods -n kube-system -o name | grep coredns | xargs kubectl delete -n kube-system
```

進去 debug-runner pod 測試看看。

```
nslookup charts.gitlab.io
```
![](../../../assets/gitlab-runner/10.png)

```
helm repo add runner https://charts.gitlab.io
```

![](../../../assets/gitlab-runner/11.png)

確定能加 repo 之後，把 debug-runner pod 和 install-runner pod 都刪掉，回去 GitLab GUI 再點一下 GitLab Runner Install 吧~
