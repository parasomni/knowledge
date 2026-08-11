# Debian installation

    sudo apt install wireguard-tools

# Generate Wireguard key on both server and client side

    umask 077                             
    wg genkey | tee private.key | wg pubkey > public.key
    cat private.key

# Server setup

Open Wireguard server config:

   sudo nano /etc/wireguard/wg0.conf

Write and save:

    [Interface]
    PrivateKey = SERVER_PRIVATE_KEY
    Address = 10.0.0.1/24
    ListenPort = 51820

    [Peer]
    PublicKey = CLIENT_PUBLIC_KEY
    AllowedIPs = 10.0.0.2/32

    [Peer]
    PublicKey = CLIENT_2_PUBLIC_KEY
    AllowedIPs = 10.0.0.3/32

Enable and start service:

    systemctl enable --now wg-quick@wg0

# Client setup 

Open wireguard client config:

    sudo nano /etc/wireguard/wg0.conf

Write and save:

    [Interface]
    PrivateKey = CLIENT_PRIVATE_KEY
    Address = 10.0.0.2/24
    DNS = 1.1.1.1

    [Peer]
    PublicKey = SERVER_PUBLIC_KEY
    Endpoint = SERVER_PUBLIC_IP:51820
    AllowedIPs = 10.0.0.0/24
    PersistentKeepalive = 25

Enable and start service:

    sudo systemctl enable --now wg-quick@wg0

# Cleanup

Shred the generated private and public key files:

    shred -uvz private.key public.key