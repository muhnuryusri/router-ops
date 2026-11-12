# router-ops

9router sebagai gateway AI bersama di VPS, untuk dipakai banyak project (Memora, dan project lain).

```
internet ──HTTPS──> Caddy (ai.example.com, hanya /v1/*) ──> 9router 127.0.0.1:20128
dashboard 9router ──> hanya lewat SSH tunnel
```

## Pasang (sekali)

Semua perintah di VPS sebagai root (`sudo -i`). Ganti `ai.example.com` dengan domain Anda, dan buat A record domain itu ke IP VPS **sebelum** Caddy dinyalakan.

```bash
# Node 22 sudah terpasang dari setup Memora. Kalau belum: curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && apt -y install nodejs
npm i -g 9router@0.5.104
adduser --system --group --home /var/lib/9router router
```

Dari laptop, kirim file:

```bash
scp 9router.service Caddyfile ubuntu@IP_VPS:/tmp/
```

Di VPS:

```bash
ln -sf "$(command -v 9router)" /usr/bin/9router
mv /tmp/9router.service /etc/systemd/system/
apt -y install caddy && mv /tmp/Caddyfile /etc/caddy/Caddyfile
sed -i 's/ai.example.com/DOMAIN_ANDA/' /etc/caddy/Caddyfile
ufw allow 80,443/tcp
systemctl daemon-reload && systemctl enable --now 9router && systemctl reload caddy
```

Cek: `journalctl -u 9router -n 30 --no-pager`, lalu `curl -s https://DOMAIN_ANDA/v1/models -H "Authorization: Bearer KEY"`.

## Buka dashboard (setup provider, buat key)

Dari laptop:

```bash
ssh -N -L 20129:127.0.0.1:20128 ubuntu@IP_VPS
```

Buka http://localhost:20129. Langkah pertama: **set password dashboard**, lalu hubungkan provider.

## Satu key per project per environment

Buat key terpisah, misalnya `memora-prod`, `memora-dev`, `projectX-prod`. Kalau satu project bocor, cabut key itu saja; yang lain tetap jalan.

Di project, isi tiga env (contoh Memora):

```
ORDER_REVIEW_9ROUTER_URL=https://DOMAIN_ANDA/v1/chat/completions
ORDER_REVIEW_9ROUTER_MODEL=<model id dari dashboard>
ORDER_REVIEW_9ROUTER_API_KEY=<key memora-prod>
```

Project baru cukup memakai URL yang sama dan key miliknya sendiri. Server tidak perlu diubah.

## Update

```bash
npm i -g 9router@<versi> && systemctl restart 9router
```

Pin versinya, jangan `latest`. Unit ini memakai `--skip-update` supaya 9router tidak memperbarui dirinya sendiri di production.

## Backup

Data (provider, key, riwayat) ada di `/var/lib/9router/.9router/db/data.sqlite`. Salin berkala, misalnya:

```bash
sqlite3 /var/lib/9router/.9router/db/data.sqlite ".backup /root/9router-$(date +%F).sqlite"
```

File ini berisi token provider. Simpan di tempat privat, jangan di repo.

## Catatan

- RAM 2 GB cukup untuk 9router dan worker Memora. Kalau worker yang memakai Chromium (order-review) ikut jalan, tambah swap 2 GB: `fallocate -l 2G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile && echo '/swapfile none swap sw 0 0' >> /etc/fstab`.
- Kalau provider memakai akun langganan personal, pastikan syarat layanannya mengizinkan pemakaian di production.
