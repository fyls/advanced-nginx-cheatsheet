# Ssh commands
## generate key
ssh-keygen -t ed25519 -C "your@email.com"
ssh-keygen -t rsa -b 4096 -C "your@email.com"


## login and move public key
ssh -i mykeyfile user@remotehost.com
cat ~/.ssh/id_rsa.pub | ssh USER@HOST "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys" - https://askubuntu.com/a/262074
ssh-copy-id user@host - https://askubuntu.com/a/46427

## tonneling 
ssh -L remotePort : 127.0.0.1:localPort -i ~/.ssh/secretKey root@remoteIp

## SCP
'''bash
#To upload a file to a remote server:
scp myfile.txt user@dest:/path
#To recursively upload a local folder to a remote server:
scp -rp sourcedirectory user@dest:/path
#To download a file from a remote server:
scp user@dest:/path/myfile.txt localpath
#To recursively download a local folder to a remote server:
scp -rp user@dest:/remotedir localpath
'''
