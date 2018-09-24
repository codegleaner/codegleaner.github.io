---
layout: post
title: "使用Ansible對Cumulus Linux設定OSPF路由協定"
---
![](../../../assets/cumulus/objective_topology.png)

(註： 以下使用**粗體字**標示GNS3上模擬的設備)

## 目標說明
只需設定**ManagementStation**，讓**Leaf1**的**Bridge.100**成功*ping*到**Leaf2**的**Bridge.200**。

## 方法說明
在不使用Ansible(或Puppet、Chef等自動化工具)時，網管人員需要登入**Leaf1**設定網路介面、啟動介面、設定路由協定、啟動路由服務，再登入**Leaf2**設定網路介面、啟動介面、設定路由協定等，如果有100台Leaf就瘋了。

本篇介紹的方法會透過**ManagementStation**上的Ansible，將參數化的設定檔同時傳入**Leaf1**和**Leaf2**，並啟用路由相關服務。如此，當需要添增或移除Leaf時，網管人員只需要實際操作拔線、接線、開機，而不再需要個別登入設定。

## 環境準備
* Cumulus VX - **Leaf1**、**Leaf2**、**DHCP Server**的NOS (Networking Operating System)，請到官網下載OVA檔。
* CentOS 7 - **ManagementStation**的作業系統，請到官網下載ISO檔，再至VirtualBox新增VM，至少保留一個Network Adapter為NAT或Bridged Adapter，匯出OVA檔。
* Oracle VM VirtualBox - 匯入**Leaf1**、**Leaf2**、**DHCP Server**、**ManagementStation**的OVA檔。
* GNS3 - 網路模擬器，使用VirtualBox中的VM作為專案新增設備時的作業系統範本。

## 設備用途
* **Cloud** - 用來連接**ManagementStation**與實體主機(桌機或筆電)的網卡，讓**ManagementStation**可以隨時下載需要的工具。
* **ManagementStation** (CentOS 7) - 
  * 架設Web server - 用來放置ZTP Script與SSH公鑰。
  * 安裝Ansible - 用來配置與啟動**Leaf1**與**Leaf2**的路由協定。
* **DHCP Server** (Cumulus VX) - 架設DHCP server，用來動態分配IP與ZTP (Zero Touch Provisioning)給**Leaf1**與**Leaf2**。
* **Leaf1**與**Leaf2** (Cumulus VX) - 接線、開機即可。

(註： 環境請自行準備 :p )

## 環境測試
為確保環境自行準備無誤，此階段暫時將**TestLeaf**加入網路拓樸，並將配置如下：
![](../../../assets/cumulus/test_topology.png)
### 環境準備目的
Ansible預設依賴SSH溝通Leaf，此階段要確認**ManagamentStation**能否成功SSH到**TestLeaf**。
* 設定**DHCP Server**
* **DHCP Server**動態配置IP與ZTP到**TestLeaf**
* ZTP自動化過程
  * **TestLeaf**向**ManagementStation**的Web server請求ZTP Script 
  * **TestLeaf**本地執行ZTP Script
* ZTP Script執行過程
  * **TestLeaf**向**ManagementStation**的Web server請求SSH公鑰 
  * **TestLeaf**將SSH公鑰儲存在本地
* **ManagementStation**能以公鑰認證(無須密碼)SSH到**TestLeaf** 

### 環境測試步驟
如果此階段不成功，請自行補足省略的操作過程，否則無法進行操作Ansible。

#### 操作ManagementStation
* 設定Web server靜態IP  
```sudo cat /etc/sysconfig/network-scripts/ifcfg-Wired_connection_1```  
![](../../../assets/cumulus/ms_int0.png)
* 重啟網路介面  
```sudo /etc/init.d/network restart```
* 放置ZTP Script  
ZTP Script 必要包含```# CUMULUS-AUTOPROVISIONING```和```exit 0```  
```sudo cat /var/www/html/ztp-ansible.sh```  
![](../../../assets/cumulus/ms_ztp0.png)
* 重啟Web server  
```sudo systemctl restart httpd.service```   

