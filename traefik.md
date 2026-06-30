tar -xzf ingress-nginx-migration-v1.2.1-linux-amd64.tar.gz

sudo mv ingress-nginx-migration /usr/local/bin/

sudo chmod +x /usr/local/bin/ingress-nginx-migration

ingress-nginx-migration version

ingress-nginx-migration --kubeconfig ~/.kube/config

# tarayıcı yoksa 
ingress-nginx-migration --kubeconfig ~/.kube/config --format markdown --output-file rapor.md
