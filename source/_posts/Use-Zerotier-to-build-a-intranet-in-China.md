---
title: 使用Zerotier基於國内網絡組建局域網
mathjax: true
copyright: true
comment: true
date: 2020-06-15 22:59:32
categories: 
- "Ops "
tags: 
- "Linux "
- "Network "
- "UDP "
- "P2P "
- "Zerotier "
---
<center><img src="https://img.madebug.net/m4d3bug/images-of-website/master/blog/ZeroTier.png" width=50% /></center>

<!-- more -->
# 什麽是Zerotier

   來自[官網](https://www.zerotier.com/manual/#1)的説明：

   > ZeroTier is a smart Ethernet switch for planet Earth.

   > It’s a distributed network hypervisor built atop a cryptographically secure global peer to peer network. It provides advanced network virtualization and management capabilities on par with an enterprise SDN switch, but across both local and wide area networks and connecting almost any kind of app or device.

# 如何使用

   ---

   1. [注冊](https://my.zerotier.com/)。

   2. 創建Network私有網絡並取得16位Network ID。

   3. 在apt系安裝zerotier：

      ```
      apt -y updateapt -y install curl sudo net-toolscurl -s https://install.zerotier.com/ | sudo bash
      ```

      其中遇到了關於gpg error的問題：

      ```
      The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 1657198823E52A61
      ```

      安裝gnupg來對其進行驗證導入再進行安裝。

      ```
      apt -y install gnupgapt-key adv --keyserver keyserver.ubuntu.com --recv-keys  1657198823E52A61curl -s https://install.zerotier.com/ | sudo bash*** Waiting for identity generation...*** Success! You are ZeroTier address [ xxxxxxxx ].
      ```

# 如何建立中轉服務器moon

   ---

   這是關於它的定義：

   > PLANET 行星級服務器，Zerotier位於各地的根服務器，有日本、新加坡等地。 MOON 衛星級服務器，用戶自建的私有根服務器，起到中轉加速的作用。 LEAF 節點級服務器，就是每臺連接到該網絡的機器節點。

   不得不驚嘆這個方案的明智之處 ，自帶支持中轉服務器的架設，這也是吸引我選擇它的初衷，p2p的組網方式本質是去中心化，不僅維護起來簡單，而且這種扁平化的網絡架構，其誕生迄今我一直認爲是一個計算機史上的思維方式的[奇異點](https://www.youtube.com/watch?v=sjx_rpay9rk)，也十分方便後期加入或者退出節點。

   1. 加入你的網絡，填入你的16位network ID。

      ```
      zerotier-cli join xxxxxxxxx
      ```

   2. 通過自身工具生成moon.json配置文件。

      ```
      cd /var/lib/zerotier-onezerotier-idtool initmoon identity.public > moon.json
      ```

   3. 將公網IP加入其中。

      ```
      cat moon.json { "id": "xxxxxxxx", "objtype": "world", "roots": [  {   "identity": "dd31a3ee4c:0:40971299f1f4875edfadfc221ffba68a283cf51a67a27fbc223da18b6257d236474b9f13e3e50b46c0ae8096339d3eb450e12ab0361bf5e8ef82c69acd86ebb8",   "stableEndpoints": []        # 把它改成"stableEndpoints": [ "xxx.xxx.xxx.xxx/9993" ]  } ], "signingKey": "592c748e40de1bd39db07bcdc98ad220ac82d67494483b410c0133939fc0c03059d3a2b2c9948ec1939c4d42e2c3f07b310a337bc47a77b94153e4f9f6df56d7", "signingKey_SECRET": "c2810d04474b737ada418e1407bdc49ea9a0879393e5cb8b585eaec0feadbb0dcbbc52c6c85db9db2a2d83b6a36c01ed839e105d67f7bd4e0ea8d22559f29fd6", "updatesMustBeSignedBy": "592c748e40de1bd39db07bcdc98ad220ac82d67494483b410c0133939fc0c03059d3a2b2c9948ec1939c4d42e2c3f07b310a337bc47a77b94153e4f9f6df56d7", "worldType": "moon"}
      ```

   4. 生成moon節點的籤名文件。

      ```
      zerotier-idtool genmoon moon.json
      ```

   5. 將該moon節點加入網絡。

      ```
      mkdir moons.dmv 000000xxxxxxxxxx.moon moons.d/
      ```

   6. 重啓zerotier-one服務即可完成moon節點的設置。

      ```
      systemctl restart zerotier-one.service
      ```

# 如何使用自建的moon節點

   ---

   1. 在其他leaf節點完成安裝后加入你的網絡，其後通過以下命令加入自建的moon節點。( ⚠️注意moon的id輸入兩次。)

      ```
      zerotier-cli orbit 000000xxxxxxxxxx 000000xxxxxxxxxx
      ```

   2. 其後在leaf節點查看加入情況。

      ```
      zerotier-cli listpeers|grep MOON
      ```

   3. 離開節點也很容易。( ⚠️注意moon的id輸入壹次。)

      ```
      zerotier-cli deorbit 000000xxxxxxxxxx
      ```

# 結語

   ---

   1. p2p組網天下第一，可以通過以下簡單的類比方式來理解其中原理，其實也是區塊鏈組網的基礎。

      > PLANET節點就是BT網絡中的根服務器 MOON節點就是BT網絡中的tracker服務器 LEAF節點就是BT網絡中客戶端的qBittorrent

   2. 組了一張大内網，我們來玩點什麽呢？

      - [ ] Kubernetes/Openshift，跨機房跨運營商實現多master多node集群，實現混合雲架構。
      - [ ] 區塊鏈實驗場景（其實沒想好）。
      - [ ] 個人專屬内網。
      - [ ] 滲透後階段，低調的後門維持方式，而且去中心化。

   3. 另外說一嘴，實驗下來，果然搬瓦工的cn2傳家寶還是沒救了，只能上trojan和squid了。
