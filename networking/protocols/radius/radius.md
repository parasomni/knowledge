# FreeRADIUS Konfiguration Guide

## Inhalt

- [1. Grundlegende Struktur des FreeRADIUS](#1-grundlegende-struktur-des-freeradius)
- [2. Authentifizierungsmethoden](#2-authentifizierungsmethoden)
  - [2.1 EAP-TLS](#21-eap-tls)
    - [2.1.1 Default Server](#211-default-server)
    - [2.1.2 EAP Modul](#212-eap-modul)
    - [2.1.3 Authenticator Konfiguration](#213-authenticator-konfiguration)
    - [2.1.4 Supplicant Konfiguration](#214-supplicant-konfiguration)
  - [2.2 EAP-TTLS-PAP](#22-eap-ttls-pap-authentifizierung-via-files)
  - [2.3 EAP-PEAP-MSCHAPv2](#23-eap-peap-mschapv2-authentifizierung-via-active-directory)
  - [2.4 MAB](#24-mab)
- [3. Sonstige FreeRADIUS Konfigurationen](#3-sonstige-konfigurationen)
  - [3.1 Zertifikatserstellung](#31-zertifikatserstellung)
  - [3.2 VLAN Policy für AD](#32-vlan-policy-für-ad)
  - [3.3 Nutzer Konfiguration](#33-nutzer-konfiguration)
    - [3.3.1 VLAN Zuweisung](#331-vlan-zuweisung)
    - [3.3.2 Firewalling Aruba AOS-CX](#332-firewalling-aruba-aos-cx)
    - [3.3.3 Firewalling FS PICOS](#333-firewalling-fs-picos)
  - [3.4 Logging Authentifizierungsversuche](#34-logging-authentifizierungsversuche)
- [4. Switch Konfiguration](#4-switch-konfiguration)
  - [4.1 Aruba AOS-CX](#41-aruba-aos-cx)
  - [4.2 FS PICOS](#42-fs-picos)
- [5. Troubleshooting](#5-troubleshooting)

## 1. Grundlegende Struktur des FreeRADIUS

Um den FreeRADIUS Server richtig zu konfigurieren, ist es wichtig den grundlegenden Aufbau
der Software zu verstehen. Deshalb werden zuerst die einzelnen Konfigurationsdateien sowie
Ordner benannt und ihre Funktion beschrieben.

> **Hinweis:**
>
> Die meisten der folgenden Bezeichnungen können in der `radiusd.conf` beliebig angepasst werden.
> Die Konfigurationen beschrieben in diesem Dokument verwenden die default Bezeichnungen der Ordner und Dateien.

| Datei / Ordner                     | Beschreibung                                                                                                                                                                                                                 |
|-----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| radiusd.conf                      | Hauptkonfigurationsdatei zum Einstellen und Anpassen des RADIUS Servers.                                                                                                                                                      |
| clients.conf                      | In dieser Datei werden Clients, die mit dem RADIUS Server kommunizieren eingerichtet. Das sind Switche, die RADIUS Anfragen von Supplicants weiterleiten.                                                                  |
| dictionary                        | Für das Einrichten von Downloadable User Roles (DUR) um Firewallregeln dynamisch durch den RADIUS Server anzuwenden, sind Herstellerspezifische Attribute zu hinterlegen. Diese werden in der `dictionary` Datei definiert, um später verwendet werden zu können. |
| proxy.conf                        | Zuständig für das Proxying von RADIUS-Anfragen. Dadurch können Anfragen an andere RADIUS-Server weitergeleitet werden.                                                                                                     |
| sites-available                   | In diesem Ordner sind virtuelle Server gespeichert. Sie können über einen Symlink in dem Ordner `sites-enabled` für die aktuelle Konfiguration eingestellt werden.                                                          |
| sites-enabled/default             | Diese Datei repräsentiert den grundsätzlichen FreeRADIUS Server, welcher Authentifizierungs und Accounting Anfragen entgegennimmt. Er ist ebenfalls zuständig für die Methode EAP-TLS und für den Aufbau eines TLS-Tunnels mit dem Supplicant, welcher von dem `inner-tunnel` verwendet wird, um Authentifizierungsdaten auszutauschen. |
| sites-enabled/inner-tunnel       | Definiert einen virtuellen Server, welcher Authentifizierungsanfragen des `default` Servers entgegennimmt. Dabei initiiert der `default` Server einen TLS-Tunnel mit dem Supplicant, in dem der `inner-tunnel` über die vordedefinierte Authentifizierungsmethode die Benutzerdaten austauscht. |
| mods-available und mods-enabled   | Ähnlich wie bei `sites-available` werden in `mods-available` anstatt virtuelle Server, FreeRADIUS Module gespeichert. Diese können dann ebenfalls mit einem Symlink in `mods-enabled` aktiviert werden.                      |
| mods-available/files              | Das Files Modul ist für den Authentifizierungsprozess über am RADIUS verwaltete lokale Dateien. Dabei wird kein klassisches Backend-System (LDAP, SQL) verwendet, sondern eine statische Datei für die Konfiguration von Nutzern. Diese Datei ist gespeichert in `mods-config/files/authorize`. Sie wird bei jedem Start bzw. Neustart des FreeRADIUS neu eingelesen und verarbeitet. Sollte das `files` Modul nicht verwendet werden, müssen die Nutzerinformationen direkt in dem `inner-tunnel` in Unlang definiert werden. |
| mods-config                       | Enthält Modul-spezifische Konfigurationsdateien, die von `mods-enabled` bzw. `mods-available` verwendet werden.                                                                                                                |
| hints                             | Konfiguration für PPP und SLIP.                                                                                                                                                                                             |
| huntgroups                        | Konfiguration von Gruppen von Netzwerkzugangsservern (NAS).                                                                                                                                                                  |
| certs                             | Speicherort für TLS-Zertifikate.                                                                                                                                                                                             |
| policy.d                          | Speicherort für Policy-Skripte.                                                                                                                                                                                             |

Generell durchläuft der FreeRADIUS folgende Prozesse:

1. Authorize
2. Authenticate
3. Post-Auth
4. Accounting
5. Session
6. Post-Proxy

> ***Hinweise:***
>
> Die Konfigurationen in den Dateien werden von FreeRADIUS sequenziell von oben nach unten eingelesen und abgearbeitet.
>
> Standardmäßig sind bereits die meisten Module über Symlinks aktiviert.
>
> Bei EAP-TTLS-PAP und EAP-PEAP-MSCHAPv2 wird EAP-TLS automatisch mit konfiguriert weshalb eine vollständige EAP-TLS Konfiguration vorraussetzend ist.

## 2. Authentifizierungsmethoden

- [2.1 EAP-TLS](#21-eap-tls)
  - [2.1.1 Default Server](#211-default-server)
  - [2.1.2 EAP Modul](#212-eap-modul)
  - [2.1.3 Authenticator Konfiguration](#213-authenticator-konfiguration)
  - [2.1.4 Supplicant Konfiguration](#214-supplicant-konfiguration)
- [2.2 EAP-TTLS-PAP](#22-eap-ttls-pap-authentifizierung-via-files)
- [2.3 EAP-PEAP-MSCHAPv2](#23-eap-peap-mschapv2-authentifizierung-via-active-directory)
- [2.4 MAB](#24-mab)

## 2.1 EAP-TLS

Für die Konfiguration der EAP-TLS Authentifizierungsmethode werden folgende Dateien bearbeitet:

- sites-enabled/default
- mods-enabled/eap

Bevor der `default` Server konfiguriert wird, sollten die nötigen Zertifikate bereits erstellt worden
sein und in dem Ordner `certs` abgelegt werden. Siehe [3.1 Zertifikatserstellung](#31-zertifikatserstellung).

### 2.1.1 Default Server

Der `default` Server definiert den grundlegenden FreeRADIUS Server für das verarbeiten von Anfragen.
Im folgenden wird beschrieben wie der FreeRADIUS für `authorisierung` und `authentifizierung` konfiguriert wird.

Basis Konfiguration:

    server default {
        listen {
            type = auth
            ipaddr = *
            port = 1812
        }
    }

Das `type` Attribut (auth/acct) beschreibt dabei, um welche Servervariante es sich handelt.

Als nächstes folgt die `authorize` Sektion. Diese sammelt alle relevanten Informationen
über den Benutzer aus dem eingestellten Backend-System (LDAP, SQL, FILES) als Vorbereitung
für den eigentlichen Authentifizierungsprozess. Sie beschreibt ebenfalls, welche Authentifizierungsmethode
und welches vorgesehene Modul dafür verwendet werden soll. In der `authenticate` Sektion wird eingestellt,
welches Modul für den Authentifizierungsprozess zuständig ist und ausgeführt wird.

`authorize` und `authenticate` Sektion:

    authorize{
        eap
    }

    authenticate{
        eap
    }

In beiden Fällen ist das `eap` Modul essenziell. Es stellt die nötigen Attribute für den Authentifizierungsprozess bereit
und führt diesen anschließend durch. Grundsätzlich reicht diese Konfiguration aus für `EAP-TLS`.

Um jedoch das Client Zertifikat anhand seines CN zu prüfen und Rollen zu zuweisen, ist ein Authentifizierungsbackend wichtig.
Dafür kann das `files` Modul und die Datei `mods-config/files/authorize` verwendet werden. Siehe [3.3 Nutzer Konfiguration](#33-nutzer-konfiguration).

Einstellung für das `files` Modul in der `authorize` Sektion:

    authorize{
        files
        eap
    }

Dieses `muss` vor dem `eap` Modul eingefügt werden, da es Attribute aus der `mods-config/files/authorize` liest, die danach von dem `eap` Modul
verarbeitet werden. Die `authenticate` Sektion bleibt dabei unverändert.

Vollständiger `default` Server für `EAP-TLS` mit `files` Module:

    server default {
        listen {
            type = auth
            ipaddr = *
            port = 1812
        }

        authorize{
            files
            eap
        }

        authenticate{
            eap
        }
    }

### 2.1.2 EAP Modul

In der `eap` Modul Konfiguration werden die TLS Einstellungen, sowie die verwendete Authentifizierungsmethode festgelegt.

Basis Konfiguration:

    eap {
        default_eap_type = tls

        tls-config tls-common {
            private_key_file = ${certdir}/server.key
            certificate_file = ${certdir}/server.pem
            ca_file = ${certdir}/ca.pem
            fragment_size = 1024
            include_length = yes
        }

        tls {
            tls = tls-common
        }
    }

Der `default_eap_type` beschreibt die standard Authentifizierungsmethode, jedoch werden
trotzdem alle anderen Methoden abgearbeitet, falls `tls` nicht funktioniert.
Deshalb kann diese Einstellung bei allen Methoden bei `tls` bleiben.

`tls-config` definiert eine TLS Einstellung `tls-common`, welche die Authentifizierung für
`EAP-TLS` bereitstellt und gleichzeitig den TLS-Tunnel für die Methoden `EAP-TTLS-PAP` und
`EAP-PEAP-MSCHAPv2` aufbaut.

Für `EAP-TLS` wird die Sektion `tls` eingerichtet und angegeben, welche `tls-config` verwendet werden soll.

### 2.1.3 Authenticator Konfiguration

Standardmäßig werden Anfragen von nicht authorisierten Authenticators abgelehnt. Deshalb `muss` dieser in der `clients.conf` eingetragen werden.

Dies wird wie folgt eingestellt:

    client aruba {
        ipaddr = 10.0.10.1
        secret = testing123
    }

Das `secret` dient zur gegenseiteigen Authentifizierung des `Authenticators` und des `FreeRADIUS`, weshalb dieses auf beiden Seiten gleich konfiguriert werden `muss`. Der `client` Name, in diesem Beispiel `aruba`,
kann frei gewählt werden.

### 2.1.4 Supplicant Konfiguration

Um einen Supplicant für `EAP-TLS` zu konfigurieren, muss das CA und Client Zertifikat auf dem System vorhanden sein.
Für die Varianten `EAP-TTLS-PAP` und `EAP-PEAP-MSCHAPv2` muss jeweils nur das CA Zertifikat am Supplicant hinterlegt werden.

Eine kabelgebundene `EAP-TLS` Beispiel Konfiguration für den `wpa_supplicant` unter linux könnte wie folgt aussehen:

    ctrl_interface=/var/run/wpa_supplicant
    ap_scan=0
    network={
        key_mgmt=IEEE8021X
        eap=TLS
        identity="radiustest"
        anonymous_identity="anonymous@domain"
        ca_cert="/etc/wpa_supplicant/ca.pem"
        client_cert="/etc/wpa_supplicant/radiustest.pem"
        private_key="/etc/wpa_supplicant/radiustest.key"
        eapol_flags=0
    }

Kabelgebundener `wpa_supplicant` für `EAP-PEAP-MSCHAPv2`:

    ctrl_interface=/var/run/wpa_supplicant
    ap_scan=0
    network={
        key_mgmt=IEEE8021X
        eap=PEAP
        identity="radiustest"
        anonymous_identity="anonymous@domain"
        password="#radius1!"
        ca_cert="/etc/wpa_supplicant/ca.pem"
        phase2="auth=MSCHAPV2"
        eapol_flags=0
    }

Kabelgebundener `wpa_supplicant` für `EAP-TTLS-PAP`:

    ctrl_interface=/var/run/wpa_supplicant
    ap_scan=0
    network={
        key_mgmt=IEEE8021X
        eap=TTLS
        identity="radiustest"
        anonymous_identity="anonymous@domain"
        password="#radius1!"
        ca_cert="/etc/wpa_supplicant/ca.pem"
        phase2="auth=PAP"
        eapol_flags=0
    }

Die Verwendung einer `anonymous_identity` dient dazu, den echten Benutzernamen im unverschlüsselten äußeren EAP‑Handshake zu verbergen und so die Privatsphäre zu schützen.
Sie ist eigentlich dafür gedacht, im äußeren EAP‑Handshake nur eine neutrale Kennung (z. B. anonymous@domain) zu übertragen, damit der RADIUS Server die Anfrage korrekt routen kann, ohne den echten Benutzernamen preiszugeben.

## 2.2 EAP-TTLS-PAP (Authentifizierung via FILES)

Für die Konfiguration der EAP-TTLS-PAP Authentifizierungsmethode über das `files` Modul, werden folgende Dateien
bearbeitet:

- sites-enabled/default
- sites-enabled/inner-tunnel
- mods-enabled/eap

Für die genaue Erläuterung des `default` Servers siehe [2.1.1 Default Servers](#211-default-server) und
für die Erläuterung der `MAB` Konfiguration siehe [2.4 Konfiguration für MAB](#24-mab).

Eine vollständiger `default` Server mit `MAB` integriert sieht wie folgt aus:

    server default {
        listen{
            type = auth
            ipaddr = *
            port = 1812
        }

        authorize{
            if (!EAP-Message) {
                update request {
                    User-Name := "%{User-Name}"
                }
            }
            files
            eap
        }

        authenticate {
            eap
        }
    }

Das `eap` Modul verwendet dieselbe Konfiguration wie aus [EAP-TLS](#212-eap-modul) mit dem folgenden Zusatz für `ttls`:

    eap {
        default_eap_type = tls

        tls-config tls-common{
            ...
        }

        tls {
            ...
        }

        ttls {
            tls = tls-common
            default_eap_type = pap
            virtual_server = "inner-tunnel"
        }
    }

Die Angabe eines `virtual_server` ist essenziell. Dieser ist zuständig für den Austausch von Authentifizierungsdaten und deren Überprüfung.

Eine `inner-tunnel` Konfiguration sieht wie folgt aus:

    server inner-tunnel{
        authorize{
            files
            pap
        }

        authenticate{
            Auth-Type PAP {
                pap
            }
        }
    }

Bei Verwendung des `files` Moduls und `EAP-TTLS-PAP` ist es gängig, den `Auth-Type` manuell auf `PAP` einzustellen, da das `files` Modul diesen unzuverlässig
setzt.
Des weiteren muss `files` im `inner-tunnel` gesetzt werden (für EAP-TTLS-PAP) und im `default` (für EAP-TLS).
Eine Konfiguration von `mods-enabled/files` und `mods-enabled/pap` ist generell nicht notwendig.

Nun kann der FreeRADIUS Server gestartet werden:

    systemctl enable freeradius
    systemctl start freeradius

Oder für Debug Zwecke:

    freeradius -X

## 2.3 EAP-PEAP-MSCHAPv2 (Authentifizierung via Active Directory)

> ***Hinweis:***
>
> In Umgebungen mit AD sollte immer zuerst die Zeit synchronisiert werden!

Für die Konfiguration der EAP-PEAP-MSCHAPv2 Authentifizierungsmethode über das `ldap` Modul, werden folgende Dateien
bearbeitet:

- sites-enabled/default
- sites-enabled/inner-tunnel
- mods-enabled/eap
- mods-enabled/ldap
- mods-enabled/mschap

Für die genaue Erläuterung des `default` Servers siehe [2.1.1 Default Servers](#211-default-server) und
für die Erläuterung der `MAB` Konfiguration siehe [2.4 Konfiguration für MAB](#24-mab).

Eine vollständiger `default` Server mit `MAB` integriert sieht wie folgt aus:

    server default{
        listen{
            type = auth
            ipaddr = *
            port = 1812
        }

        authorize{
            if (!EAP-Message) {
                update request {
                    User-Name := "%{User-Name}"
                }
            }
            ldap
            eap
        }

        authenticate {
            eap
        }

        post-auth {
            vlan_from_ou
        }
    }

Die `post-auth` Sektion ist notwendig, falls nach erfolgreicher Authentifizierung, VLAN-Zuweisungen und Nutzerrollen aus der AD
entnommen werden sollen. Siehe [3.2 VLAN Policy für AD](#32-vlan-policy-für-ad). Dies gilt ebenfalls für den `inner-tunnel`.

Das `ldap` Modul verbindet sich vor der Authentifizierung mit der AD und speichert die nötigen Attribute.

Beispiel Konfiguration des `ldap` Moduls:

    ldap{
        server = 'IP_ADRESSE'
        port = 389
        base_dn = 'DC=example,DC=org'
        identity = 'CN=Administrator,CN=Users,DC=example,DC=org'
        password = 'Sup3rs3curePassw0rd!'

        user {
            base_dn = "${..base_dn}"
            filter  = "(sAMAccountName=%{%{Stripped-User-Name}:-%{User-Name}})"
            scope = "sub"
        }

        group {
            membership_attribute = memberOf
        }

        update {
            control: += "memberOf"
            control: += "userPrincipalName"
        }

        options {
            chase_referrals = yes
            rebind = yes
        }

        timeout = 4
        timelimit = 3
        net_timeout = 1
    }

`user` Abschnitt:

- `base_dn` definiert den Start der Nutzer Suche.
- `filter` definiert, wie ein Benutzer gefunden wird. Dabei soll der `Stripped-User-Name` (ohne Domain) verwendet werden,
  falls er nicht existiert, wird `User-Name` genommen. Nützlich bei Umgebungen mit `Domain\User`.
- `scope = "sub"` gibt an, dass die Suche rekursiv in allen Unterstrukturen von base_dn erfolgen soll.

`group` Abschnitt:

- Definiert wie Gruppenmitgliedschaften geprüft werden sollen.
- `memberOf` wird von LDAP genutzt, um die Gruppen eines Nutzers zu erfassen.

`update` Abschnitt:

- Daten aus dem LDAP-Eintrag des Benutzers werden in FreeRADIUS-Attribute übernommen.
- `memberOf` speichert Gruppeninformationen.
- `userPrincipalName` speichert den UPN.

`options` Abschnitt:

- `chase_referrals = yes`: Wenn der LDAP-Server auf einen anderen Server verweist (Referral), folgt der FreeRADIUS-Server diesem Verweis automatisch.
  Wichtig in AD mit mehreren Domänen oder Domaincontrollern.
- `rebind=yes` zwingt den FreeRADIUS dazu sich neu zu verbinden, wenn ein Referral erfolgt.

Sonstiges:

- `timeout` definiert die maximale Dauer der LDAP-Anfrage in Sekunden.
- `timelimit` definiert das Zeitlimit für die Suchoption selbst.
- `net_timeout` definiert das Timeout fürr die Netwerkverbindung zu LDAP in Sekunden.

Als nächstes folgt die Konfiguration des `eap` Moduls. Dieses verwendet dieselbe Einstellungen aus [EAP-TLS](#212-eap-modul) mit dem folgenden Zusatz für `peap`:

    eap {
        default_eap_type = tls

        tls-config tls-common{
            ...
        }

        tls {
            ...
        }

        peap {
            tls = tls-common
            default_eap_type = mschapv2
            virtual_server = "inner-tunnel"
        }

        mschapv2{}
    }

Da `mschapv2` eine EAP Methode ist, muss sie in dem `eap` block noch einmal aufgerufen werden. Dies ist bei `pap` nicht notwendig.
Die Angabe eines `virtual_server` ist essenziell. Dieser ist zuständig für den Austausch von Authentifizierungsdaten und deren Überprüfung.

Eine `inner-tunnel` Konfiguration sieht wie folgt aus:

    server inner-tunnel{
        authorize{
            ldap
            mschap
            eap {
              ok = return
            }
            eap
        }

        authenticate{
            Auth-Type MS-CHAP{
              mschap
            }
            eap
        }

        post-auth {
            vlan_from_ou
        }
    }

Hier wird in dem `authenticate` Abschnitt der Auth-Type `MS-CHAP` auf das Modul `mschap` gemappt, damit der FreeRADIUS
den Auth-Type korrekt zuordnen und verarbeiten kann.

In der `authorize` Sektion wird das `mschap` Modul angegeben werden, da dies eine externe AD-Abfrage mit Benutzerdatenabgleich
initiert.

Für diesen Vorgang ist es erforderlich einen korrekt installierten `winbind` Prozess bereits konfiguriert zu haben.
FreeRADIUS hat für `mschap` zwei verschiedene Methoden für den Benutzerdatenabgleich, die beide `winbind` verwenden.

`ntlm_auth` ist ein Programm, welches der FreeRADIUS bei jeder Abfrage als eigenen System Prozess aufruft. Auch wenn diese Methode
laut offiziellen Angaben zuverlässiger ist, hat sich die zweite Methode erfahrungsgemäß deutlich besser bewährt.

FreeRADIUS bietet nämlich die Möglichkeit, direkt mit dem winbind daemon zu kommunizieren. Dies wird wie folgt in `mschap`
konfiguriert:

    mschap{
        winbind_username = "%{mschap:User-Name}"
        winbind_domain = "%{mschap:NT-Domain}"
    }

Nun kann der FreeRADIUS Server gestartet werden:

    systemctl enable freeradius
    systemctl start freeradius

Oder für Debug Zwecke:

    freeradius -X

## 2.4 MAB

Um Mac Authentication Bypass einzurichten, muss in dem `default` Server folgender
Ausschnitt in die `authorize` Sektion hinzugefügt werden:

    authorize{
      if (!EAP-Message) {
        update request {
          User-Name := "%{User-Name}"
        }
      }
    }

Dabei prüft FreeRADIUS die Anfrage nach Merkmalen einer EAP Nachricht. Sollte dies nicht der Fall sein, sendet
der Authenticator die MAC-Adresse des Supplicants als `User-Name`, welcher für den Authentifizierungsprozess
manuell gespeichert wird. Dadurch kann die MAC Adresse als Benutzername in der `mods-config/files/authorize`
Datei hinterlegt werden.

## 3. Sonstige Konfigurationen

- [3.1 Zertifikatserstellung](#31-zertifikatserstellung)
- [3.2 VLAN Policy für AD](#32-vlan-policy-für-ad)
- [3.3 Nutzer Konfiguration](#33-nutzer-konfiguration)
  - [3.3.1 VLAN Zuweisung](#331-vlan-zuweisung)
  - [3.3.2 Firewalling Aruba](#332-firewalling-aruba-aos-cx)
  - [3.3.3 Firewalling PicOS](#333-firewalling-fs-picos)
- [3.4 Logging Authentifizierungsversuche](#34-logging-authentifizierungsversuche)

## 3.1 Zertifikatserstellung

> ***Hinweis:***
>
> Bei der Erstellung der Zertifikate ist das `-subj` anzupassen.

### Root CA

Erzeugung des Private Key's:

    openssl ecparam -name prime256v1 -genkey -noout -out ca.key

Selbst-Signierung des CA Zertifikats:

    openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.pem \
    -subj "/C=DE/ST=State/L=City/O=Org/OU=CA/CN=Org-CA"

### Server

Erzeugung des Private Key's:

    openssl ecparam -name prime256v1 -genkey -noout -out server.key

Erstellung eines Certificate-Signing-Requests:

    openssl req -new -key server.key -out server.csr \
    -subj "/C=DE/ST=State/L=City/O=Org/OU=RADIUS/CN=radius.org"

Signierung mittels CA Zertifikat:

    openssl x509 -req -in server.csr -CA ca.pem -CAkey ca.key -CAcreateserial \
    -out server.pem -days 1095 -sha256

### Client

Erzeugung des Private Key's:

    openssl ecparam -name prime256v1 -genkey -noout -out client.key

Erstellung eines Certificate-Signing-Requests:

    openssl req -new -key client.key -out client.csr \
    -subj "/C=DE/ST=State/L=City/O=Org/OU=Users/CN=radiustest"

> ***Hinweis:***
> Der CN wird bei EAP-TLS als Identifier verwendet.

Signierung mittels CA Zertifikat:

    openssl x509 -req -in client.csr -CA ca.pem -CAkey ca.key \
    -CAcreateserial -out client.pem -days 1095 -sha256

## 3.2 VLAN Policy für AD

Um VLAN und Nutzerrollen über Active Directory zuzuweisen, wird in dem `policy.d` ein virtueller Server
eingerichtet und in der jeweiligen `post-auth` Sektion von `default` und `inner-tunnel` angegeben.

In dem folgenden Beispiel erfolgt die VLAN Zuweisung dynamisch anhand der Organization Unit `OU`, in der ein Nutzer
zugeordnet ist. Dabei wird der vollständige Distinguished Name `DN` aus der AD abgefragt und die VLAN ID aus der OU
herausgefiltert.

Beispiel DN:

    CN=radiustest,OU=VLAN100,OU=Users,DC=example,DC=org

`vlan_from_ou` Konfiguration:

    vlan_from_ou {
        if (&control:LDAP-UserDN) {
            update control { Tmp-String-0 := "%{control:LDAP-UserDN}" }
        }
        else {
            ok
            return
        }

        if (&control:Tmp-String-0 =~ /OU=VLAN\s*([0-9]+)/i) {
            update control {
                Tmp-Integer-0 := "%{1}"
            }
        }

        if (&control:Tmp-Integer-0) {
            update reply {
                Tunnel-Type := VLAN
                Tunnel-Medium-Type := IEEE-802
                Tunnel-Private-Group-Id := "%{control:Tmp-Integer-0}"
            }

            update control {
                Tmp-Integer-0 := 0
                Tmp-String-0  := ""
            }
        }
    }

- `control:LDAP-UserDN` ist ein Attribut, dass das LDAP Modul bereitstellt und den vollständigen DN des Benutzers enthält.
- `if (&control:Tmp-String-0 =~ /OU=VLAN\s*([0-9]+)/i) ...` filtert die VLAN ID und speichert sie in dem Integer `Tmp-Integer-0`.
- `if (&control:Tmp-Integer-0) ...` setzt die VLAN Nummer, falls eine gefunden wurde.
- `update control ...` setzt die Variablen zurück.

## 3.3 Nutzer Konfiguration

In dieser Sektion wird das Einrichten der Nutzerdatei für die Authentifizierung, VLAN und Rollenzuweisung beschrieben.
Die Datei befindet sich unter dem Pfad `mods-config/files/authorize` und funktioniert nur in Verbindung mit dem `files` Modul.

Dabei wird nach folgendem Schema vorgegangen:

    IDENTIFIER Attribut := "VERGLEICHSWERT"
            Tunnel-Type := VLAN,
            Tunnel-Medium-Type := IEEE-802,
            Tunnel-Private-Group-Id := ID,
            FILTER-RULE := "RULE",
            FILTER-RULE += "RULE"

Die `Tunnel` sowie `Filter` Attribute werden als eine Liste in dem `Access-Accept` geschickt, weshalb die Werte durch Kommas getrennt sind.
Die Operatoren `:=` und `+=` definieren dabei den ersten Zuweisungswert bzw. fügen weitere Werte hinzu.

### 3.3.1 VLAN Zuweisung

Für die Passwort-basierte Methode `EAP-TTLS-PAP` mit `sha256` als Hash Wert sieht eine VLAN Zuweisung wie folgt aus:

    radiustest SHA2-Password := 469fa782379ed44aac1f60edbb7abf3d0555685775e6944abf9a301751ea6b39
            Tunnel-Type := VLAN,
            Tunnel-Medium-Type := IEEE-802,
            Tunnel-Private-Group-Id := 10

Beispiel Konfiguration für `EAP-TLS`:

    radiustest TLS-Client-Cert-Common-Name := "radiustest@EXAMPLE.CORP"
            Tunnel-Type := VLAN,
            Tunnel-Medium-Type := IEEE-802,
            Tunnel-Private-Group-Id := 10

Beispiel Konfiguration für `MAB`:

    d481d7b536ce Auth-Type := Accept
            Tunnel-Type := VLAN,
            Tunnel-Medium-Type := IEEE-802,
            Tunnel-Private-Group-Id := 10

Bei `MAB` erfolgt keine richtige Authentifizierung, deshalb wird hier mit `Auth-Type := Accept` definiert, dass die angegebene MAC akzeptiert wird.

### 3.3.2 Firewalling Aruba AOS-CX

Um Firewall Regeln an Aruba Switche via RADIUS senden zu können, muss zuerst ein Attribut dafür in der `dictionary` Datei durch folgende Zeile angelegt werden:

    ATTRIBUTE       NAS-Filter-Rule                 92      string

Die Filter Rollen selbst werden in der `mods-config/files/authorize` Datei hinzugefügt.

Dabei wird folgende Syntax verwendet:

    NAS-Filter-Rule = "<permit|deny> in <ip|ip-protocol-value> from any to <any|host|<ip-addr>|ipv4-addr/mask|IPv6-address/prefix> [<tcp/udp-port|tcp/udp-port range> ] [cnt]"

Beispiele:

    NAS-Filter-Rule := "permit in tcp from any to any 23",
    NAS-Filter-Rule += "permit in ip from any to 10.10.10.1/24",
    NAS-Filter-Rule += "deny in ip from any to any"

Eine ausführlichere Dokumentation der Syntax findet man auf der [Aruba Seite](https://arubanetworking.hpe.com/techdocs/AOS-S/16.11/ASG/YC/content/common%20files/ace-syn-rad-ser.htm).

Das `NAS-Filter-Rule` Attribut wird wie folgt in der `authorize` Datei angehängt:

    radiustest Cleartext-Password := "#radius1!"
            Tunnel-Type := VLAN,
            Tunnel-Medium-Type := IEEE-802,
            Tunnel-Private-Group-Id := 10,
            NAS-Filter-Rule := "permit in tcp from any to any 23",
            NAS-Filter-Rule += "permit in ip from any to 10.10.10.1/24",
            NAS-Filter-Rule += "deny in ip from any to any"

### 3.3.3 Firewalling FS PICOS

Um Firewall Regeln an Aruba Switche via RADIUS senden zu können, müssen zuerst Attribute dafür in der `dictionary` Datei durch folgende Zeilen angelegt werden:

    ATTRIBUTE      Pica8-IP-Downloadable-ACL-Name  3       string
    ATTRIBUTE      Pica8-IP-Downloadable-ACL-Rule  2       string

Beispiele:

    radiustest Cleartext-Password := "#radius1!"
            Tunnel-Type := VLAN,
            Tunnel-Medium-Type := IEEE-802,
            Tunnel-Private-Group-Id := 10,
            Pica8-IP-Downloadable-ACL-Name := "DENY_ICMP",
            Pica8-IP-Downloadable-ACL-Rule := "sequence 10 from protocol icmp from source-address-ipv4 10.10.10.1/32 from destination-address-ipv4 10.10.10.2/32 then action discard",
            Pica8-IP-Downloadable-ACL-Rule := "sequence 20 then action forward"

Eine ausführlichere Dokumentation über die Syntax und unterstützten Keywords findet man [hier](https://pica8-fs.atlassian.net/wiki/spaces/PicOS443sp/pages/10455780/Principle+of+NAC#PrincipleofNAC-DownloadableACL).

## 3.4 Logging Authentifizierungsversuche

Um fehlgeschlagene Authentifizierungsversuche zu loggen, wird die `radiusd.conf` Datei bearbeitet.

Dabei gibt es die folgenden verschiedenen Output Möglichkeiten:

    - files => logs werden in eine Datei geschrieben
    - syslog => logs werden nach syslog geschrieben
    - stdout
    - stderr

In allen Fällen wird die `log` Sektion der `radiusd.conf` angepasst.

Konfiguraiton am Beispiel `files`:

    log {
        destination = files
        file = ${logdir}/radius.log
        auth_reject = yes
        auth_badpass = no
        stripped_names = yes

        syslog_facility = daemon
        colourise = yes
    }

- `destination` gibt eine der vier Output Möglichkeiten an.
- `file` legt die Logdatei fest, falls `files` verwendet wird.
- `auth_reject = yes` gibt an, dass fehlgeschlagene Authentifizierungsversuche geloggt werden.
- `auth_badpass = no` gibt an, dass das Passwort nicht geloggt werden soll bei einem Fehlversuch.
- `stripped_names = yes` gibt an, dass das vollständige `User-Name` Attribut geloggt werden soll.
- `syslog_facility` gibt die `facility` an, in die im Falle von `syslog` geschrieben werden soll.
- `colourise = yes` markiert wichtige Nachrichten farbig, falls `stdout` oder `stderr` verwendet wird.

## 4. Switch Konfiguration

- [4.1 Aruba AOS-CX](#41-aruba-aos-cx)
- [4.2 FS PICOS](#42-fs-picos)

## 4.1 Aruba AOS-CX

Um `dot1x` und `MAB` Authentifizierung nutzen zu können müssen diese vorab global aktiviert werden:

    config
    aaa authentication port-access dot1x authenticator enable
    aaa authentication port-access mac-auth enable

Als nächstes wird der RADIUS Server eingerichtet:

    radius-server host [HOST_IP]
    radius-server key [SHARED SECRET]

Da `EAPOL` Frames als `untagged` am Switch ankommen, wird ein sogenanntes `uncontrolled` VLAN benötigt um den Port aktiv zu halten und Frames verarbeiten zu können.
Als `interface` werden die Ports angegeben, auf welchen `dot1x` Authentifizierung stattfinden soll.

    vlan [VLAN-ID]
    name "uncontrolled"
    interface [INTERFACE]
    vlan access [VLAN-ID]

Nun können die Interface Einstellungen für `dot1x` getroffen werden:

    interface [INTERFACE]
    no shutdown
    aaa authentication port-access auth-mode device-mode
    aaa authentication port-access auth-precedence dot1x mac-auth
    aaa authentication port-access auth-priority dot1x mac-auth
    aaa authentication port-access dot1x authenticator enable

`device-mode` gibt an, dass nur das Gerät authentifiziert werden soll und nicht der ganze Port.
`auth-precedence` legt die Reihenfolge der Authentifizierungsversuche fest.
`auth-priority` legt die Priorität bei gleichzeitiger Authentifizierung fest.
`authenticator enable` aktiviert den Port für `dot1x` Authentifizierung.

Weitere `aaa authentication port-access dot1x authenticator` Einstellungen:

- `reauth` aktiviert periodischen Re-Authentifizierungsprozess.
- `reauth-perioid` gibt die Zeitspanne an, in der erneut authentifiziert wird.
- `eapol-timeout` gibt das Timeout an, um auf die Antwort des Clients zu warten, bevor EAPOL erneut gesendet wird.
- `discovery-period` gibt die Zeitspanne an, nach der erneut ein EAPOL Request/Identity gesendet wird.
- `maximum-eapol-requests` gibt die maximale Anzahl an EAPOL Nachrichten an, die an den Supplicant gesendet werden, bevor die Authentifizierung fehlschlägt.
- `max-retries` gibt die Anzahl an Versuchen an, den Client zu authentifizieren, bevor diese fehlschlägt.

Wichtige `aaa authentication port-access mac-auth` Einstellungen für die `MAB` Authentifizierung:

- `enable` aktiviert MAC Authentifizierung.
- `quiet-period` gibt die Zeit an, in der nicht versucht wird, den Client zu authentifizieren.
- `reauth` aktiviert periodischen Re-Authentifizierungsprozess.
- `reauth-perioid` gibt die Zeitspanne an, in der erneut authentifiziert wird.

## 4.2 FS PICOS

Als erstes wird das Interface für OOB-Management eingerichtet:

    configure
    set system management-ehternet eth0 ip-address [IP]/[CIDR]
    set system management-ethernet eth0 ip-gateway [IP]
    commit

Als nächstes wird der RADIUS konfiguriert:

    configure
    set protocols dot1x aaa radius authentication
    set protocols dot1x aaa radius authentication server-ip [IP] port [PORT]
    set protocols dot1x aaa radius authentication server-ip [IP] shared-key [KEY]
    set protocols dot1x aaa radius authentication nas-ip [IP]
    commit

Nun wird das Interface für `dot1x` und `MAB` konfiguriert:

    configure
    set vlans vlan-id 99
    set interface gigabit-ethernet ge-1/1/1 family ethernet-switching port-mode trunk
    set interface gigabit-ethernet ge-1/1/1 family ethernet-switching native-vlan-id 99
    set vlans vlan-id 99 l3-interface vlan99
    set l3-interface vlan-interface vlan99 address [IP] prefix-length [CIDR]
    commit
    set protocols dot1x interface ge-1/1/1 auth-mode 802.1x
    set protocols dot1x interface ge-1/1/1 auth-mode mac-radius
    set protocols dot1x interface ge-1/1/1 host-mode single
    commit

## 5. Troubleshooting

Manchmal kann es sein, dass bei einem `Access-Reject` Attribute bezüglich der VLAN-Zuweisung nicht korrekt zurückgesetzt werden. Dies hat zur Folge, dass dem Client die VLAN-Zuweisungs Attribute der letzten Session mitgeschickt werden und fälschlicherweise der Client einem VLAN zugewiesen wird, obwohl er abgelehnt wurde.

Das kann verhindert werden, indem man die Attribute in der `post-auth` Sektion des `default` Servers zurrücksetzt:

    server default{
        listen{
            ...
        }

        authorize{
            ...
        }
        authenticate{
            ...
        }

        post-auth {
            if (reject) {
                update reply {
                    Tunnel-Type !* ANY
                    Tunnel-Medium-Type !* ANY
                    Tunnel-Private-Group-Id !* ANY
                }
            }
        }
    }
