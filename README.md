# Zakshin IP-VN

Danh sách CIDR IPv4/IPv6 của Việt Nam, dùng với `ipset` và `iptables` để **chỉ cho phép IP Việt Nam truy cập port 53 (DNS)** trên server Linux.

| File | Mô tả |
|------|--------|
| [`ipv4.txt`](ipv4.txt) | Danh sách CIDR IPv4 Việt Nam |
| [`ipv6.txt`](ipv6.txt) | Danh sách CIDR IPv6 Việt Nam |

> Nguồn dữ liệu: [IP2Location Free Visitor Blocker](https://www.ip2location.com/free/visitor-blocker) — nên cập nhật danh sách **mỗi tháng**.

---

## Tóm tắt rule

| Giao thức | Port | Nguồn thuộc `vn4` / `vn6` | Nguồn không thuộc VN |
|-----------|------|---------------------------|----------------------|
| IPv4 | 53 (TCP/UDP) | ACCEPT | DROP |
| IPv6 | 53 (TCP/UDP) | ACCEPT | DROP |

Các port khác (SSH, 80, 443, 853, …) **không bị ảnh hưởng**.

---

## 1. Cài công cụ cần thiết

```bash
sudo apt update
sudo apt install -y ipset iptables-persistent netfilter-persistent curl
```

---

## 2. Tạo script cập nhật list IP Việt Nam

```bash
sudo nano /usr/local/sbin/update-vn-ipset.sh
```

Dán nội dung sau:

```bash
#!/usr/bin/env bash
set -euo pipefail

IPV4_URL="https://raw.githubusercontent.com/ZakShinn/Zakshin_IP-VN/refs/heads/main/ipv4.txt"
IPV6_URL="https://raw.githubusercontent.com/ZakShinn/Zakshin_IP-VN/refs/heads/main/ipv6.txt"

TMP4="/tmp/vn_ipv4.txt"
TMP6="/tmp/vn_ipv6.txt"

echo "[1] Download IPv4 list..."
curl -fsSL "$IPV4_URL" -o "$TMP4"

echo "[2] Download IPv6 list..."
curl -fsSL "$IPV6_URL" -o "$TMP6"

echo "[3] Create temporary ipset..."
ipset create vn4_new hash:net family inet hashsize 8192 maxelem 300000 -exist
ipset flush vn4_new

ipset create vn6_new hash:net family inet6 hashsize 8192 maxelem 300000 -exist
ipset flush vn6_new

echo "[4] Import IPv4 CIDR..."
tr ',; \t' '\n' < "$TMP4" \
  | sed 's/\r//g' \
  | grep -E '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+/[0-9]+$' \
  | while read -r net; do
      ipset add vn4_new "$net" -exist
    done

echo "[5] Import IPv6 CIDR..."
tr ',; \t' '\n' < "$TMP6" \
  | sed 's/\r//g' \
  | grep -Ei '^[0-9a-f:]+/[0-9]+$' \
  | while read -r net; do
      ipset add vn6_new "$net" -exist
    done

echo "[6] Swap ipset..."
ipset create vn4 hash:net family inet hashsize 8192 maxelem 300000 -exist
ipset create vn6 hash:net family inet6 hashsize 8192 maxelem 300000 -exist

ipset swap vn4_new vn4
ipset swap vn6_new vn6

ipset destroy vn4_new
ipset destroy vn6_new

echo "[7] Save ipset..."
ipset save > /etc/ipset.conf

echo "Done."
echo "IPv4 entries:"
ipset list vn4 | grep "Number of entries"
echo "IPv6 entries:"
ipset list vn6 | grep "Number of entries"
```

Cấp quyền chạy và chạy lần đầu:

```bash
sudo chmod +x /usr/local/sbin/update-vn-ipset.sh
sudo /usr/local/sbin/update-vn-ipset.sh
```

Kiểm tra:

```bash
sudo ipset list vn4 | grep "Number of entries"
sudo ipset list vn6 | grep "Number of entries"
```

---

## 3. Backup trước khi sửa

```bash
sudo iptables-save | sudo tee ~/iptables-backup-before-port53-$(date +%F-%H%M%S).rules >/dev/null
sudo ip6tables-save | sudo tee ~/ip6tables-backup-before-port53-$(date +%F-%H%M%S).rules >/dev/null
```

---

## 4. Tạo rule IPv4: chỉ IP Việt Nam được vào port 53

```bash
sudo iptables -N VN_DNS53_ONLY 2>/dev/null || true
sudo iptables -F VN_DNS53_ONLY

sudo iptables -A VN_DNS53_ONLY -m set --match-set vn4 src -j ACCEPT
sudo iptables -A VN_DNS53_ONLY -j DROP

sudo iptables -C INPUT -p udp --dport 53 -j VN_DNS53_ONLY 2>/dev/null || sudo iptables -I INPUT 1 -p udp --dport 53 -j VN_DNS53_ONLY
sudo iptables -C INPUT -p tcp --dport 53 -j VN_DNS53_ONLY 2>/dev/null || sudo iptables -I INPUT 1 -p tcp --dport 53 -j VN_DNS53_ONLY
```

---

## 5. Tạo rule IPv6: chỉ IPv6 Việt Nam được vào port 53

```bash
sudo ip6tables -N VN6_DNS53_ONLY 2>/dev/null || true
sudo ip6tables -F VN6_DNS53_ONLY

sudo ip6tables -A VN6_DNS53_ONLY -m set --match-set vn6 src -j ACCEPT
sudo ip6tables -A VN6_DNS53_ONLY -j DROP

sudo ip6tables -C INPUT -p udp --dport 53 -j VN6_DNS53_ONLY 2>/dev/null || sudo ip6tables -I INPUT 1 -p udp --dport 53 -j VN6_DNS53_ONLY
sudo ip6tables -C INPUT -p tcp --dport 53 -j VN6_DNS53_ONLY 2>/dev/null || sudo ip6tables -I INPUT 1 -p tcp --dport 53 -j VN6_DNS53_ONLY
```

---

## 6. Kiểm tra rule đã vào đúng chưa

```bash
sudo iptables -L INPUT -n -v --line-numbers
sudo iptables -L VN_DNS53_ONLY -n -v --line-numbers

sudo ip6tables -L INPUT -n -v --line-numbers
sudo ip6tables -L VN6_DNS53_ONLY -n -v --line-numbers
```

Kết quả đúng sẽ có dạng:

```
Chain VN_DNS53_ONLY
ACCEPT  all  --  anywhere  anywhere  match-set vn4 src
DROP    all  --  anywhere  anywhere
```

Và IPv6:

```
Chain VN6_DNS53_ONLY
ACCEPT  all  --  anywhere  anywhere  match-set vn6 src
DROP    all  --  anywhere  anywhere
```

---

## 7. AdGuard Home chạy bằng Docker (tùy chọn)

Nếu AdGuard chạy qua Docker, thêm rule sau vì Docker đôi khi đi qua chain `DOCKER-USER`:

```bash
sudo iptables -C DOCKER-USER -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN 2>/dev/null || sudo iptables -I DOCKER-USER 1 -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN

sudo iptables -C DOCKER-USER -p udp --dport 53 -m set ! --match-set vn4 src -j DROP 2>/dev/null || sudo iptables -I DOCKER-USER 2 -p udp --dport 53 -m set ! --match-set vn4 src -j DROP
sudo iptables -C DOCKER-USER -p tcp --dport 53 -m set ! --match-set vn4 src -j DROP 2>/dev/null || sudo iptables -I DOCKER-USER 2 -p tcp --dport 53 -m set ! --match-set vn4 src -j DROP
```

> Nếu AdGuard chạy trực tiếp trên host thì **không cần** phần Docker này.

---

## 8. Lưu lại cấu hình

```bash
sudo ipset save | sudo tee /etc/ipset.conf >/dev/null
sudo netfilter-persistent save
```

---

## 9. Test xem có chặn thật không

Reset bộ đếm:

```bash
sudo iptables -Z VN_DNS53_ONLY
sudo ip6tables -Z VN6_DNS53_ONLY
```

Theo dõi:

```bash
watch -n 1 "sudo iptables -L VN_DNS53_ONLY -n -v; echo; sudo ip6tables -L VN6_DNS53_ONLY -n -v"
```

Nếu IP ngoài Việt Nam truy cập port 53, dòng `DROP` sẽ tăng packet/bytes.

Kiểm tra port 53 đang mở:

```bash
sudo ss -lntup | grep ':53'
sudo ss -lnuup | grep ':53'
```

---

## 10. Tự động cập nhật ipset bằng cron

Dùng **root crontab** để script có quyền cập nhật ipset.

### 10.1. Đảm bảo script chạy được

```bash
sudo chmod +x /usr/local/sbin/update-vn-ipset.sh
sudo /usr/local/sbin/update-vn-ipset.sh
```

Kiểm tra list:

```bash
sudo ipset list vn4 | grep "Number of entries"
sudo ipset list vn6 | grep "Number of entries"
```

### 10.2. Mở crontab của root

```bash
sudo crontab -e
```

Thêm 2 dòng sau vào cuối file:

```cron
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
0 3 1 * * /usr/bin/flock -n /run/update-vn-ipset.lock /usr/local/sbin/update-vn-ipset.sh >> /var/log/update-vn-ipset.log 2>&1
```

| Thành phần | Ý nghĩa |
|------------|---------|
| `0 3 1 * *` | Chạy lúc 03:00 ngày 01 hằng tháng |
| `flock` | Tránh chạy trùng nếu lần trước chưa xong |
| `>> /var/log/update-vn-ipset.log` | Ghi log cập nhật |

### 10.3. Kiểm tra cron đã lưu

```bash
sudo crontab -l
```

### 10.4. Bật dịch vụ cron

```bash
sudo systemctl enable --now cron
sudo systemctl status cron
```

### 10.5. Xem log

```bash
sudo tail -n 100 /var/log/update-vn-ipset.log
```

Hoặc theo dõi liên tục:

```bash
sudo tail -f /var/log/update-vn-ipset.log
```

### 10.6. Test cron ngay trong 1 phút (tùy chọn)

Thêm tạm dòng sau vào `sudo crontab -e`:

```cron
* * * * * /usr/bin/flock -n /run/update-vn-ipset.lock /usr/local/sbin/update-vn-ipset.sh >> /var/log/update-vn-ipset.log 2>&1
```

Đợi 1–2 phút rồi kiểm tra log. Test xong thì **xóa** dòng `* * * * *`, chỉ giữ lại dòng chạy mỗi tháng.

### 10.7. Kiểm tra nhanh sau mỗi lần cập nhật

```bash
sudo ipset list vn4 | grep "Number of entries"
sudo ipset list vn6 | grep "Number of entries"
sudo iptables -L VN_DNS53_ONLY -n -v --line-numbers
sudo ip6tables -L VN6_DNS53_ONLY -n -v --line-numbers
```

Mỗi tháng server sẽ tự tải lại `ipv4.txt` và `ipv6.txt`, cập nhật ipset `vn4`/`vn6`; rule port 53 **giữ nguyên**.

---

<br>

---

# English

Vietnam IPv4/IPv6 CIDR lists for use with `ipset` and `iptables` to **allow only Vietnamese IPs to access port 53 (DNS)** on a Linux server.

| File | Description |
|------|-------------|
| [`ipv4.txt`](ipv4.txt) | Vietnam IPv4 CIDR list |
| [`ipv6.txt`](ipv6.txt) | Vietnam IPv6 CIDR list |

> Data source: [IP2Location Free Visitor Blocker](https://www.ip2location.com/free/visitor-blocker) — update the lists **monthly**.

---

## Summary

| Protocol | Port | Source in `vn4` / `vn6` | Non-VN source |
|----------|------|---------------------------|---------------|
| IPv4 | 53 (TCP/UDP) | ACCEPT | DROP |
| IPv6 | 53 (TCP/UDP) | ACCEPT | DROP |

Other ports (SSH, 80, 443, 853, …) are **not affected**.

---

## 1. Install required tools

```bash
sudo apt update
sudo apt install -y ipset iptables-persistent netfilter-persistent curl
```

---

## 2. Create the Vietnam IP update script

```bash
sudo nano /usr/local/sbin/update-vn-ipset.sh
```

Paste the following:

```bash
#!/usr/bin/env bash
set -euo pipefail

IPV4_URL="https://raw.githubusercontent.com/ZakShinn/Zakshin_IP-VN/refs/heads/main/ipv4.txt"
IPV6_URL="https://raw.githubusercontent.com/ZakShinn/Zakshin_IP-VN/refs/heads/main/ipv6.txt"

TMP4="/tmp/vn_ipv4.txt"
TMP6="/tmp/vn_ipv6.txt"

echo "[1] Download IPv4 list..."
curl -fsSL "$IPV4_URL" -o "$TMP4"

echo "[2] Download IPv6 list..."
curl -fsSL "$IPV6_URL" -o "$TMP6"

echo "[3] Create temporary ipset..."
ipset create vn4_new hash:net family inet hashsize 8192 maxelem 300000 -exist
ipset flush vn4_new

ipset create vn6_new hash:net family inet6 hashsize 8192 maxelem 300000 -exist
ipset flush vn6_new

echo "[4] Import IPv4 CIDR..."
tr ',; \t' '\n' < "$TMP4" \
  | sed 's/\r//g' \
  | grep -E '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+/[0-9]+$' \
  | while read -r net; do
      ipset add vn4_new "$net" -exist
    done

echo "[5] Import IPv6 CIDR..."
tr ',; \t' '\n' < "$TMP6" \
  | sed 's/\r//g' \
  | grep -Ei '^[0-9a-f:]+/[0-9]+$' \
  | while read -r net; do
      ipset add vn6_new "$net" -exist
    done

echo "[6] Swap ipset..."
ipset create vn4 hash:net family inet hashsize 8192 maxelem 300000 -exist
ipset create vn6 hash:net family inet6 hashsize 8192 maxelem 300000 -exist

ipset swap vn4_new vn4
ipset swap vn6_new vn6

ipset destroy vn4_new
ipset destroy vn6_new

echo "[7] Save ipset..."
ipset save > /etc/ipset.conf

echo "Done."
echo "IPv4 entries:"
ipset list vn4 | grep "Number of entries"
echo "IPv6 entries:"
ipset list vn6 | grep "Number of entries"
```

Make it executable and run once:

```bash
sudo chmod +x /usr/local/sbin/update-vn-ipset.sh
sudo /usr/local/sbin/update-vn-ipset.sh
```

Verify:

```bash
sudo ipset list vn4 | grep "Number of entries"
sudo ipset list vn6 | grep "Number of entries"
```

---

## 3. Backup before making changes

```bash
sudo iptables-save | sudo tee ~/iptables-backup-before-port53-$(date +%F-%H%M%S).rules >/dev/null
sudo ip6tables-save | sudo tee ~/ip6tables-backup-before-port53-$(date +%F-%H%M%S).rules >/dev/null
```

---

## 4. IPv4 rules: allow only Vietnam IPs on port 53

```bash
sudo iptables -N VN_DNS53_ONLY 2>/dev/null || true
sudo iptables -F VN_DNS53_ONLY

sudo iptables -A VN_DNS53_ONLY -m set --match-set vn4 src -j ACCEPT
sudo iptables -A VN_DNS53_ONLY -j DROP

sudo iptables -C INPUT -p udp --dport 53 -j VN_DNS53_ONLY 2>/dev/null || sudo iptables -I INPUT 1 -p udp --dport 53 -j VN_DNS53_ONLY
sudo iptables -C INPUT -p tcp --dport 53 -j VN_DNS53_ONLY 2>/dev/null || sudo iptables -I INPUT 1 -p tcp --dport 53 -j VN_DNS53_ONLY
```

---

## 5. IPv6 rules: allow only Vietnam IPv6 on port 53

```bash
sudo ip6tables -N VN6_DNS53_ONLY 2>/dev/null || true
sudo ip6tables -F VN6_DNS53_ONLY

sudo ip6tables -A VN6_DNS53_ONLY -m set --match-set vn6 src -j ACCEPT
sudo ip6tables -A VN6_DNS53_ONLY -j DROP

sudo ip6tables -C INPUT -p udp --dport 53 -j VN6_DNS53_ONLY 2>/dev/null || sudo ip6tables -I INPUT 1 -p udp --dport 53 -j VN6_DNS53_ONLY
sudo ip6tables -C INPUT -p tcp --dport 53 -j VN6_DNS53_ONLY 2>/dev/null || sudo ip6tables -I INPUT 1 -p tcp --dport 53 -j VN6_DNS53_ONLY
```

---

## 6. Verify rules

```bash
sudo iptables -L INPUT -n -v --line-numbers
sudo iptables -L VN_DNS53_ONLY -n -v --line-numbers

sudo ip6tables -L INPUT -n -v --line-numbers
sudo ip6tables -L VN6_DNS53_ONLY -n -v --line-numbers
```

Expected output:

```
Chain VN_DNS53_ONLY
ACCEPT  all  --  anywhere  anywhere  match-set vn4 src
DROP    all  --  anywhere  anywhere
```

IPv6:

```
Chain VN6_DNS53_ONLY
ACCEPT  all  --  anywhere  anywhere  match-set vn6 src
DROP    all  --  anywhere  anywhere
```

---

## 7. AdGuard Home via Docker (optional)

If AdGuard runs in Docker, add these rules because Docker traffic may pass through the `DOCKER-USER` chain:

```bash
sudo iptables -C DOCKER-USER -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN 2>/dev/null || sudo iptables -I DOCKER-USER 1 -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN

sudo iptables -C DOCKER-USER -p udp --dport 53 -m set ! --match-set vn4 src -j DROP 2>/dev/null || sudo iptables -I DOCKER-USER 2 -p udp --dport 53 -m set ! --match-set vn4 src -j DROP
sudo iptables -C DOCKER-USER -p tcp --dport 53 -m set ! --match-set vn4 src -j DROP 2>/dev/null || sudo iptables -I DOCKER-USER 2 -p tcp --dport 53 -m set ! --match-set vn4 src -j DROP
```

> Skip this section if AdGuard runs directly on the host.

---

## 8. Persist configuration

```bash
sudo ipset save | sudo tee /etc/ipset.conf >/dev/null
sudo netfilter-persistent save
```

---

## 9. Test blocking

Reset counters:

```bash
sudo iptables -Z VN_DNS53_ONLY
sudo ip6tables -Z VN6_DNS53_ONLY
```

Monitor:

```bash
watch -n 1 "sudo iptables -L VN_DNS53_ONLY -n -v; echo; sudo ip6tables -L VN6_DNS53_ONLY -n -v"
```

Non-Vietnam IPs hitting port 53 will increment the `DROP` packet/byte counters.

Check listening port 53:

```bash
sudo ss -lntup | grep ':53'
sudo ss -lnuup | grep ':53'
```

---

## 10. Automatic monthly ipset updates via cron

Use the **root crontab** so the script has permission to update ipset.

### 10.1. Ensure the script works

```bash
sudo chmod +x /usr/local/sbin/update-vn-ipset.sh
sudo /usr/local/sbin/update-vn-ipset.sh
```

Verify lists:

```bash
sudo ipset list vn4 | grep "Number of entries"
sudo ipset list vn6 | grep "Number of entries"
```

### 10.2. Edit root crontab

```bash
sudo crontab -e
```

Add these two lines at the end:

```cron
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
0 3 1 * * /usr/bin/flock -n /run/update-vn-ipset.lock /usr/local/sbin/update-vn-ipset.sh >> /var/log/update-vn-ipset.log 2>&1
```

| Component | Meaning |
|-----------|---------|
| `0 3 1 * *` | Run at 03:00 on the 1st of every month |
| `flock` | Prevent overlapping runs |
| `>> /var/log/update-vn-ipset.log` | Append update logs |

### 10.3. Verify cron entry

```bash
sudo crontab -l
```

### 10.4. Enable cron service

```bash
sudo systemctl enable --now cron
sudo systemctl status cron
```

### 10.5. View logs

```bash
sudo tail -n 100 /var/log/update-vn-ipset.log
```

Or follow live:

```bash
sudo tail -f /var/log/update-vn-ipset.log
```

### 10.6. Quick cron test (optional)

Temporarily add to `sudo crontab -e`:

```cron
* * * * * /usr/bin/flock -n /run/update-vn-ipset.lock /usr/local/sbin/update-vn-ipset.sh >> /var/log/update-vn-ipset.log 2>&1
```

Wait 1–2 minutes and check the log. Remove the `* * * * *` line afterward; keep only the monthly schedule.

### 10.7. Quick check after each update

```bash
sudo ipset list vn4 | grep "Number of entries"
sudo ipset list vn6 | grep "Number of entries"
sudo iptables -L VN_DNS53_ONLY -n -v --line-numbers
sudo ip6tables -L VN6_DNS53_ONLY -n -v --line-numbers
```

Each month the server re-downloads `ipv4.txt` and `ipv6.txt`, refreshes ipset `vn4`/`vn6`, while port 53 rules **remain unchanged**.