#### 操作DHCP Server  
* 設定 */etc/network/interfaces*  
實體介面 (除了Loopback是邏輯介面)  
![](../../../assets/cumulus/dhcp_int0.png)  
SVI (Switch Virtual Interface)  
![](../../../assets/cumulus/dhcp_int1.png)
* 重啟網路介面  
```sudo ifreload -a```  
* 設定 */etc/dhcp/dhcpd.conf*  
包含ZTP option和ZTP Script URL  
![](../../../assets/cumulus/dhcp_conf0.png)  
主機名稱、對應網路介面所分配的IP  
![](../../../assets/cumulus/dhcp_conf1.png)  
* 設定 */etc/default/isc-dhcp-server*  
分配*subnet*的網路介面  
![](../../../assets/cumulus/dhcp_conf2.png)  
* 重啟DHCP server  
```sudo systemctl restart dhcpd.service```  
* 若分配IP失敗，請檢查  
```sudo grep dhcpd /var/log/syslog | less```   

#### 操作TestLeaf  
1. **TestLeaf**開機後，```eth0```應配得IP  
```sudo ip addr show | grep -A3 eth0```  
3. ZTP自動化完成後，得到SSH公鑰  
```sudo ls /root/.ssh/authorized_keys```  
3. 若要重新ZTP，務必刪掉所有ZTP相關檔案  
```sudo ztp --reset && sudo reboot```  

#### 操作ManagementStation  
* 設定**TestLeaf**的主機名稱與對應的IP  
```sudo cat /etc/hosts```  
![](../../../assets/cumulus/ms_etc_hosts.png)  
* 測試SSH到**TestLeaf**  
Cumulus VX 預設已啟動SSH server，並監聽22 port  
```sudo ssh root@test.cumulus```  

## Ansible目錄
以下操作**ManagementStation**： 
* 創建Ansible目錄  
```sudo git clone https://github.com/derailment/ansible-cumulus-ospf```  
```cd ansible-cumulus-ospf```  
```sudo tree```  
![](../../../assets/cumulus/ms_tree.png)  
* 安裝Cumulus Linux的Ansible add-on  
```sudo ansible-galaxy install cumulus.CumulusLinux```  
* ansible.cfg  
*inventory*： 放置主機名稱清單的位置  
*library*： 放置Cumulus Linux的Ansible add-on的位置  
![](../../../assets/cumulus/ms_ansible_cfg.png)  
* hosts 
Ansible所管理的主機名稱（必須在/etc/hosts或DNS server找得到對應的IP） 
  ![](../../../assets/cumulus/ms_hosts.png)  
* playbook.yml  
![](../../../assets/cumulus/ms_playbook.png)  
* playbook.yml為ospf角色指定以下行為：  
  * 如果roles/ospf/tasks/main.yml存在，就將其所列出的task加進play。 
  ![](../../../assets/cumulus/ms_tasks.png)  
  * 所有template task可以引用roles/ospf/templates/中的Jinja2檔案，不需指明檔案路徑。 
  ![](../../../assets/cumulus/ms_templates.png)  
  * 獲取遠端主機資訊  
  ```sudo ansible test.cumulus -m setup | grep ansible_nodename```  
  ![](../../../assets/cumulus/ms_setup.png)
  * 如果roles/ospf/vars/main.yml存在，就將其所列出的variable加進play。 
  ![](../../../assets/cumulus/ms_vars.png)  
  * 如果roles/ospf/handlers/main.yml存在，就將其所列出的handler加進play。 
  ![](../../../assets/cumulus/ms_handlers.png)  

## 目標測試  
拔掉**TestLeaf**，接上**Leaf1**、**Leaf2**、**bond0**，並將所有設備開機，配置如下：  
![](../../../assets/cumulus/objective_test_topology.png)

### 執行Ansible目錄的playbook.yml  
* 操作ManagementStation  
```cd ansible-cumulus-ospf```  
```sudo ansible-playbook playbook.yml```  
（看到*test.cumulus failed*的訊息是正常的，如果不喜歡就從hosts移除*test.cumulus*） 

### 檢查路由  
* 操作Leaf1  
檢查**Leaf1**是否成功學習到**Leaf2**的**bridge.200**的路由  
```sudo ip route show```  
![](../../../assets/cumulus/leaf1_route.png)  
* 操作Leaf2  
檢查**Leaf2**是否成功學習到**Leaf1**的**bridge.100**的路由  
```sudo ip route show```  
![](../../../assets/cumulus/leaf2_route.png)  

### *ping*測試  
* 操作Leaf1  
```ping 10.1.1.22```  
![](../../../assets/cumulus/leaf1_bridge200.png)  
* 操作Leaf2  
```ping 10.1.1.11```  
![](../../../assets/cumulus/leaf2_bridge100.png)  


