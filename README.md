# Raspberry Pi 1B 無線 Tor Tunnel 建置文件
Jan. 2026
### 這是 AI 生成的紀錄

我在製作過程中交替使用 Perplexity、Gemini、Claude 三個 AI 的多個對話，中間處理了很多坑跟 BUG ，最後再叫 AI 幫我整理出正確的操作紀錄。
我有大致看過，除了漏掉修正驅動之外沒啥大問題 (阿這就是最大的坑阿...)。
已補上驅動跟一些方便的前置作業。本來想再弄一次確認流程，但實在是有點無趣。

總之，如果網卡是用 RTL8188 系列的晶片，AI 會不知道要換驅動，其他的部分 AI 可以搞定。

![IMG_20260129_002928](https://github.com/user-attachments/assets/c25e456f-4356-47b5-ac47-278e1963a81c)

> **動機**
> 一直想搭建一個不貴的 Wifi 中介裝置，把一台手機或電腦的連線透過硬體轉換接到 TOR 或 VPN 上，確保不會有封包洩漏之類的風險。也可以當行動網路的安全設備，減少連上公共 Wifi 的安全風險。
> 類似的裝置也有 GL.iNet 的 Mango (GL-MT300N-v2)。只要台幣1000多，速度又較快，還是 CP 值很高的選擇。
> 
> 使用 RTL8188 系列晶片的網卡需要特別的驅動才能正常運作在AP模式，也是導致很多問題的根源，所以也比較便宜。
> 用其他種類的網卡應該會容易一些，有機會可以直接使用 DietPi，設定又更容易了。
> 
> **目標**
> 使用 Raspberry Pi 1B 建立無線 TOR 隧道路由器。
> 
> **硬體需求**

> - Raspberry Pi 1B
> - Usb 網卡*2
> 
> **系統環境**
> Raspberry Pi OS 32-Bit
> - 建議安裝桌面版方便作業，但設定開機至 CLI 避免效能太差
> - 使用 Raspberrp Connect 方便配合剪貼簿跟 AI 使用

---

## 📋 目錄

0. [燒錄 OS](#0-燒錄-os)
1. [系統準備](#1-系統準備)
2. [驅動編譯與安裝](#2-驅動編譯與安裝)
3. [固定網卡名稱](#3-固定網卡名稱)
4. [安裝必要軟體](#4-安裝必要軟體)
5. [配置網路服務](#5-配置網路服務)
6. [配置 TOR](#6-配置-tor)
7. [配置防火牆](#7-配置防火牆)
8. [自動化啟動](#8-自動化啟動)
9. [測試驗證](#9-測試驗證)
10. [故障排除](#10-故障排除)

---

## 0. 燒錄 OS
去 Raspberry Pi 官方網站下載 Imager，
順便申請 Raspberry Pi Connect 的帳戶跟金鑰，
之後可以用遠端命令行操作。

 - 燒錄的時候可以選有桌面的版本，比較方便操作。
 - 記得直接貼上金鑰，以免到時候要手打。
 - 可以直接建帳密可以開機自動登入。
 - 選擇開機到 CLI，效能才不會誇張差。有需要使用桌面環境的時候再用下面的指令進入桌面。
```
startx
```
 - 如果跟我一樣使用最原始(跟便宜)的 Raspberry Pi 1，開機會需要滿久的。

## 1. 系統準備

### 1.1 更新系統

```bash
sudo apt update
sudo apt upgrade -y
```

### 1.2 安裝編譯工具

```bash
sudo apt install -y git build-essential dkms linux-headers-$(uname -r)
```

### 1.3 確認網卡識別

```bash
lsusb
```

應該看到：
```
Bus 001 Device 006: ID 0bda:8179 Realtek Semiconductor Corp. RTL8188EUS
Bus 001 Device 007: ID 0bda:0179 Realtek Semiconductor Corp. RTL8188ETV
```

---

## 2. 驅動編譯與安裝

### 2.1 下載 lwfinger 驅動

版本非常重要。

```bash
cd ~
git clone -b v5.2.2.4 https://github.com/lwfinger/rtl8188eu.git
cd rtl8188eu
```

### 2.2 檢查 USB ID 支援

```bash
nano os_dep/linux/usb_intf.c
```

確認包含以下兩行（如果沒有則手動添加）：
```c
{USB_DEVICE(0x0BDA, 0x8179)}, /* Realtek RTL8188EUS */
{USB_DEVICE(0x0BDA, 0x0179)}, /* Realtek RTL8188ETV */
```

### 2.3 編譯並安裝

```bash
make all
sudo make install
```

### 2.4 禁用內建衝突驅動

```bash
sudo nano /etc/modprobe.d/blacklist-rtl.conf
```

添加內容：
```
# 禁用內建的不穩定驅動
blacklist r8188eu
blacklist rtl8xxxu

# 確保不會被載入
install r8188eu /bin/true
install rtl8xxxu /bin/true
```

### 2.5 配置新驅動參數

```bash
sudo nano /etc/modprobe.d/8188eu.conf
```

添加內容：
```
# 使用編譯的新驅動
options 8188eu rtw_power_mgnt=0 rtw_enusbss=0
```

### 2.6 徹底禁用舊驅動（可選但推薦）

```bash
sudo find /lib/modules/$(uname -r) -name "rtl8xxxu.ko*" -exec mv {} {}.disabled \;
sudo find /lib/modules/$(uname -r) -name "r8188eu.ko*" -exec mv {} {}.disabled \;
sudo depmod -a
```

### 2.7 重新啟動

```bash
sudo reboot
```

### 2.8 驗證驅動載入

重啟後執行：
```bash
lsmod | grep 8188
dmesg | grep -i "rtl8188eu"
```

應該看到：
```
8188eu                xxx  0
RTW: rtl8188eu v5.2.2.4_25483.20171222
```

---

## 3. 固定網卡名稱

### 3.1 查看網卡 MAC 地址

```bash
ip link show | grep -A 1 wlan
```

記下兩張網卡的 MAC 地址。

### 3.2 創建 udev 規則

```bash
sudo nano /etc/udev/rules.d/70-persistent-net.rules
```

添加內容（**替換成你的實際 MAC 地址**）：
```
# RTL8188EUS - 連接上游 WiFi
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="00:ac:05:02:61:51", NAME="wlan-sta"

# RTL8188ETV - 提供 AP 熱點
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="cc:79:cf:a6:62:58", NAME="wlan-ap"
```

### 3.3 重新載入規則

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo reboot
```

### 3.4 驗證網卡名稱

重啟後執行：
```bash
ip link show
```

應該看到 `wlan-sta` 和 `wlan-ap`。

---

## 4. 安裝必要軟體

### 4.1 安裝網路服務

```bash
sudo apt install -y hostapd dnsmasq tor iptables
```

### 4.2 停止服務（稍後手動啟動）

```bash
sudo systemctl stop hostapd dnsmasq tor
sudo systemctl disable hostapd dnsmasq tor
```

---

## 5. 配置網路服務

### 5.1 配置 wlan-sta (連接上游 WiFi)

創建 wpa_supplicant 配置：
```bash
sudo nano /etc/wpa_supplicant/wpa_supplicant-wlan-sta.conf
```

內容：
```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=TW

network={
    ssid="上游 WiFi名稱"
    psk="上游 WiFi密碼"
    key_mgmt=WPA-PSK
}
```

### 5.2 配置 wlan-ap (AP 熱點)

#### 5.2.1 配置 hostapd

```bash
sudo nano /etc/hostapd/hostapd.conf
```

內容：
```
interface=wlan-ap
driver=nl80211

ssid=TOR_Gateway
wpa_passphrase=yourpassword123

hw_mode=g
channel=6
ieee80211n=1
wmm_enabled=1
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

#### 5.2.2 指定 hostapd 配置檔

```bash
sudo nano /etc/default/hostapd
```

找到 `#DAEMON_CONF=""` 這行，改為：
```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

#### 5.2.3 取消遮蔽並啟用 hostapd

```bash
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
```

### 5.3 配置 dnsmasq (DHCP + DNS)

#### 5.3.1 備份原始配置

```bash
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.backup
```

#### 5.3.2 創建新配置

```bash
sudo nano /etc/dnsmasq.conf
```

內容：
```
interface=wlan-ap
bind-interfaces
dhcp-range=10.0.0.10,10.0.0.100,255.255.255.0,24h
dhcp-option=3,10.0.0.1
dhcp-option=6,10.0.0.1
server=127.0.0.1#9053
domain-needed
bogus-priv
no-resolv
log-queries
log-dhcp
```

#### 5.3.3 啟用 dnsmasq

```bash
sudo systemctl enable dnsmasq
```

---

## 6. 配置 TOR

### 6.1 編輯 TOR 配置

```bash
sudo nano /etc/tor/torrc
```

在檔案**最後**添加：
```
VirtualAddrNetworkIPv4 10.192.0.0/10
AutomapHostsOnResolve 1
TransPort 0.0.0.0:9040
DNSPort 0.0.0.0:9053
```

**重要說明**：
- 使用 `0.0.0.0` 而非 `10.0.0.1`，這樣 TOR 才能正確綁定
- DNS 端口使用 `9053`（與 dnsmasq 配置一致）

### 6.2 啟用 TOR

```bash
sudo systemctl enable tor@default
```

---

## 7. 配置防火牆

### 7.1 啟用 IP 轉發

```bash
sudo nano /etc/sysctl.conf
```

找到 `#net.ipv4.ip_forward=1`，取消註解：
```
net.ipv4.ip_forward=1
```

立即生效：
```bash
sudo sysctl -p
```

### 7.2 創建 iptables 規則腳本

```bash
sudo nano /usr/local/bin/tor-iptables.sh
```

內容：
```bash
#!/bin/bash

# 清除現有規則
iptables -F
iptables -t nat -F

# 允許本地流量
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# 允許已建立的連線
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 允許 wlan-sta 的流量
iptables -A INPUT -i wlan-sta -j ACCEPT
iptables -A OUTPUT -o wlan-sta -j ACCEPT

# 允許 wlan-ap 的 DHCP 和 DNS
iptables -A INPUT -i wlan-ap -p udp --dport 67 -j ACCEPT
iptables -A INPUT -i wlan-ap -p udp --dport 53 -j ACCEPT

# NAT: wlan-ap -> wlan-sta
iptables -t nat -A POSTROUTING -o wlan-sta -j MASQUERADE

# TOR 透明代理
iptables -t nat -A PREROUTING -i wlan-ap -p tcp --syn -j REDIRECT --to-ports 9040
iptables -t nat -A PREROUTING -i wlan-ap -p udp --dport 53 -j REDIRECT --to-ports 9053

# 允許轉發
iptables -A FORWARD -i wlan-ap -o wlan-sta -j ACCEPT
iptables -A FORWARD -i wlan-sta -o wlan-ap -m state --state ESTABLISHED,RELATED -j ACCEPT
```

賦予執行權限：
```bash
sudo chmod +x /usr/local/bin/tor-iptables.sh
```

---

## 8. 自動化啟動

### 8.1 創建啟動腳本

```bash
sudo nano /usr/local/bin/tor-gateway-start.sh
```

內容：
```bash
#!/bin/bash

# 等待系統完全啟動
sleep 10

# 停止可能衝突的服務
killall wpa_supplicant 2>/dev/null || true

# 清理並配置 wlan-ap
ip addr flush dev wlan-ap 2>/dev/null || true
ip link set wlan-ap down
sleep 2
ip link set wlan-ap up
ip addr add 10.0.0.1/24 dev wlan-ap
sleep 2

# 啟動 wlan-sta 連線
wpa_supplicant -B -i wlan-sta -c /etc/wpa_supplicant/wpa_supplicant-wlan-sta.conf
sleep 10

# 設定 iptables 規則
/usr/local/bin/tor-iptables.sh
sleep 2

# 啟動 dnsmasq
systemctl start dnsmasq
sleep 3

# 啟動 hostapd
systemctl start hostapd
sleep 3

# 啟動 TOR
systemctl start tor@default
sleep 3

# 記錄啟動完成
logger "TOR Gateway started successfully"
```

賦予執行權限：
```bash
sudo chmod +x /usr/local/bin/tor-gateway-start.sh
```

### 8.2 創建 systemd 服務

```bash
sudo nano /etc/systemd/system/tor-gateway.service
```

內容：
```ini
[Unit]
Description=TOR WiFi Gateway
After=network.target
Wants=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/tor-gateway-start.sh
RemainAfterExit=yes
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

### 8.3 啟用服務

```bash
sudo systemctl daemon-reload
sudo systemctl enable tor-gateway.service
```

### 8.4 測試啟動（不重啟系統）

```bash
sudo systemctl start tor-gateway.service
```

等待約 30 秒，檢查狀態：
```bash
sudo systemctl status tor-gateway.service
sudo systemctl status hostapd
sudo systemctl status dnsmasq
sudo systemctl status tor@default
```

---

## 9. 測試驗證

### 9.1 檢查服務狀態

```bash
# 檢查 wlan-sta 連線
iwconfig wlan-sta
ip addr show wlan-sta

# 檢查 wlan-ap AP 模式
iwconfig wlan-ap
ip addr show wlan-ap

# 檢查 TOR 端口
sudo netstat -tulnp | grep tor
```

### 9.2 手機連線測試

1. **掃描 WiFi**：應該能看到 `TOR_Gateway`
2. **連接**：使用密碼 `yourpassword123`
3. **取得 IP**：應該在 `10.0.0.10 ~ 10.0.0.100` 範圍內

### 9.3 驗證 TOR 功能

在手機瀏覽器訪問：

#### 測試 1：TOR 檢測
```
https://check.torproject.org
```
應該顯示：
```
Congratulations. This browser is configured to use Tor.
```

#### 測試 2：檢查 IP
```
https://icanhazip.com
```
應該顯示 TOR 出口節點的 IP（不是你的真實 IP）

#### 測試 3：一般網站
```
http://example.com
```
應該能正常訪問。

### 9.4 檢查日誌

```bash
# dnsmasq 日誌（查看 DHCP 分配）
sudo journalctl -u dnsmasq -f

# TOR 日誌
sudo journalctl -u tor@default -f

# hostapd 日誌
sudo journalctl -u hostapd -f

# 系統日誌
sudo journalctl -f | grep "TOR Gateway"
```

---

## 10. 故障排除

### 10.1 hostapd 無法啟動

**問題**：`sudo systemctl status hostapd` 顯示 failed

**檢查**：
```bash
sudo journalctl -u hostapd -n 50
```

**常見原因**：
1. wlan-ap 被其他程序占用
   ```bash
   sudo killall wpa_supplicant
   sudo systemctl restart hostapd
   ```

2. 頻道衝突
   ```bash
   sudo nano /etc/hostapd/hostapd.conf
   # 將 channel=6 改為 channel=1 或 channel=11
   ```

3. 驅動問題
   ```bash
   iw list | grep -A 10 "Supported interface modes"
   # 確認有 AP 模式
   ```

### 10.2 手機無法取得 IP

**問題**：手機卡在「正在取得 IP 位址」

**檢查**：
```bash
# 確認 wlan-ap 有 IP
ip addr show wlan-ap

# 檢查 dnsmasq 日誌
sudo journalctl -u dnsmasq -n 50

# 手動設定 IP（如果遺失）
sudo ip addr add 10.0.0.1/24 dev wlan-ap
sudo systemctl restart dnsmasq
```

### 10.3 連上 WiFi 但無法上網

**問題**：手機有 IP 但無法訪問網站

**檢查**：
```bash
# 1. 確認 wlan-sta 有上游連線
ping -c 3 -I wlan-sta 8.8.8.8

# 2. 檢查 iptables 規則
sudo iptables -t nat -L -n -v
sudo iptables -L FORWARD -n -v

# 3. 檢查 TOR 狀態
sudo systemctl status tor@default
sudo netstat -tulnp | grep -E "9040|9053"

# 4. 重新應用規則
sudo /usr/local/bin/tor-iptables.sh
```

### 10.4 TOR 無法啟動

**問題**：`sudo systemctl status tor@default` 顯示 failed

**檢查**：
```bash
sudo journalctl -u tor@default -n 50
```

**常見原因**：
1. 配置錯誤
   ```bash
   sudo nano /etc/tor/torrc
   # 確認使用 0.0.0.0 而非 10.0.0.1
   ```

2. 端口被占用
   ```bash
   sudo netstat -tulnp | grep -E "9040|9050|9053"
   ```

### 10.5 重啟後網卡順序對調

**問題**：重啟後 wlan-sta 和 wlan-ap 對調

**解決**：
```bash
# 檢查 udev 規則
cat /etc/udev/rules.d/70-persistent-net.rules

# 確認 MAC 地址正確
ip link show

# 重新載入規則
sudo udevadm control --reload-rules
sudo udevadm trigger
```

### 10.6 完整重啟腳本

如果遇到問題，使用此腳本完整重啟：

```bash
sudo nano /usr/local/bin/tor-gateway-restart.sh
```

內容：
```bash
#!/bin/bash

echo "停止所有服務..."
systemctl stop tor-gateway hostapd dnsmasq tor@default
killall wpa_supplicant 2>/dev/null || true
sleep 3

echo "清理網路介面..."
ip addr flush dev wlan-ap 2>/dev/null || true
ip link set wlan-ap down
sleep 2

echo "重新啟動..."
systemctl start tor-gateway

echo "等待服務啟動（30秒）..."
sleep 30

echo "檢查狀態..."
systemctl status tor-gateway --no-pager
systemctl status hostapd --no-pager
systemctl status dnsmasq --no-pager
systemctl status tor@default --no-pager

echo "網路介面："
ip addr show wlan-ap
iwconfig wlan-ap
```

賦予執行權限並使用：
```bash
sudo chmod +x /usr/local/bin/tor-gateway-restart.sh
sudo /usr/local/bin/tor-gateway-restart.sh
```

---

## 附錄 A：常用診斷指令

```bash
# 查看所有網路介面
ip link show

# 查看 WiFi 狀態
iwconfig

# 查看 USB 裝置
lsusb

# 查看已載入的驅動模組
lsmod | grep 8188

# 查看核心訊息
dmesg | tail -50

# 查看 TOR 電路
sudo -u debian-tor tor-circuit-info

# 測試 DNS 解析
dig @127.0.0.1 -p 9053 google.com

# 查看 iptables 規則
sudo iptables -L -n -v
sudo iptables -t nat -L -n -v

# 查看連線統計
sudo netstat -s

# 查看活躍連線
sudo netstat -tulnp
```

---

## 附錄 B：效能優化建議

### B.1 降低 TOR 日誌等級

```bash
sudo nano /etc/tor/torrc
```

添加：
```
Log notice file /var/log/tor/notices.log
```

### B.2 限制 DHCP 租約數量

```bash
sudo nano /etc/dnsmasq.conf
```

修改：
```
dhcp-range=10.0.0.10,10.0.0.30,255.255.255.0,24h
```

### B.3 降低 hostapd 日誌

```bash
sudo nano /etc/hostapd/hostapd.conf
```

添加：
```
logger_syslog=-1
logger_syslog_level=2
```

---

## 附錄 C：備份與恢復

### C.1 備份重要配置

```bash
# 創建備份目錄
mkdir -p ~/tor-gateway-backup

# 備份配置檔
sudo cp /etc/hostapd/hostapd.conf ~/tor-gateway-backup/
sudo cp /etc/dnsmasq.conf ~/tor-gateway-backup/
sudo cp /etc/tor/torrc ~/tor-gateway-backup/
sudo cp /etc/wpa_supplicant/wpa_supplicant-wlan-sta.conf ~/tor-gateway-backup/
sudo cp /etc/udev/rules.d/70-persistent-net.rules ~/tor-gateway-backup/
sudo cp /usr/local/bin/tor-*.sh ~/tor-gateway-backup/
sudo cp /etc/systemd/system/tor-gateway.service ~/tor-gateway-backup/

# 打包
tar -czf ~/tor-gateway-backup.tar.gz ~/tor-gateway-backup/
```

### C.2 完整 SD 卡映像備份

在 Linux/Mac 上：
```bash
# 插入 SD 卡，找到裝置名稱（例如 /dev/sdb）
lsblk

# 備份（替換 /dev/sdX 為實際裝置）
sudo dd if=/dev/sdX of=~/rpi-tor-gateway.img bs=4M status=progress

# 壓縮
gzip ~/rpi-tor-gateway.img
```

恢復：
```bash
gunzip ~/rpi-tor-gateway.img.gz
sudo dd if=~/rpi-tor-gateway.img of=/dev/sdX bs=4M status=progress
```

---

## 附錄 D：安全性建議

### D.1 更改預設密碼

```bash
# 更改 Pi 用戶密碼
passwd

# 更改 AP 密碼
sudo nano /etc/hostapd/hostapd.conf
# 修改 wpa_passphrase
sudo systemctl restart hostapd
```

### D.2 定期更新

```bash
sudo apt update
sudo apt upgrade -y
sudo reboot
```

### D.3 監控連線

```bash
# 查看連線的裝置
sudo iw dev wlan-ap station dump

# 查看 DHCP 租約
cat /var/lib/misc/dnsmasq.leases
```

---

## 總結

完成上述步驟後，你的 Raspberry Pi 1B 就成為了一個功能完整的 TOR WiFi 隧道路由器：

✅ **wlan-sta** 自動連接上游 WiFi  
✅ **wlan-ap** 提供受 TOR 保護的 AP 熱點  
✅ 所有流量透過 TOR 網路匿名化  
✅ 開機自動啟動  
✅ 穩定運行

**享受你的匿名網路環境！** 🎉

---

**文檔版本**：v1.0  
**最後更新**：2026-01-28  
**測試平台**：Raspberry Pi 1B + Raspberry Pi OS (Lite)
