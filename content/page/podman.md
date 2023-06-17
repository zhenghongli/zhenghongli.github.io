---
title: 淺談 Podman 與使用分享
date: 2022-07-23
publishdate: 2017-03-24
categories:
- Container
tags:
- Podman
keywords:
- tech
comments:       false
showMeta:       false
showActions:    false
---

在容器技術蓬勃發展的年代，許多人都想要來分一杯羹，因此也不乏許多新的容器技術。
Podman作為RedHat推出的產品，是一個RHEL OS原生tool讓你方便的在OCI規範下去使用run,build,deploy等指令操作Continaer。以下列出幾個特點。
<!--more-->

* Daemonless
* 指令與Docker相似
* 可以在root or non-privileged user執行(會有各自的storage以及user)
* 支援RESTFul API 方式操作Podman(需要自行設置啟動)

我們可以過架構圖很好的比較Docker與Podman，很明顯Redhat想打造自己的生態圈，除了Podman之外還包含了Skopeo、Buildah，讓build image功能以及對registry的操作分開，讓Podman能夠更專注在container上面。
![](https://i.imgur.com/5nrIg4p.jpg)
(圖片擷取自 https://ios.dz/from-docker-to-podman/)

### OCI Projects Plans

Podman 使用了OCI規範以及各種的lib去建構，以下是他在各個面向中所使用的不同技術:
* Runtime: 可以支援OCI相關的runtime像是runc以及crun。
* Images: 管理image是使用[containers/image](https://github.com/containers/image)，另外skopeo也是使用此lib去撰寫的，並且擁有其他更便利的功能。
* Storage: 容器與image儲存的管理是使用 [containers/storage](https://github.com/containers/storage)。
* Networking: 網路的架構使用 Netavark 以及 Aardvark專案，Rootless Network namespces 則是使用slirp4netns專案。此外它也支援CNI。
* Builds: Podman本身自帶Build指令的Source code是從Buildah來的，不過更多其他的功能還是Buildah本身才有辦法使用。
* Conmon: Conmon 是監控 OCI containers的工具，除了Podman之外也被使用在CRI-O，。
* Seccomp: 可定義Policy對Podman, Buildah, and CRI-O進行System call等權限控管。

## 不同容器技術的關聯
下圖是一張很棒的架構圖，是從某一篇文章的作者擷取過來，有興趣可以參考連結閱讀原文。作者將相關的技術都列出加以分類，這邊我簡略介紹各層級的作用
* High-level container management: 管控指令介面，可以透過CLI or API or CRI的方式呼叫High-level runtime
* High-level container runtime: 就是實作OCI的規範(Image spec, Runtime spec)產出bundle以及config給low-level runtime執行。此外network相關設定也是這層所建立的，包含創建bridge、veth，以及連結到namespace。
* Low-level container runtime: 根據Image產出的config建立對應的cgroup、namespace環境，以及載入bundle到環境內。
![](https://i.imgur.com/OXywAao.jpg)
(圖片擷取自: https://merlijn.sebrechts.be/blog/2020-01-docker-podman-kata-cri-o/)

### CRI-O 與 Podman 的關聯
因為Podman是使用libpod做的，而libpod是從CRI-O分出來的，所以可能會有人誤會Podman可以直接提供給Kubernetes作為CRI使用，為釐清他們之間的關聯，我們可以透過流程圖來理解他們之間的關係。根據圖上所示CRI-O是透過CRI協定呼叫，並且使用containers/image以及runc去執行出一個container環境，而Podman則是自己直接實作溝通的方式，因此Podman的指令就只有自己使用，而不能透過CRI操作Podman，但因為Podman與CRI-O都是使用containers/image、containers/storage作為lib，所以透過sudo podman images與crictl images可以看到相同的結果。
![](https://i.imgur.com/T4yoWyo.png)
(圖片擷取自  https://cloud.redhat.com/hubfs/Imported_Blog_Media/Contianer-Standards-Work-Podman-vs_-CRICTL.png)


以下是我的環境，同時安裝了CRI-O以及Podman，並且安裝了K8S，因此可以看到都有K8S相關的images

crictl指令 :
`crictl images | gre k8s`
![](https://i.imgur.com/MmweFp7.png)

podman指令(請自行切換成root or 使用sudo) :
`podman images | grep k8s`
![](https://i.imgur.com/zdmyyTR.png)

## 實際操作

### 環境
* OS:RHEL 8.4
* RAM: 8 Gb
* CPU: 4 Core

### Podman 安裝
1. yum安裝
`sudo yum -y install podman`
2. 版本檢查
`podman --verison`
3. 檢查狀態
`podman info`



### Podman CLI
1. 拉取Image
`podman pull <image>`
如有多個來源的話可以選擇從哪個registry抓取
![](https://i.imgur.com/0fYc9dT.png)
2. 顯示Image
`podman images`
![](https://i.imgur.com/wHS69SR.png)
3. 執行Container
`podman run -d -p 8080:80 nginx`
用curl檢查Container服務
`curl -I localhost:8080`
![](https://i.imgur.com/KGngpyt.png)
4. 顯示執行的image
`podman ps`
![](https://i.imgur.com/KEzCcbT.png)
5. 檢查running cintainer
`podman inspect <image-id/name>`
6. 停止 container
`podman stop <image-id/name>`
7. 刪除 container
`podman rm <image-id/name>`


### Registry 設定
增加Harbor ca.crt憑證
```
# downlaod harbor ca
HARBOR_DOMAIN={YOUR_HARBOR_CORE_DOMAIN}
curl https://${HARBOR_DOMAIN}/api/v2.0/systeminfo/getcert  --insecure --output ca.crt

# move cat to trust anchors
sudo mv ca.crt /etc/pki/ca-trust/source/anchors
# update trust list
sudo update-ca-trust
```
registry來源清單設定
```
sudo vim /etc/containers/registries.conf
# unqualified-search-registries 指定使用內網的 registry
# 將自己的registry加入清單中
unqualified-search-registries = ['registry.access.redhat.com', 'registry.redhat.io', 'docker.io', 'local.registry.example']
```
重啟服務
```
sudo systemctl daemon-reload
sudo systemctl restart crio
```

## 資料參考
* https://middlewaretechnologies.in/2020/11/how-to-use-podman-rest-api-service-to-query-and-manage-linux-containers-system.html
* https://github.com/containers/podman/blob/main/docs/tutorials/podman_tutorial.md
* https://www.jianshu.com/p/2a58136886ba
* https://dywang.csie.cyut.edu.tw/dywang/rhcsa8/node156.html
* https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/sec-shared-system-certificates
* https://ios.dz/from-docker-to-podman/
* https://faun.pub/kubernetes-story-deep-into-container-runtime-db1a41ed2132,
* https://ithelp.ithome.com.tw/articles/10216880
* https://www.raygecao.cn/posts/oci-distribution/
* https://www.readfog.com/a/1668597455567032320