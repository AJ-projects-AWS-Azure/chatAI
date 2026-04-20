
# Add OpenTofu repo
sudo apt-get update
sudo apt-get install -y gnupg software-properties-common curl

curl -fsSL https://packages.opentofu.org/opentofu/tofu/gpgkey | \
  gpg --dearmor -o /usr/share/keyrings/opentofu.gpg

echo "deb [signed-by=/usr/share/keyrings/opentofu.gpg] https://packages.opentofu.org/opentofu/tofu/ubuntu $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/opentofu.list

# Install tofu
sudo apt-get update
sudo apt-get install -y tofu