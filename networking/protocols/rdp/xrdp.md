# XRDP Server setup

    sudo apt update
    sudo apt install -y xfce4 xfce4-goodies dbus-x11 xorgxrdp
    sudo apt install -y xrdp
    
    sudo systemctl enable xrdp
    sudo systemctl start xrdp

Add XRDP to ssl:

    sudo adduser xrdp ssl-cert
    sudo systemctl restart xrdp

Change xfce session:

    nano ~/.xsession
    startxfce4

    chmod +x ~/.xsession

Set backend:

    sudo nano /etc/xrdp/xrdp.ini


Add line:

    exec startxfce4
    use_vsock=false
    enable_drdynvc=true


Fix PolicyKit permissions:

    sudo nano /etc/polkit-1/localauthority/50-local.d/45-allow-colord.pkla

Add:

    [Allow colord for all users]
    Identity=unix-user:*
    Action=org.freedesktop.color-manager.create-device
    ResultAny=yes
    ResultInactive=yes
    ResultActive=yes

Restart:

    sudo systemctl restart xrdp
    sudo systemctl restart xrdp-sesman


Fix Xauthority permissions

    # Ensure your home is owned by you
    sudo chown -R "$USER":"$USER" "$HOME"

    # Clean potentially broken session auth files
    rm -f ~/.Xauthority ~/.xsession-errors

    # Ensure /tmp has correct sticky bit permissions
    sudo chmod 1777 /tmp


Open file:

    sudo nano /etc/xrdp/startwm.sh

Add or replace:

    #!/bin/sh

    # Make sure dbus is available for the session
    if command -v dbus-launch >/dev/null 2>&1; then
        eval "$(dbus-launch --sh-syntax)"
    fi

    # Start XFCE
    exec startxfce4

Then:

    sudo chmod +x /etc/xrdp/startwm.sh
    sudo systemctl restart xrdp xrdp-sesman

Cleanup:

    rm -f ~/.Xauthority ~/.xsession-errors ~/.ICEauthority
    mkdir -p ~/.cache

Ensure permissions are set:

    sudo chown -R srv-core:srv-core /home/srv-core
    sudo chmod 700 /home/srv-core


# Client side

Dynamic resizing:

    xfreerdp /v:VM_IP /u:srv-core /clipboard /dynamic-resolution /scale:100


Fix Size:

    xfreerdp /v:VM_IP /u:srv-core /clipboard /size:1920x1080
