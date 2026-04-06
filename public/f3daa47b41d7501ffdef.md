---
title: IPv6のみポート開放しているとかいう特殊環境でウェブサイトをArgoCDを使ってデプロイした話
tags:
  - Linux
  - Web
  - '#k8s'
  - '#MicroK8s'
private: false
updated_at: '2026-04-05T23:21:17+09:00'
id: f3daa47b41d7501ffdef
organization_url_name: unipro-tech
slide: false
ignorePublish: false
---
まずはsnapをインストール
```
sudo snap version
sudo apt update
sudo apt install snap
```
次に設定ファイルを設置
```
mkdir -p /var/snap/microk8s/common/
vi /var/snap/microk8s/common/.microk8s.yaml
```
中身は、
```
version: 0.1.0
extraCNIEnv:
  IPv4_SUPPORT: true
  IPv4_CLUSTER_CIDR: 10.3.0.0/16
  IPv4_SERVICE_CIDR: 10.153.183.0/24
  IPv6_SUPPORT: true
  IPv6_CLUSTER_CIDR: fd02::/64
  IPv6_SERVICE_CIDR: fd99::/108
extraSANs:
  - 10.153.183.1
```
---

## 1. ネットワーク構成 (extraCNIEnv)
ここでは、クラスター内部で使用するIPアドレスの「範囲（予約席）」を決めています。

| 項目 | 値 | 意味 |
| :--- | :--- | :--- |
| **IPv4_CLUSTER_CIDR** | `10.3.0.0/16` | **ポッド(Pod)用**のIPv4アドレス範囲。最大約65,000個のIPをポッドに割り当てられます。 |
| **IPv4_SERVICE_CIDR** | `10.153.183.0/24` | **サービス(Service)**用のIPv4アドレス範囲。ロードバランサー的な役割の仮想IPに使われます（254個分）。 |
| **IPv6_CLUSTER_CIDR** | `fd02::/64` | **ポッド(Pod)用**のIPv6アドレス範囲。ほぼ無限に近いアドレス空間です。 |
| **IPv6_SERVICE_CIDR** | `fd99::/108` | **サービス(Service)**用のIPv6アドレス範囲。 |

> [!NOTE]
> **CIDR（サイダー）とは？**
> `10.3.0.0/16` の後ろの数字は「固定する桁数」を表します。数字が小さいほど、そのネットワーク内で使えるIPアドレスの数は多くなります。

---

## 2. 証明書の追加設定 (extraSANs)
**SANs**（Subject Alternative Names）は、クラスターのAPIサーバーなどに安全にアクセスするために「この名前（またはIP）からのアクセスは正当なものです」と証明書に書き加えるリストです。

* **値:** `10.153.183.1`
* **意味:** 通常、これは **IPv4_SERVICE_CIDR の最初のIP** です。Kubernetes内部では、このIPが `kubernetes.default.svc` という名前のサービス（APIサーバーへの入り口）として予約されるため、証明書がこのIPを「自分自身の身分」として認められるように設定しています。

内容は適宜変更してください

これで設定出来たら

```
snap install microk8s --classic
```
でインストールします

# 4.CalicoのIPv6対応とIPプール設定

```
microk8s kubectl patch ippool default-ipv6-ippool --type=merge -p '{"spec":{"natOutgoing":true}}'
```
を実行します
その後、再起動し、確認。
```
microk8s kubectl delete pod -n kube-system -l k8s-app=calico-node
```
```
microk8s kubectl get ippools.crd.projectcalico.org -o yaml
```

# 5.kube-proxy/kubeletのデュアルスタック設定
```
cat /var/snap/microk8s/current/args/kube-proxy
```
- # kubeletの設定
```
vi /var/snap/microk8s/current/args/kubelet
```
を開いたら、
```
--node-ip=ホストのIPv4アドレス,ホストのIPv6アドレス
```
を追記します
```
snap restart microk8s.daemon-kubelite
```
で再起動します
jqをインストールして、
```
snap install jq
```
```
microk8s kubectl get nodes -o jsonpath='{.items[*].status.addresses}' | jq
```

# 6.Metallb周りの設定
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```
- IPAddressPoolの作成
IPv4用、IPv6用をそれぞれ作って適用。
```
vi ipaddresspool-v4.yaml
```
```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: my-ip-pool-v4
  namespace: metallb-system
spec:
  addresses:
  - 192.168.10.240-192.168.10.250
```
```
kubectl apply -f ipaddresspool-v4.yaml```
```
反映。IPv6も同じ要領で
```
vi ipaddresspool-v6.yaml
```
```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: my-ip-pool-v6
  namespace: metallb-system
spec:
  addresses:
    - 任意のIPv6セグメント
  autoAssign: true
  ```
  反映。
  ```
  kubectl apply -f ipaddresspool-v6.yaml
  ```
  以下のコマンドで設定したIPが出ればok
  ```
  kubectl get ipaddresspool -n metallb-system
  ```
- # L2Advertisementの作成
```
vi l2advertisement.yaml
```
```
  apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
    - my-ip-pool-v4
    - my-ip-pool-v6
```
反映
```
kubectl apply -f l2advertisement.yaml
```
以下のコマンドで設定したプールが二つとも出ればok
```
kubectl get l2advertisement -n metallb-system
```

# 7.ロードバランサーの設定

IPv4,6共に固定したいのでサービスを作成

- IPv4用
```
vi bind-service-v4.yaml
```
```
apiVersion: v1
kind: Service
metadata:
  name: dns-service-v4
spec:
  selector:
    app: dns-bind
  ports:
    - name: dns-tcp
      protocol: TCP
      port: 53
      targetPort: 53
    - name: dns-udp
      protocol: UDP
      port: 53
      targetPort: 53
  type: LoadBalancer
  loadBalancerIP: 172.16.200.50
```
反映。
```
kubectl apply -f bind-service-v4.yaml
```
- IPv6用
```
vi bind-service-v6.yaml
```
```
apiVersion: v1
kind: Service
metadata:
  name: dns-service-v6
spec:
  selector:
    app: dns-bind
  ports:
    - name: dns-tcp
      protocol: TCP
      port: 53
      targetPort: 53
    - name: dns-udp
      protocol: UDP
      port: 53
      targetPort: 53
  type: LoadBalancer
  ipFamilies:
    - IPv6
  ipFamilyPolicy: SingleStack
  loadBalancerIP: 240b:13:55e1:3200:200::50
  ```
これまた反映
```
kubectl apply -f bind-service-v6.yaml
```
以下のコマンドで設定した値になっていればok
```
kubectl get service
```
# 8.ArgoCDまわりをつつく
```
mkdir k8s && cd k8s
nano applications.yaml
```
yamlを作って、
```
kubectl apply -f applications.yaml
```
反映する。
```
microk8s kubectl create namespace argocd

microk8s kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Argocd等を有効化。
cert-manager等も必要であれば。
```
microk8s kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```
```
nano ClusterIssuer.yaml
```
```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: あなたのメール@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
```
```
microk8s kubectl apply -f clusterissuer.yaml
```
証明書の取得が済んだら完成。

# 終わりに
お疲れ様でした！
なかなかめんどくさい環境での作業ですが環境に対応できるとかなり幅が広がるかなと思います！
