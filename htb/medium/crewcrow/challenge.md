# CrewCrow - Sherlock - DFIR

We are given a dump of the desktop used containing 9474 files.
## Task 1 - Identify the conferencing application used by CrewCrow members for their communications.
By checking the desktop folder there is a terms and conditions file which declares to use Zoom.

## Task 2 - Determine the last time Nefarious used the conferencing application.
A prefetch file is made for ZOOM which contains the info. I used WinPrefetchView to find this last used date.

## Task 3 - Where is the conferencing application's data stored?
This is usually stored in AppData so I have a look there. Found it at C:\Users\Nefarious\AppData\Roaming\Zoom\data

## Task 4 - Which Windows data protection service is used to secure the conferencing application's database files?
Just google to find out DPAPI

## Task 5 - Determine the sign-in option used by Nefarious.
Determine sign in options used by Nefarious.
To do this I loaded the SOFTWARE registry hives along with the log files associated with it in registry explorer. I navigated to SOFTWARE/Microsoft/Windows/CurrentVersion/Authentication/LogonUI which provides a LastLoggedOnProvider field. This provides a GUID which I found out to be a regular username password login.
<img width="1183" height="410" alt="image" src="https://github.com/user-attachments/assets/0d16b2c1-944b-4264-acf7-04a45e5c9886" />

## Task 6 - Retrieve the password used by Nefarious
To get the password used by Nefarious I used the SAM hive along with the SYSTEM hive using mimikatz.
```
mimikatz # lsadump::sam /system:C:\Users\[]\Desktop\HTB\Sherlocks\CrewCrow\C\Windows\System32\config\SYSTEM /sam:C:\Users\[]\Desktop\HTB\Sherlocks\CrewCrow\C\Windows\System32\config\SAM
...
RID  : 000003e9 (1001)
User : Nefarious
  Hash NTLM: 42703fb3aeb2716687c641c665d26b3c
```
This is the NTLM hash that needs to be cracked.
I used crackstation to find out the password which was ```ohsonefarious92```
<img width="1525" height="563" alt="image" src="https://github.com/user-attachments/assets/b2e94fe2-185d-45b9-9676-a371ef5e94cc" />

## Task 7 & 8 - Find the key derivation function iterations used in the encryption process of the conferencing application's database. & Find the key derivation function page size used in the encryption process.
After some research found out that it used 1024 page size and 4000 iterations

## Task 9 - Identify Nefarious email address.
Now I have to decrypt the database and access its contents.
In Zoom.us.ini a key is provided. The base64 key is ```AQAAANCMnd8BFdERjHoAwE/Cl+sBAAAANKu7KG7QckOmM9kk+ ....``` ignoring ZWOSKEY
<img width="1268" height="239" alt="image" src="https://github.com/user-attachments/assets/8b991042-4056-4626-b805-054c73eae76d" />
I then decoded this using cyberchef and saved it to a file. Then I decrypted the masterkey using this command and the password supplied.
```
mimikatz # dpapi::masterkey /in:C:\Users\[]\Desktop\HTB\Sherlocks\CrewCrow\C\Users\Nefarious\AppData\Roaming\Microsoft\Protect\S-1-5-21-3675116117-3467334887-929386110-1001\28bbab34-d06e-4372-a633-d924fbab301b /system:C:\Users\[]\Desktop\HTB\Sherlocks\CrewCrow\C\Windows\System32\config\SYSTEM /password:ohsonefarious92
```
This result is then saved in mimikatz so I can then unprotect the decoded key.
Then I ran 
```
mimikatz # dpapi::blob /in:C:\Users\ar\Desktop\decodedkey.txt /user:"C:\Users\ar\Desktop\HTB\Sherlocks\CrewCrow\C\Users\Nefarious\NTUSER.DAT" /system:"C:\Users\ar\Desktop\HTB\Sherlocks\CrewCrow\C\Windows\System32\config\SYSTEM" /unprotect
...
data: 57 32 6b 2b 30 32 47 7a 42 56 65 5a 4b 4a 68 58 73 6e 52 49 71 4e 72 74 72 57 56 55 42 41 76 73 30 67 4c 4e 65 35 32 7a 58 4b 77 3d
```
Here is the data in hex format which is then converted from hex into a UTF-8 encoded string which becomes ```W2k+02GzBVeZKJhXsnRIqNrtrWVUBAvs0gLNe52zXKw=```. Now I can use this key to enter into DB Browser SQLCipher with these settings.
<img width="1045" height="668" alt="image" src="https://github.com/user-attachments/assets/46f4a3eb-2db6-494a-93b7-af552d8f1182" />
Now I can view the database!
<img width="1259" height="858" alt="image" src="https://github.com/user-attachments/assets/604e5cbd-4b3a-44e2-ba4a-b09052164aa0" />

Now scrolling through the database in "Browse Data" in DB Browser under "zoom_user_account_enc" I found an entry with nefarious' email. 
<img width="1265" height="240" alt="image" src="https://github.com/user-attachments/assets/1b3263b7-cecc-4539-823b-a17380ffe681" />

## Task 10 - What is the Meeting ID?
Now I navigated to zoom_kv table and found a meeting Id under the key ```com.zoom.client.saved.meetingid.[...]``` its value was ```RNpZaXfokRphhecoO6sHn9U02wtiPGaxi8UuhoAMGM2MEe175kZQQ2d7/Bk6WjUc4bz5EFCFpvrwYy/KTd56mA==```. I used this tutorial https://github.com/danjethh/extract_zoom_PMI_from_Forensic_disk_image/blob/main/Zoom_Personal_Meeting_ID.pdf to help me decode the meetingId.

Here is the script I made.
```python
from base64 import b64decode
from Crypto.Cipher import AES
from Crypto.Hash import SHA256

sid = b'S-1-5-21-3675116117-3467334887-929386110-1001'

key = SHA256.new(sid).digest()
iv = SHA256.new(key).digest()[:16]

raw_data = b64decode(b'RNpZaXfokRphhecoO6sHn9U02wtiPGaxi8UuhoAMGM2MEe175kZQQ2d7/Bk6WjUc4bz5EFCFpvrwYy/KTd56mA==')
cipher = AES.new(key, AES.MODE_CBC, iv)
plaintext = cipher.decrypt(raw_data)
print(plaintext.decode('utf-8'))

```
And the output was ```86233834426|Nefarious Leet's Zoom Meeting;100000``` where the first number is the meetingId.
## Task 11 - Retrieve the password used to encrypt the plan PDF file from the meeting chat.
This is revealed in "zoom_conf_chat_gen2_enc" during one of the messages.

## Task 12 - Discover the location from which the upcoming cyber-attack will be launched.
Using the password gained I can now view the pdf which contains the launch location: Eastern Europe 
