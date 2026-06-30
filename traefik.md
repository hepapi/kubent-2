tar -xzf ingress-nginx-migration-v1.2.1-linux-amd64.tar.gz

sudo mv ingress-nginx-migration /usr/local/bin/

sudo chmod +x /usr/local/bin/ingress-nginx-migration

ingress-nginx-migration version

ingress-nginx-migration --kubeconfig ~/.kube/config

# tarayıcı yoksa 
ingress-nginx-migration --kubeconfig ~/.kube/config --format markdown --output-file rapor.md




- html raporunu indirme bunu kubent folderında çalıştıralım
```bash
ingress-nginx-migration --kubeconfig ~/.kube/config --addr 127.0.0.1:18080 &

sleep 3

curl -s http://127.0.0.1:18080/ > rapor.html

kill $SRV
```

- Dosyayı görme
```bash
grep -o 'const reportJSON = {.*};' rapor.html | sed 's/const reportJSON = //;s/;$//' | jq .
```

- Özet
```bash
grep -o 'const reportJSON = {.*};' rapor.html | sed 's/const reportJSON = //;s/;$//' \
  | jq -r '"Toplam: \(.ingressCount) | Uyumlu: \(.compatibleIngressCount) | Sorunlu: \(.unsupportedIngressCount)"'


# Hızlı özet (terminalde):
grep -o 'const reportJSON = {.*};' rapor.html | sed 's/const reportJSON = //;s/;$//' | jq .


grep -A1 'annotation-badge\|badge-v3' rapor.html \
  | grep -oE 'nginx\.ingress\.kubernetes\.io/[a-z-]+|v3\.[67]' \
  | paste -d ' ' - -


# unsupported:
grep -o 'const reportJSON = {.*};' rapor.html | sed 's/const reportJSON = //;s/;$//' \
  | jq '.unsupportedIngressAnnotations'
```
