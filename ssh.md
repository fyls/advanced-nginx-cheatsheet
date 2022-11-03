# Ssh commands
## generate key
ssh-keygen -t ed25519 -C "your@email.com"
ssh-keygen -t rsa -b 4096 -C "your@email.com"


## login and move public key
```bash
ssh -i mykeyfile user@remotehost.com
cat ~/.ssh/id_rsa.pub | ssh USER@HOST "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys" - https://askubuntu.com/a/262074
ssh-copy-id user@host - https://askubuntu.com/a/46427
#Exit Dead SSH Sessions
#To kill an unresponsive SSH session, hit, subsequently.
Enter, ~, .
```
## tonneling 
```bash
ssh -L remotePort : 127.0.0.1:localPort -i ~/.ssh/secretKey root@remoteIp
#Use SSH forwarding

ssh username@jumphost -N -f -L localport:targethost:targetport
#The following command establishes an SSH tunnel between my local machine (@port 3307) and an RDS database (@port 3306), via an EC2 jump host (18.11.11.11).

ssh ec2-user@18.11.11.11 -N -f -L 3307:marcotestme.12345.eu-central-1.rds.amazonaws.com:3306
#You could now, for example, use the mysql client to connect to localhost:3307, which will be transparently tunneled to RDS for you.

mysql -h localhost -P 3307
```
## SCP
```bash
#To upload a file to a remote server:
scp myfile.txt user@dest:/path
#To recursively upload a local folder to a remote server:
scp -rp sourcedirectory user@dest:/path
#To download a file from a remote server:
scp user@dest:/path/myfile.txt localpath
#To recursively download a local folder to a remote server:
scp -rp user@dest:/remotedir localpath
```
# SSH_Config
```bash
Create a file ~/.ssh/config to manage your SSH hosts. Example:

Host dev-meta*
    User ec2-user
    IdentityFile ~/.ssh/johnsnow.pem

Host dev-meta-facebook
    Hostname 192.168.178.1

Host dev-meta-whatsapp
    Hostname 192.168.178.2

Host api.google.com
    User googleUser
    IdentityFile ~/.ssh/targaryen.key
    
#different keys for projects
Host github-work.com
    Hostname github.com
    IdentityFile ~/.ssh/id_work

Host github-personal.com
    Hostname github.com
    IdentityFile ~/.ssh/id_personal

#Then instead of cloning from github.com.
#git clone git@github.com:marcobehlerjetbrains/buildpipelines.git
#Clone from either github-work.com or github-personal.com.
#git clone git@github-work.com:marcobehlerjetbrains/buildpipelines.git
#
```

