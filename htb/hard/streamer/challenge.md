# Streamer - Hard - DFIR
https://app.hackthebox.com/sherlocks/Streamer<br>
We are given a dump of the Windows System.

<img width="825" height="422" alt="image" src="https://github.com/user-attachments/assets/147d166a-8a8a-403b-8f86-be869f037f80" />
I also added this file location to autopsy to run in the background while I am doing most of the tasks, as it does take a while to injest everything.

## Task 1 - What's the original name of the malicious zip file which the user downloaded thinking it was a legit copy of the software?
Firstly, I used MFTECmd which converts the Master File Table to an easily readable csv format and opened that up into Timeline Explorer as this records all file changes on the system.
<img width="1517" height="366" alt="image" src="https://github.com/user-attachments/assets/cb0fb42b-18ca-4b7d-8f1d-ee1df0675669" />
I searched for ".zip" to narrow it down and found Obs Streaming Software.zip and its Zone Identifier which contains additional information about this download.
<img width="1911" height="513" alt="image" src="https://github.com/user-attachments/assets/3a6b2b67-9a74-401a-9a12-99183c622535" />
If I scroll across to a few columns down it has the original download name of ```OBS-Studio-28.1.2-Full-Installer-x64.zip```.
<img width="1339" height="514" alt="image" src="https://github.com/user-attachments/assets/d29858f8-c540-4297-8b5a-d3a3e041efa2" />

## Task 2 - Simon Stark renamed the downloaded zip file to something else. What's the renamed Name of the file alongside the full path?
The renamed file is Obs Streaming Software.zip so its this but with the full path: ```C:\Users\Simon.stark\Documents\Streaming Software\Obs Streaming Software.zip```

## Task 3 - What's the timestamp when the file was renamed?
This timestamp is on Last Record Change column for the same zip file: ```2023-05-05 10:22:23```.

## Task 4 - What's the Full URL from where the software was downloaded?
Using the previous Zone.Identifier and the contents table it shows the referer URL: ```http://obsproicet.net/download/v28_23/OBS-Studio-28.1.2-Full-Installer-x64.zip```, which is the full download path.

## Task 5 - Dig down deeper and find the IP Address on which the malicious domain was being hosted.
There is a firewall log named pfirewall.log that is close to a csv format so I used CyberChef to convert it to a csv by removing the first few lines and replacing spaces with commas.
<img width="1226" height="864" alt="image" src="https://github.com/user-attachments/assets/a9a2bfc0-ec4c-4935-98d7-5123dcf324b7" />
Once I had the csv I again used Timeline Explorer. However, I could not find a time that is close to the download timestamp of: ```2023-05-05 10:19:46``` as found in the MFT. As the first entry in pfirewall.log at 2023-05-05 begins at 14:59:53. I realised that the time must be different as there is a small hint at the top of the log file where the time format is set to local. MFT and evtx use UTC time so this is why there is difference. Now to find the local time zone of Simon Stark.

This is stored in the SYSTEM registry hive so I loaded that up using Registry Explorer and navigated to ROOT\ControlSet001\Control\TimeZoneInformation.
<img width="1919" height="556" alt="image" src="https://github.com/user-attachments/assets/91b83394-1c47-4586-88c7-6f6c1bb1ea86" />
Here it shows that the victim PC uses Pakistan Standard Time which is UTC+5. Now the time should be just before 15:19:46, so I set this in Timeline Explorer to filter out any results after this. It is also most likely using TCP so I filter that too as well as dst-port to port 80 as it is HTTP only.
Now I have only 51 lines from 24,237 and as this isn't relatively a big file it would be the closest to 15:19:46. 
<img width="1917" height="1006" alt="image" src="https://github.com/user-attachments/assets/b1f4b256-784e-4605-b4d9-3154d0a0e353" />
Here, I have found the IP address of that website: ```13.232.96.186```.

## Task 6 - Multiple Source ports connected to communicate and download the malicious file from the malicious website. Answer the highest source port number from which the machine connected to the malicious website.
Using Timeline Explorer, I have set the time filter too low so it is missing one result. However, this can just be removed and since I already know the destination IP it only added one extra row which was the highest source port of 50045

## Task 7 - The zip file had a malicious setup file in it which would install a piece of malware and a legit instance of OBS studio software so the user has no idea they got compromised. Find the hash of the setup file.
Back to the MFT csv I found the line where Obs Streaming Software.zip was at and set the filter to that parent path and filtered the extension to .exe or .msi as this is an installer. This gave 13 results with the malicious file being OBS-Studio-28.1.2-Full-Installer-x64.exe. 
<img width="1914" height="1003" alt="image" src="https://github.com/user-attachments/assets/85305ed4-3b3b-46d1-9dd9-1520b2188564" />
I then opened Amcache.hve found at C:\Windows\AppCompat\Programs with its LOG files too. This contains a SHA1 hash of the file under FileId but with the leading zeros removed.
<img width="1712" height="914" alt="image" src="https://github.com/user-attachments/assets/b2e0160b-288d-494a-895a-a01f769c234f" />

