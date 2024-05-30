# How to Set Up an IKEv2 VPN Server with StrongSwan on Ubuntu 20.04

[Jamon Camisso](https://www.digitalocean.com/community/users/jamonation)

*A previous version of this tutorial was written by [Justin Ellingwood](https://www.digitalocean.com/community/users/jellingwood) and [Namo](https://www.digitalocean.com/community/users/namo)*

## Introduction

A virtual private network, or VPN, allows you to securely encrypt traffic as it travels through untrusted networks, such as those at the coffee shop, a conference, or an airport.

Internet Key Exchange v2, or IKEv2, is a protocol that allows for direct IPSec tunneling between the server and client. In IKEv2 VPN implementations, IPSec provides encryption for the network traffic. IKEv2 is natively supported on some platforms (OS X 10.11+, iOS 9.1+, and Windows 10) with no additional applications necessary, and it handles client hiccups quite smoothly.

In this tutorial, you’ll set up an IKEv2 VPN server using StrongSwan on an Ubuntu 20.04 server. You’ll then learn how to connect to it with Windows, macOS, Ubuntu, iOS, and Android clients.

## Prerequisites

To complete this tutorial, you will need:

One Ubuntu 20.04 server configured by following the Ubuntu 20.04 initial server setup guide, including a sudo non-root user and a firewall.

## Step 1 — Installing StrongSwan

First, we’ll install StrongSwan, an open-source IPSec daemon which we’ll configure as our VPN server. We’ll also install the public key infrastructure (PKI) component so that we can create a Certificate Authority (CA) to provide credentials for our infrastructure.

Start by updating the local package cache:

```
sudo apt update
```

Then install the software by typing:

```
sudo apt install strongswan strongswan-pki libcharon-extra-plugins libcharon-extauth-plugins libstrongswan-extra-plugins
```

The additional `libcharon-extauth-plugins` package is used to ensure that various clients can authenticate to your server using a shared username and passphrase. The `libstrongswan-extra-plugins` package is included so that Strongswan supports elliptic curve cipher suites that use the `Curve25519` cryptography suite.

Now that everything’s installed, let’s move on to creating our certificates.

## Step 2 — Creating a Certificate Authority

An IKEv2 server requires a certificate to identify itself to clients. To help create the required certificate, the `strongswan-pki` package comes with a utility called `pki` to generate a Certificate Authority and server certificates.

To begin, let’s create a few directories to store all the assets we’ll be working on. The directory structure matches some of the directories in `/etc/ipsec.d`, where we will eventually move all of the items we create:

```
mkdir -p ~/pki/{cacerts,certs,private}
```

Then lock down the permissions so that our private files can’t be seen by other users:

```
chmod 700 ~/pki
```

Now that we have a directory structure to store everything, we can generate a root key. This will be a 4096-bit RSA key that will be used to sign our root certificate authority.

Execute these commands to generate the key:

```
pki --gen --type rsa --size 4096 --outform pem > ~/pki/private/ca-key.pem
```

Following that we can move on to creating our root certificate authority, using the key that we just generated to sign the root certificate:

```
pki --self --ca --lifetime 3650 --in ~/pki/private/ca-key.pem \
    --type rsa --dn "CN=VPN root CA" --outform pem > ~/pki/cacerts/ca-cert.pem
```

The `--lifetime 3650` flag is used to ensure that the certificate authority’s root certificate will be valid for 10 years. The root certificate for an authority does not change typically, since it would have to be redistributed to every server and client that rely on it, so 10 years is a safe default expiry value.

You can change the distinguished name (DN) value to something else if you would like. The common name (CN field) here is just the indicator, so it doesn’t have to match anything in your infrastructure.

Now that we’ve got our root certificate authority up and running, we can create a certificate that the VPN server will use.

## Step 3 — Generating a Certificate for the VPN Server

We’ll now create a certificate and key for the VPN server. This certificate will allow the client to verify the server’s authenticity using the CA certificate we just generated.

First, create a private key for the VPN server with the following command:

```
pki --gen --type rsa --size 4096 --outform pem > ~/pki/private/server-key.pem
```

Now, create and sign the VPN server certificate with the certificate authority’s key you created in the previous step. Execute the following command, but change the Common Name (CN) and the Subject Alternate Name (SAN) field to your VPN server’s DNS name or IP address:

```
pki --pub --in ~/pki/private/server-key.pem --type rsa \
    | pki --issue --lifetime 1825 \
        --cacert ~/pki/cacerts/ca-cert.pem \
        --cakey ~/pki/private/ca-key.pem \
        --dn "CN=server_domain_or_IP" --san server_domain_or_IP \
        --flag serverAuth --flag ikeIntermediate --outform pem \
    >  ~/pki/certs/server-cert.pem
```

> Note: If you are using an IP address instead of a DNS name, you will need to specify multiple `--san` entries. The line in the previous command block where you specify the distinguished name (`--dn ...`) will need to be modified with the extra entry like the following excerpted line:

```
--dn "CN=IP address" --san @IP_address --san IP_address \
```

> The reason for this extra `--san @IP_address` entry is that some clients will check whether the TLS certificate has both an DNS entry and an IP Address entry for a server when they verify its identity.

The `--flag serverAuth` option is used to indicate that the certificate will be used explicitly for server authentication, before the encrypted tunnel is established. The `--flag ikeIntermediate` option is used to support older macOS clients.

Now that we’ve generated all of the TLS/SSL files StrongSwan needs, we can move the files into place in the `/etc/ipsec.d` directory by typing:

```
sudo cp -r ~/pki/* /etc/ipsec.d/
```

In this step, we’ve created a certificate pair that will be used to secure communications between the client and the server. We’ve also signed the certificates with the CA key, so the client will be able to verify the authenticity of the VPN server using the CA certificate. With all of these certificates ready, we’ll move on to configuring the software.

## Step 4 — Configuring StrongSwan

StrongSwan has a default configuration file with some examples, but we will have to do most of the configuration ourselves. Let’s back up the file for reference before starting from scratch:

```
sudo mv /etc/ipsec.conf{,.original}
```

Create and open a new blank configuration file using your preferred text editor. Here, we’ll use `nano`:

```
sudo nano /etc/ipsec.conf
```

> Note: As you work through this section to configure the server portion of your VPN, you will encounter settings that refer to left and right sides of a connection. When working with IPSec VPNs, the left side by convention refers to the local system that you are configuring, in this case the server. The right side directives in these settings will refer to remote clients, like phones and other computers.

> When you move on to configuring clients later in this tutorial, the client configuration files will refer to themselves using various left directives, and the server will be referred to using right side terminology.

First, we’ll tell StrongSwan to log daemon statuses for debugging and allow duplicate connections. Add these lines to the file:

```
config setup
    charondebug="ike 1, knl 1, cfg 0"
    uniqueids=no
```

Then, we’ll create a configuration section for our VPN. We’ll also tell StrongSwan to create IKEv2 VPN Tunnels and to automatically load this configuration section when it starts up. Append the following lines to the file:

```
. . .
conn ikev2-vpn
    auto=add
    compress=no
    type=tunnel
    keyexchange=ikev2
    fragmentation=yes
    forceencaps=yes
```

We’ll also configure dead-peer detection to clear any “dangling” connections in case the client unexpectedly disconnects. Add these lines:

```
. . .
conn ikev2-vpn
    . . .
    dpdaction=clear
    dpddelay=300s
    rekey=no
```

Next, we’ll configure the server’s “left” side IPSec parameters. Each of the following parameters ensures that the server is configured to accept connections from clients and to identify itself correctly. You’ll add each of these settings to the `/etc/ipsec.conf` file once you are familiar with what they are and why they are used:

- `left=%any` The `%any` value ensures that the server will use the network interface where it receives incoming connections for subsequent communication with clients. For example, if you are connecting a client over a private network, the server will use the private IP address where it receives traffic for the rest of the connection.
- `leftid=@server_domain_or_IP` This option controls the name that the server presents to clients. When combined with the next option `leftcert`, the `leftid` option ensures that the server’s configured name and the Distinguished Name (DN) that is contained in the public certificate match.
- `leftcert=server-cert.pem` This option is the path to the public certificate for the server that you configured in Step 3. Without it, the server will not be able to authenticate itself with clients, or finish negotiating the IKEv2 set up.
- `leftsendcert=always` The `always` value ensures that any client that connects to the server will always receive a copy of the server’s public certificate as part of the initial connection set up.
- `leftsubnet=0.0.0.0/0` The last “left” side option that you will add tells clients about the subnets that are reachable behind the server. In this case, `0.0.0.0/0` is used to represent the entire set of IPv4 addresses, meaning that the server will tell clients to send all their traffic over the VPN by default.

Now that you are familiar with each of the relevant “left” side options, add them all to the file like this:

```
. . .
conn ikev2-vpn
    . . .
    left=%any
    leftid=@server_domain_or_IP
    leftcert=server-cert.pem
    leftsendcert=always
    leftsubnet=0.0.0.0/0
```

> Note: When configuring the server ID (leftid), only include the @ character if your VPN server will be identified by a domain name:

```
    . . .
    leftid=@vpn.example.com
    . . .
```

If the server will be identified by its IP address, just put the IP address in:

```
    . . .
    leftid=your_server_ip
    . . .
```

Next, we can configure the client’s “right” side IPSec parameters. Each of the following parameters tells the server how to accept connections from clients, how clients should authenticate to the server, and the private IP address ranges and DNS servers that clients will use. Add each of these settings to the `/etc/ipsec.conf` file once you are familiar with what they are and why they are used:

- `right=%any` The `%any` option for the `right` side of the connection instructs the server to accept incoming connections from any remote client.
- `rightid=%any` This option ensures that the server will not reject connections from clients that provide an identity before the encrypted tunnel is established.
- `rightauth=eap-mschapv2` This option configures the authentication method that clients will use to authenticate to the server. `eap-mschapv2` is used here for broad compatibility to support clients like Windows, macOS, and Android devices.
- `rightsourceip=10.10.10.0/24` This option instructs the server to assign private IP addresses to clients from the specified `10.10.10.0/24` pool of IPs.
- `rightdns=8.8.8.8,8.8.4.4` These IP addresses are Google’s public DNS resolvers. They can be changed to use other public resolvers, the VPN server’s resolvers, or any other resolver that clients can reach.
- `rightsendcert=never` This option instructs the server that clients do not need to send a certificate to authenticate themselves.

Now that you are familiar with the required “right” side options for the VPN, add the following lines to `/etc/ipsec.conf`:

```
. . .
conn ikev2-vpn
    . . .
    right=%any
    rightid=%any
    rightauth=eap-mschapv2
    rightsourceip=10.10.10.0/24
    rightdns=8.8.8.8,8.8.4.4
    rightsendcert=never
```

Now we’ll tell StrongSwan to ask the client for user credentials when they connect:

```
. . .
conn ikev2-vpn
    . . .
    eap_identity=%identity
```

Finally, add the following lines to support Linux, Windows, macOS, iOS, and Android clients. These lines specify the various key exchange, hashing, authentication, and encryption algorithms (commonly referred to as *Cipher Suites*) that StrongSwan will allow different clients to use:

```
. . .
conn ikev2-vpn
    . . .
    ike=chacha20poly1305-sha512-curve25519-prfsha512,aes256gcm16-sha384-prfsha384-ecp384,aes256-sha1-modp1024,aes128-sha1-modp1024,3des-sha1-modp1024!
    esp=chacha20poly1305-sha512,aes256gcm16-ecp384,aes256-sha256,aes256-sha1,3des-sha1!
```

Each supported cipher suite is delineated from the others by a comma. For example `chacha20poly1305-sha512-curve25519-prfsha512` is one suite, and `aes256gcm16-sha384-prfsha384-ecp384` is another. The cipher suites that are listed here are selected to ensure the widest range of compatibility across Windows, macOS, iOS, Android, and Linux clients.

The complete configuration file should look like this:

```
config setup
    charondebug="ike 1, knl 1, cfg 0"
    uniqueids=no

conn ikev2-vpn
    auto=add
    compress=no
    type=tunnel
    keyexchange=ikev2
    fragmentation=yes
    forceencaps=yes
    dpdaction=clear
    dpddelay=300s
    rekey=no
    left=%any
    leftid=@server_domain_or_IP
    leftcert=server-cert.pem
    leftsendcert=always
    leftsubnet=0.0.0.0/0
    right=%any
    rightid=%any
    rightauth=eap-mschapv2
    rightsourceip=10.10.10.0/24
    rightdns=8.8.8.8,8.8.4.4
    rightsendcert=never
    eap_identity=%identity
    ike=chacha20poly1305-sha512-curve25519-prfsha512,aes256gcm16-sha384-prfsha384-ecp384,aes256-sha1-modp1024,aes128-sha1-modp1024,3des-sha1-modp1024!
    esp=chacha20poly1305-sha512,aes256gcm16-ecp384,aes256-sha256,aes256-sha1,3des-sha1!
```

Save and close the file once you’ve verified that you’ve added each line correctly. If you used `nano`, do so by pressing `CTRL + X`, `Y`, then `ENTER`.

Now that we’ve configured the VPN parameters, let’s move on to creating an account so our users can connect to the server.

## Step 5 — Configuring VPN Authentication

Our VPN server is now configured to accept client connections, but we don’t have any credentials configured yet. We’ll need to configure a couple things in a special configuration file called `ipsec.secrets`:

- We need to tell StrongSwan where to find the private key for our server certificate, so the server will be able to authenticate to clients.
- We also need to set up a list of users that will be allowed to connect to the VPN.

Let’s open the secrets file for editing:

```
sudo nano /etc/ipsec.secrets
```

First, we’ll tell StrongSwan where to find our private key and how to parse it.

```
: RSA "server-key.pem"
```

Make sure that the line begins with the `:` character and that there is a space after it so that the entire line reads `: RSA "server-key.pem"`.

Then, we’ll define the user credentials. You can make up any username or password combination that you like:

```
your_username : EAP "your_password"
```

Save and close the file. Now that we’ve finished working with the VPN parameters, we’ll restart the VPN service so that our configuration is applied:

```
sudo systemctl restart strongswan-starter
```

Now that the VPN server has been fully configured with both server options and user credentials, it’s time to move on to configuring the most important part: the firewall.

## Step 6 — Configuring the Firewall & Kernel IP Forwarding

With the StrongSwan configuration complete, we need to configure the firewall to allow VPN traffic through and forward it.

If you followed the prerequisite initial server setup tutorial, you should have a UFW firewall enabled. If you don’t yet have UFW configured, you should start by adding a rule to allow SSH connections through the firewall so your current session doesn’t close when you enable UFW:

```
sudo ufw allow OpenSSH
```

Then enable the firewall by typing:

```
sudo ufw enable
```

Then, add a rule to allow UDP traffic to the standard IPSec ports, `500` and `4500`:

```
sudo ufw allow 500,4500/udp
```

Next, we will open up one of UFW’s configuration files to add a few low-level policies for routing and forwarding IPSec packets. Before we we can do this, though, we need to find which network interface on our server is used for internet access. Find this interface by querying for the device associated with the default route:

```
ip route show default
```

Your public interface should follow the word “dev”. For example, this result shows the interface named `eth0`, which is highlighted in the following example:

```
Output
default via your_server_ip dev eth0 proto static
```

When you have your public network interface, open the `/etc/ufw/before.rules` file in your text editor. The rules in this file are added to the firewall before the rest of the usual input and output rules. They are used to configure network address translation (NAT) so that the server can correctly route connections to and from clients and the Internet.

```
sudo nano /etc/ufw/before.rules
```

Near the top of the file (before the `*filter` line), add the following configuration block. Change each instance of `eth0` in the above configuration to match the interface name you found with `ip route`. The `*nat` lines create rules so that the firewall can correctly route and manipulate traffic between the VPN clients and the internet. The `*mangle` line adjusts the maximum packet segment size to prevent potential issues with certain VPN clients:

```
*nat
-A POSTROUTING -s 10.10.10.0/24 -o eth0 -m policy --pol ipsec --dir out -j ACCEPT
-A POSTROUTING -s 10.10.10.0/24 -o eth0 -j MASQUERADE
COMMIT

*mangle
-A FORWARD --match policy --pol ipsec --dir in -s 10.10.10.0/24 -o eth0 -p tcp -m tcp --tcp-flags SYN,RST SYN -m tcpmss --mss 1361:1536 -j TCPMSS --set-mss 1360
COMMIT

*filter
:ufw-before-input - [0:0]
:ufw-before-output - [0:0]
:ufw-before-forward - [0:0]
:ufw-not-local - [0:0]
. . .
```

Next, after the `*filter` and chain definition lines, add one more block of configuration:

```
. . .
*filter
:ufw-before-input - [0:0]
:ufw-before-output - [0:0]
:ufw-before-forward - [0:0]
:ufw-not-local - [0:0]

-A ufw-before-forward --match policy --pol ipsec --dir in --proto esp -s 10.10.10.0/24 -j ACCEPT
-A ufw-before-forward --match policy --pol ipsec --dir out --proto esp -d 10.10.10.0/24 -j ACCEPT
```

These lines tell the firewall to forward ESP (Encapsulating Security Payload) traffic so the VPN clients will be able to connect. ESP provides additional security for our VPN packets as they’re traversing untrusted networks.

When you’re finished, ave and close the file once you’ve verified that you’ve added each line correctly. If you used `nano`, do so by pressing `CTRL + X`, `Y`, then `ENTER`.

Before restarting the firewall, we’ll change some network kernel parameters to allow routing from one interface to another. The file that controls these settings is called `/etc/ufw/sysctl.conf`. We’ll need to configure a few things in the file.

First IPv4 packet forwarding needs to be turned on so that traffic can move between the VPN and public facing network interfaces on the server. Next we’ll disable Path MTU discovery to prevent packet fragmentation problems. Finally we will not accept ICMP redirects nor send ICMP redirects to prevent man-in-the-middle attacks.

Open UFW’s kernel parameters configuration file using `nano` or your preferred text editor:

```
sudo nano /etc/ufw/sysctl.conf
```

Now add the following `net/ipv4/ip_forward=1` setting at the end of the file to enable forwarding packets between interfaces:

```
. . .
net/ipv4/ip_forward=1
```

Next block sending and receiving ICMP redirect packets by adding the following lines to the end of the file:

```
. . .
net/ipv4/conf/all/accept_redirects=0
net/ipv4/conf/all/send_redirects=0
```

Finally, turn off Path MTU discovery by adding this line to the end of the file:

```
. . .
net/ipv4/ip_no_pmtu_disc=1
```

Save the file when you are finished. Now we can enable all of our changes by disabling and re-enabling the firewall, since UFW applies these settings any time that it restarts:

```
sudo ufw disable
sudo ufw enable
```

You’ll be prompted to confirm the process. Type `Y` to enable UFW again with the new settings.

## Step 7 — Testing the VPN Connection on Windows, macOS, Ubuntu, iOS, and Android

Now that you have everything set up, it’s time to try it out. First, you’ll need to copy the CA certificate you created and install it on your client device(s) that will connect to the VPN. The easiest way to do this is to log into your server and output the contents of the certificate file:

```
cat /etc/ipsec.d/cacerts/ca-cert.pem
```

You’ll see output similar to this:

```
Output
-----BEGIN CERTIFICATE-----
MIIFNDCCAxygAwIBAgIIHCsidG5mXzgwDQYJKoZIhvcNAQEMBQAwODELMAkGA1UE

. . .

H2YUdz8XNHrJHvMQKWFpi0rlEcMs+MSXPFWE3Q7UbaZJ/h8wpSldSUbQRUlphExJ
dJ4PX+MUJO/vjG1/ie6Kh25xbBAc3qNq8siiJZDwrg6vjEK7eiZ1rA==
-----END CERTIFICATE-----
```

Copy this output to your computer, including the `-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----` lines, and save it to a file with a recognizable name, such as `ca-cert.pem`. Ensure the file you create has the `.pem` extension.

Alternatively, use SFTP to transfer the file to your computer.

Once you have the `ca-cert.pem` file downloaded to your computer, you can set up the connection to the VPN.

## Connecting from Windows

There are multiple ways to import the root certificate and configure Windows to connect to a VPN. The first method uses graphical tools for each step. The second method uses PowerShell commands, which can be scripted and modified to suit your VPN configuration.

> Note: These instructions have been tested on Windows 10 installations running versions 1903 and 1909.

### Configuring Windows with Graphical Tools

First, import the root certificate by following these steps:

- Press `WINDOWS+R` to bring up the **Run** dialog, and enter `mmc.exe` to launch the Windows Management Console.

- From the **File** menu, navigate to **Add or Remove Snap-in**, select **Certificates** from the list of available snap-ins, and click **Add**.

- We want the VPN to work with any user, so select **Computer Account** and click **Next**.

- We’re configuring things on the local computer, so select **Local Computer**, then click **Finish**.

- Under the **Console Root** node, expand the **Certificates (Local Computer)** entry, expand **Trusted Root Certification Authorities**, and then select the **Certificates** entry

- From the **Action** menu, select **All Tasks** and click **Import** to display the Certificate Import Wizard. Click **Next** to move past the introduction.

- On the **File to Import** screen, press the **Browse** button, ensure that you change the file type from “X.509 Certificate (.cer;.crt)” to “All Files (.)”, and select the `ca-cert.pem` file that you’ve saved. Then click **Next**.

- Ensure that the **Certificate Store** is set to **Trusted Root Certification Authorities**, and click **Next**.

- Click **Finish** to import the certificate.

Then configure the VPN with these steps:

- Launch **Control Panel**, then navigate to the **Network and Sharing Center**.
- Click on **Set up a new connection or network**, then select **Connect to a workplace**.
- Select **Use my Internet connection (VPN)**.
- Enter the VPN server details. Enter the server’s domain name or IP address in the **Internet address** field, then fill in **Destination name** with something that describes your VPN connection. Then click **Done**.

### Configuring Windows using PowerShell

To import the root CA certificate using PowerShell, first open a PowerShell prompt with administrator privileges. To do so, right click the Start menu icon and select `Windows PowerShell (Admin)`. You can also open a command prompt as administrator and type `powershell`.

Next we’ll import the certificate using the `Import-Certificate` PowerShell cmdlet. In the following command, the first `-CertStoreLocation` argument will ensure that the certificate is imported into the computer’s **Trusted Root Certification Authorities** store so that all programs and users will be able to verify the VPN server’s certificate. The `-FilePath` argument should point to the location where you copied the certificate. In the following example the path is `C:\Users\sammy\Documents\ca-cert.pem`. Ensure that you edit the command to match the location that you used.

```
Import-Certificate `
    -CertStoreLocation cert:\LocalMachine\Root\ `
    -FilePath C:\users\sammy\Documents\ca-cert.pem
```

The command will output something like the following:

```
Output
   PSParentPath: Microsoft.PowerShell.Security\Certificate::LocalMachine\Root

Thumbprint                                Subject
----------                                -------
DB00813B4087E9367861E8463A60CEA0ADC5F002  CN=VPN root CA
```

Now to configure the VPN using PowerShell, run the following command. Substitute your server’s DNS name or IP address on the `-ServerAddress` line. The various flags will ensure that Windows is correctly configured with the appropriate security parameters that match the options that you set in `/etc/ipsec.conf`.

```
Add-VpnConnection -Name "VPN Connection" `
    -ServerAddress "server_domain_or_IP" `
    -TunnelType "IKEv2" `
    -AuthenticationMethod "EAP" `
    -EncryptionLevel "Maximum" `
    -RememberCredential `
```

If the command is successful there will not be any output. To confirm the VPN is configured correctly, use the `Get-VPNConnection` cmdlet:

```
Get-VpnConnection -Name "VPN Connection"
```

You will receive output like the following:

```
Output
Name                  : VPN Connection
ServerAddress         : your_server_ip
AllUserConnection     : False
Guid                  : {B055A1AB-175C-4028-B4A8-D34309A2B20E}
TunnelType            : Ikev2
AuthenticationMethod  : {Eap}
EncryptionLevel       : Maximum
L2tpIPsecAuth         :
UseWinlogonCredential : False
EapConfigXmlStream    : #document
ConnectionStatus      : Disconnected
RememberCredential    : True
SplitTunneling        : False
DnsSuffix             :
IdleDisconnectSeconds : 0
```

By default Windows chooses older and slower algorithms. Run the `Set-VpnConnectionIPsecConfiguration` cmdlet to upgrade the encryption parameters that Windows will use for the IKEv2 key exchange, and to encrypt packets:

```
Set-VpnConnectionIPsecConfiguration -Name "VPN Connection" `
    -AuthenticationTransformConstants GCMAES256 `
    -CipherTransformConstants GCMAES256 `
    -DHGroup ECP384 `
    -IntegrityCheckMethod SHA384 `
    -PfsGroup ECP384 `
    -EncryptionMethod GCMAES256
```

> Note: If you would like to delete the VPN connection and reconfigure it with different options, you can run the Remove-VpnConnection cmdlet.
```
Remove-VpnConnection -Name "VPN Connection" -Force
```
> The `-Force` flag will skip prompting you to confirm the removal. You must be disconnected from the VPN if you attempt to remove it using this command.

### Connecting to the VPN

Once you have the certificate imported and the VPN configured using either method, your new VPN connection will be visible under the list of networks. Select the VPN and click **Connect**. You’ll be prompted for your username and password. Type them in, click **OK**, and you’ll be connected.

## Connecting from macOS

Follow these steps to import the certificate:

- Double-click the certificate file. **Keychain Access** will pop up with a dialog that says “Keychain Access is trying to modify the system keychain. Enter your password to allow this.”
- Enter your password, then click on **Modify Keychain**
- Double-click the newly imported VPN certificate. This brings up a small properties window where you can specify the trust levels. Set **IP Security (IPSec)** to **Always Trust** and you’ll be prompted for your password again. This setting saves automatically after entering the password.

Now that the certificate is imported and trusted, configure the VPN connection with these steps:

- Go to **System Preferences** and choose **Network**.
- Click on the small “plus” button on the lower-left of the list of networks.
- In the popup that appears, set **Interface** to **VPN**, set the **VPN Type** to **IKEv2**, and give the connection a name.
- In the **Server** and **Remote ID** field, enter the server’s domain name or IP address. Leave the **Local ID** blank.
- Click on **Authentication Settings**, select **Username**, and enter your username and password you configured for your VPN user. Then click **OK**.

Finally, click on **Connect** to connect to the VPN. You should now be connected to the VPN.

### Connecting from Ubuntu

To connect from an Ubuntu machine, you can set up and manage StrongSwan as a service or use a one-off command every time you wish to connect. Instructions are provided for both.

#### Managing StrongSwan as a Service

To manage StrongSwan as a service, you will need to perform the following configuration steps.
First, update your local package cache using `apt`

```
sudo apt update
```

Next, install StrongSwan and the required plugins for authentication:

```
sudo apt install strongswan libcharon-extra-plugins
```

Now you’ll need a copy of the CA certificate in the `/etc/ipsec.d/cacerts` directory so that your client can verify the server’s identity. Run the following command to copy the `ca-cert.pem` file into place:

```
sudo cp /tmp/ca-cert.pem /etc/ipsec.d/cacerts
```

To ensure the VPN only runs on demand, use `systemctl` to disable StrongSwan from running automatically:

```
sudo systemctl disable --now strongswan-starter
```

Next configure the username and password that you will use to authenticate to the VPN server. Edit `/etc/ipsec.secrets` using nano or your preferred editor:

```
sudo nano /etc/ipsec.secrets
```

Add the following line, editing the highlighted username and password values to match the ones that you configured on the server:

```
your_username : EAP "your_password"
```

Finally, edit the `/etc/ipsec.conf` file to configure your client to match the server’s configuration:

```
config setup

conn ikev2-rw
    right=server_domain_or_IP
    # This should match the `leftid` value on your server's configuration
    rightid=server_domain_or_IP
    rightsubnet=0.0.0.0/0
    rightauth=pubkey
    leftsourceip=%config
    leftid=username
    leftauth=eap-mschapv2
    eap_identity=%identity
    auto=start
```

To connect to the VPN, type:

```
sudo systemctl start strongswan-starter
```

To disconnect again, type:

```
sudo systemctl stop strongswan-starter
```

#### Using the `charon-cmd` Client for One-Off Connections

To manage StrongSwan as a service, you will need to perform the following configuration steps.

First, update your local package cache using `apt`

```
sudo apt update
```

Next, install StrongSwan and the required plugins for authentication:

```
sudo apt install strongswan libcharon-extra-plugins
```

Now you’ll need a copy of the CA certificate in the `/etc/ipsec.d/cacerts` directory so that your client can verify the server’s identity. Run the following command to copy the `ca-cert.pem` file into place:

```
sudo cp /tmp/ca-cert.pem /etc/ipsec.d/cacerts
```

At this point you can connect to the VPN server with `charon-cmd` using the server’s CA certificate, the VPN server’s IP address, and the username you configured.

Run the following command whenever you want to connect to the VPN:

```
sudo charon-cmd --cert ca-cert.pem --host vpn_domain_or_IP --identity your_username
```

When prompted, provide the VPN user’s password and you will be connected to the VPN. To disconnect, press `CTRL+C` in the terminal and wait for the connection to close.

### Connecting from iOS

To configure the VPN connection on an iOS device, follow these steps:

- Send yourself an email with the root certificate attached.
- Open the email on your iOS device and tap on the attached certificate file, then tap `Install` and enter your passcode. Once it installs, tap `Done`.
- Go to `Settings`, `General`, `VPN` and tap `Add VPN Configuration`. This will bring up the VPN connection configuration screen.
- Tap on `Type` and select `IKEv2`.
- In the `Description` field, enter a short name for the VPN connection. This could be anything you like.
- In the `Server` and `Remote ID` field, enter the server’s domain name or IP address. The `Local ID` field can be left blank.
- Enter your username and password in the `Authentication` section, then tap `Done`.
- Select the VPN connection that you just created, tap the switch on the top of the page, and you’ll be connected.

### Connecting from Android

Follow these steps to import the certificate:

- Send yourself an email with the CA certificate attached. Save the CA certificate to your downloads folder.
- Download the StrongSwan VPN client from the Play Store.
- Open the app. Tap the “more” icon (. . .) in the upper-right corner and select `CA certificates`.
- Tap the “more” icon (. . .) in the upper-right corner again. Select `Import certificate`.
- Browse to the CA certificate file in your downloads folder and select it to import it into the app.

Now that the certificate is imported into the StrongSwan app, you can configure the VPN connection with these steps:

- In the app, tap `ADD VPN PROFILE` at the top.
- Fill out the `Server` with your VPN server’s domain name or public IP address.
- Make sure `IKEv2 EAP (Username/Password)` is selected as the VPN Type.
- Fill out the `Username` and `Password` with the credentials you defined on the server.
- Deselect `Select automatically` in the `CA certificate` section and click `Select CA certificate`.
- Tap the `IMPORTED` tab at the top of the screen and choose the CA you imported (it will be named “VPN root CA” if you didn’t change the “DN” earlier).
- If you’d like, fill out `Profile name (optional)` with a more descriptive name.

When you wish to connect to the VPN, click on the profile you just created in the StrongSwan application.

### Troubleshooting Connections

If you are unable to import the certificate, ensure the file has the `.pem` extension, and not `.pem.txt`.

If you’re unable to connect to the VPN, check the server name or IP address you used. The server’s domain name or IP address must match what you’ve configured as the common name (CN) while creating the certificate. If they don’t match, the VPN connection won’t work. For example, if you set up a certificate with the CN of `vpn.example.com`, you must use `vpn.example.com` when you enter the VPN server details. Double-check the command you used to generate the certificate, and the values you used when creating your VPN connection.

Finally, double-check the VPN configuration to ensure the `leftid` value is configured with the `@` symbol if you’re using a domain name:

```
  leftid=@vpn.example.com
```

If you’re using an IP address, ensure that the `@` symbol is omitted. Also make sure that when you generated the `server-cert.pem` file that you included both `--san @IP_address` and `--san IP_address` flags.

## Conclusion

In this tutorial, you’ve built a VPN server that uses the IKEv2 protocol. You learned about the directives that control the `left` and `right` sides of a connection on both server and clients. You also configured a Windows, macOS, iOS, Android, or Linux client to connect to the VPN.

To add or remove users, skip to Step 5 again. Each line in `/etc/ipsec.secrets` is for one user, so adding or removing users, or changing passwords just requires editing the file.

Now you can be assured that your online activities will remain secure wherever you go and with any device that you use to access the internet.
