---
title: LVM Extent Volume
date: 2022-07-23
publishdate: 2017-03-24
categories:
- Storage
tags:
- LVM
keywords:
- tech
comments:       false
showMeta:       false
showActions:    false
---

LVM (Logical Volume Manager)。Linux Kernel提供的邏輯磁區管理器，主要提供動態調整磁區功能，當磁區空間不足時，可以利用其他硬碟或未切割硬碟的空間進行調整，而非對原先的磁區操作，因此調整時不會影響原先磁區作業。
<!--more-->

![](https://i.imgur.com/02hOokn.png)

## 架構圖
### 名詞介紹:
* Physical Volume(PV) : 硬體切割出來的Partition，經轉換後變成PV。
* Volume Group(VG) : PV組成的Group，可視為多個硬碟Parition組成的一個大磁區。
* Physical Extent (PE) : LVM blocksize，通常代表LV可使用的空間。
* Logical Volume (LV) : 實際從VG切割除來的空間，即系統可直接使用之空間。通常名稱為/dev/mapper/<lv-name>

P.S. 在此提醒，若是想要縮減 root的操作，請另外掛載一個作業系統進行操作，否則可以能影響到現在的作業系統。

LVM擴展操作
觀看目前硬碟Partition狀態
`sudo lsblk`
![](https://i.imgur.com/PzlSKlX.png)

`sudo fdisk -l`
![](https://i.imgur.com/XmAw6KG.png)

觀察目前有多少PV
`sudo pvscan`
![](https://i.imgur.com/rTBU32v.png)

觀看VG
`sudo vgdisplay`
![](https://i.imgur.com/Ys5C9ZH.png)

根據lsblk可得知目前硬碟是5顆硬碟做raid5，並且總共有3.3T空間，被切成三個Partition磁區，透過pvscan可以得知最後一個Parition md125p3已做為PV使用，vgdisplay表示md125p3已加入VG，並且可用空間還有2.24T，因此後續若要增加空間即可用現有的VG，不用在新增新的Partition->PV->VG的流程。
## LVM擴展Partition指令
`sudo lvextend –L +1000G /dev/mapper/Ubuntu—vg-ubuntu—lv`
對File System Resize
`Resize2fs /dev/mapper/Ubuntu—vg-ubuntu—lv`
原先空間:
![](https://i.imgur.com/dMtPLWk.png)
擴展後的空間:
![](https://i.imgur.com/tHTlxPu.png)

 
Reference
https://wiki.itcollege.ee/index.php?title=File:Lvm%271.jpg&limit=20
https://ithelp.ithome.com.tw/articles/10081243
https://www.golinuxcloud.com/resize-root-lvm-partition-extend-shrink-rhel/#Step_2_Boot_into_rescue_mode

