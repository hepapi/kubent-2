tar -xzf kubectx_v0.11.0_linux_x86_64.tar.gz
chmod +x kubectx
sudo mv kubectx /usr/local/bin/

tar -xzf kubens_v0.11.0_linux_x86_64.tar.gz
chmod +x kubens
sudo mv kubens /usr/local/bin/

kubectx --version
kubens --version


cat <<'EOF' >> ~/.bashrc

# kubectx / kubens
alias kx='kubectx'
alias kn='kubens'
EOF

source ~/.bashrc
