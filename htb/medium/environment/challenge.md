# Environment - Linux
https://app.hackthebox.com/machines/Environment <br>

Ran an NMAP scan first
![image](https://github.com/user-attachments/assets/a2bed107-3f7f-4fc4-bc09-c37eb80ad23c)
Added environment.htb to the /etc/hosts
Used gobuster to dicover directories.

![image](https://github.com/user-attachments/assets/07ca7723-a257-48d9-91ce-2c33c7156a3b)
403 means access denied but it is available for other users.

## /mailing route
![image](https://github.com/user-attachments/assets/a4f1ad33-be2a-49b5-badf-0fa59db18a42)

This shows some of the code in index.php and tells us what version of PHP and Laravel.
PHP: 8.2.28
Laravel: 11.30.0

# /login route
This has an email, password and _token parameter. As before it shows errors in the code, removing password does this again.
![image](https://github.com/user-attachments/assets/9a55d6b8-77db-44eb-99e5-6acdaf47e43d)

On this error page we can also see the sqlite database query (select * from "sessions" where "id" = 'f04aS5JFf71XGo1YiQ2i27Jrr7YMytLpgRsfp3EZ' limit 1). 
![image](https://github.com/user-attachments/assets/ea3ee0e2-9b37-4c25-ba47-387bb061d68e)

So this website gets the session id assuming it returns the username.

Next, I changed the remember me paramter to 'Falsse' to trigger an error on line 75 to see more of the code. The comment is //QOL: login directly as me in dev/local/preprod envs
![image](https://github.com/user-attachments/assets/281a3b81-c24e-46d2-952f-0ab8c7831e86)

## Vulnerability 
I found CVE-2024-52301 which allows you to change environment variables. This is useful as if the environment can be changed to "preprod" I can gain access to the management dashboard. 
By including ?--env=preprod in the POST I am able to override the environment variable.
![image](https://github.com/user-attachments/assets/600cc826-cbb3-45dd-829d-fd620b3cfb98)

![image](https://github.com/user-attachments/assets/a4f1edae-d4b1-40f0-b63d-22fe1f791d9b)

Under profile I am able to upload a new profile picture.
![image](https://github.com/user-attachments/assets/370acaeb-faf1-4f89-8e9b-f2a658397ce8)

This API response shows file location it is sent to.
There must be a magic bytes header present at the beginning of the file.
Managed to upload php file with the png header to achieve command exeuction
![image](https://github.com/user-attachments/assets/ba2d71fa-d47a-48f7-9f6a-d51b4cc19166)

Now I have got the user flag!
![image](https://github.com/user-attachments/assets/5d709d58-e45c-4e64-839e-6cf98fe5565c)
Tried to execute reverse shell, not working.

Hish has a keyvault.gpg key maybe I can copy it into the the storage/files directory by using 
```cp /home/hish/backup/keyvault.gpg /var/www/app/storage/app/public/files``` but url encoded.
Now I can download it.

Found /home/hish/.gnupg
```
drwxr-xr-x 2 hish hish 4096 Jun 19 02:35 openpgp-revocs.d
drwxr-xr-x 2 hish hish 4096 Jun 19 02:35 private-keys-v1.d
-rwxr-xr-x 1 hish hish 1446 Jan 12 03:13 pubring.kbx
-rwxr-xr-x 1 hish hish   32 Jan 12 03:11 pubring.kbx~
-rwxr-xr-x 1 hish hish  600 Jan 12 11:48 random_seed
-rwxr-xr-x 1 hish hish 1280 Jan 12 11:48 trustdb.gpg
```
Managed to copy this to a temp folder and performed --list-keys on it. ```gpg --homedir /tmp/gpg --list-secret-keys```
/![image](https://github.com/user-attachments/assets/4cddebe9-ccfd-430a-8575-27eba0b1308f)

Used this command ```gpg --homedir /tmp/gpg --output /tmp/decrypted.txt --decrypt /home/hish/backup/keyvault.gpg``` to decrypt this.
Now I can view the passwords
![image](https://github.com/user-attachments/assets/e9830240-2ef8-4778-9946-580a5107a6c2)
```
PAYPAL.COM -> Ihaves0meMon$yhere123
ENVIRONMENT.HTB -> marineSPm@ster!!
FACEBOOK.COM -> summerSunnyB3ACH!!
```
Now, I use environment.htb password with hish to get ssh.
Checked sudo -l and user can run systeminfo which is just a bash script running different commands.
I checked ```find / -perm -4000 2>/dev/null``` for suid binaries and there was an interesting one at /tmp/rootbash
I then executed this /tmp/rootbash -p which gave me root.