## Task 8 - The malicious software automatically installed a backdoor on the victim's workstation. What's the name and filepath of the backdoor?
From the prefetch viewer the malicious exe install was last run at 05/05/2023 11:23:14. So if I look at programs ran after this it should tell me what the backdoor is. There is one program that has an unusually long and jumbled name.
<img width="1715" height="610" alt="image" src="https://github.com/user-attachments/assets/9772bfae-0bba-4442-bdb4-e0b86a0e6c22" />
Here is the malicious process so the path is: ```C:\Users\Simon.stark\Miloyeki ker konoyogi\lat takewode libigax weloj jihi quimodo datex dob cijoyi mawiropo.exe```

## Task 9 - Find the prefetch hash of the backdoor.
This is the hash of the path of where the program was run and is stored in the last part of the .pf name so for this file it is: ```D8A6D943```.

## Task 10 - The backdoor is also used as a persistence mechanism in a stealthy manner to blend in the environment. What's the name used for persistence mechanism to make it look legit?
For this one, I loaded up Registry Explorer with the SOFTWARE hive along with its LOG files and used the find feature to search through all the records to find if ```lat takewode... .exe``` was mentioned anywhere.
<img width="1134" height="625" alt="image" src="https://github.com/user-attachments/assets/36a2cbb7-c22c-42c5-8434-f452e41a4b43" />
It found an entry at ```Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tasks\{07ED6739-D67B-4769-AF25-CBCFCE1226BF}``` which shows that this program was added to the task scheduler. This used \COMSurrogate to disguise itself as a legitimate process. 

## Task 11 - What's the bogus/invalid randomly named domain which the malware tried to reach?
Looking back at Task 8 at the prefetch viewer, that malicious application was last run at 11:23:21 so I look close to that time in the DNS Client Operational logs. At 11:23:23 a DNS query is made to ```oaueeewy3pdy31g3kpqorpc4e.qopgwwytep```.
<img width="1330" height="766" alt="image" src="https://github.com/user-attachments/assets/4a4b0372-9590-4e9c-9080-4708ac352822" />

## Task 12 - The malware tried exfiltrating the data to a s3 bucket. What's the url of s3 bucket?
This would be on the same DNS logs so I searched for s3 close to that time and found ```bbuseruploads.s3.amazonaws.com```.
<img width="1331" height="765" alt="image" src="https://github.com/user-attachments/assets/ca395272-291a-492c-a0b2-8180cf9eab1a" />

## Task 13 - What topic was simon going to stream about in week 1? Find a note or something similar and recover its content to answer the question.
For this one I can search for .txt files in the MFT table using the output from MFTEcmd earlier, here I found Week 1 Plan.txt in its records. However, this file isn't present in the dump. I used MFTEcmd to recover this file as it is a "resident" file which means its contents are stored in MFT. I can extract this using the dr flag with MFTEcmd. This generated a Resident folder containing this raw data
<img width="1594" height="355" alt="image" src="https://github.com/user-attachments/assets/3dec7fae-1f9d-48b8-9204-bf6b305d03b1" />
In this folder I found Week 1 plan.txt.bin which contains the topic.
<img width="784" height="443" alt="image" src="https://github.com/user-attachments/assets/e8f8db4d-87be-4cd2-869f-33a86b60777b" />



## Task 14 - What's the name of Security Analyst who triaged the infected workstation?
From the users on the system, CyberJunkie stood out as under AppData it had a ConnectedDevicesPlatform folder which allows other devices to be connected. Within this, there is an ActivitiesCache.db which contains actions that the user has completed. This shows a setup.exe being run suggesting it was for triaging.
<img width="1714" height="826" alt="image" src="https://github.com/user-attachments/assets/0254d659-cf00-47f0-9b44-a173447d50d6" />

## Task 15 - What's the network path from where acquisition tools were run?
To find this, I searched through NTUSER.DAT under Simon.stark using Registry Explorer's find feature with the string ```\\``` as this signifies a UNC path. Under the third result it found ```\\DESKTOP-887GK2L\Users\CyberJunkie\Desktop\Forela-Triage-Workstation\Acquisiton and Triage tools\KAPE\gkape.exe``` which was the full execution path. This is correct as CyberJunkie is running KAPE which is a well-known triage program. 
<img width="1718" height="619" alt="image" src="https://github.com/user-attachments/assets/836562ed-67ca-406c-bb4b-48ce9e1c4db8" />
