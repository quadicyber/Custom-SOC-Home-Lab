# Custom-SOC-Home-Lab
In this project, I build my own custom SOC Home Lab, with both on-prem and cloud components, completely for free and with an automation component!

_DISCLAIMER: This is **NOT** a tutorial!_

## *Software Used* 
1. Wazuh - SIEM and XDR
2. The Hive - Case Management 
3. Shuffle - SOAR Capabilities

I have previous Experience working in Microsoft Azure utilizing the Sentinel SIEM, but I wanted more experience and opportunity to learn how real SOCs operate by creating my own custom one.

This project is broken down into 5 Phases, each of which containing distinctive actions that build upon one another.

## *Phase One -- Design*
The goal is to go from nothing at all to a full blown Wazuh instance, complete with Case Management from The Hive, and SOAR capabilities from Shuffle.

First, I mapped out the SOC so that I can know beforehand all required pieces to the puzzle and what role each of them will play within the environment. 

![image](https://github.com/quadicyber/Custom-SOC-Home-Lab/assets/104177919/1f3edc47-0fdc-4352-93ba-437a8647dc18)

### -- LOGICAL WORKFLOW BREAKDOWN -- 
1. I have Windows 10 Wazuh Agent that is generating and sending events through a router to the Internet.
2. From there the Wazuh Manager receives these events.
3. If the Wazuh Manager recieves events that generate an alert, that alert is sent to Shuffle.
4. When Shuffle recieves the alert, it will perform some OSINT gathering to enrich the IOC's through a connection to the Internet. 
5. After the IOC's are enriched, Shuffle will send an alert over to The Hive. 
6. Shuffle will also need to send an email to the SOC analyst
7. The SOC Analyst will send and receive the email. The email itself should contain details about the alert, and it should ask the question: "Do you want to contain/perform a certain action? Yes or No?" Depending on the response, the action should then be sent to Shuffle, which will be sent to Wazuh, and then the Windows 10 Client. 
8. Send the SOC Analyst's response action to Shuffle, and then to the Wazuh Manager.
9. Wazuh Manager instructs the Wazuh Agent to perform the response action.

## *Phase Two -- Installations*
The objective of this phase is to install applications and virtual machines. By the end of this phase, there will be:

- 1 Windows 10 Machine with Sysmom 
- 1 Wazuh Server w/ Ubuntu 22.04 (cloud)
- 1 The Hive Server w/ Ubuntu 22.04 (cloud)

I installed Oracle VM VirtualBox on my PC, and installed a Windows 10 Pro ISO on it. I used a Powershell script in order to install the Wazuh Agent on the VM. 

![Powershell to Install WA](https://github.com/quadicyber/Custom-SOC-Home-Lab/assets/104177919/f0b85e61-c659-4157-9b56-926d7130bb8f)
*NOTE: The latest version of Sysmon was downloaded from Microsoft, and the configuration I used is* [here](https://github.com/olafhartong/sysmon-modular/blob/master/sysmonconfig.xml).

I chose Digital Ocean as my cloud provider, due to getting a free $200 credit for the first 60 days of use. The resources I used will not come anywhere near using $200 worth so this is a cost-effective measure.

Within Digital Ocean, I created a cloud Ubuntu 22.04 instance (for compatibility with Wazuh Server), used PuTTY to SSH into it, updated it, and then installed Wazuh. 

![SSH Into Wazuh](https://github.com/quadicyber/Custom-SOC-Home-Lab/assets/104177919/6a46423e-b665-4870-8a16-b36c21cb492d)

I then created another Ubuntu 22.04 instance to install The Hive on, which again will be used to handle Case Management. I then followed similar steps to SSH into The Hive VM to update Ubuntu and install it. After installing The Hive and all of its dependences, I moved onto the next phase, Configurations. 

## *Phase Three -- Configurations*
It is now time to configure The Wazuh and the The Hive server to get them up and running correctly. By the end of this phase, both servers will be fully functional, along with the Windows 10 Agent reporting to Wazuh!

Cassandra is a dependency used by The Hive for database purposes, and it's configuration files must be edited. The listening address and rpc_address was changed to the IP of The Hive as given in Digital Ocean. Finally, the address of the seed_provider was changed from the local host to the IP of The Hive. 

![Editing Cassandra](https://github.com/quadicyber/Custom-SOC-Home-Lab/assets/104177919/131f2bb5-c608-4cc3-97d3-165f69502a92)

Next, Elastic Search is configured, which is used to manage data indices or query data. Notably, the discovery seed is what would be configured if I needed to scale out, but since this is a lab it was left alone. 

Now it is time to configure The Hive itself, but before doing so it is necessary to make sure that The Hive user and group has access to a certain filepath. The "chown" command was used to change ownership of The Hive user and The Hive group over from the root to the destination directories.

![Changing Hive Ownership](https://github.com/quadicyber/Custom-SOC-Home-Lab/assets/104177919/ba39607d-1753-4ccb-b27d-4e18bafff91d)

After this, The Hive config is accessed and set up, and now I can access The Hive!

![The Hive Dashboard](https://github.com/quadicyber/Custom-SOC-Home-Lab/assets/104177919/401d649f-66db-4b08-8bb9-babd6393370a)

It is now time to configure the Wazuh Dashboard, by adding the Windows 10 VM as an agent. A Powershell command (provided by the Wazuh Manager) was run in the Windows VM to download and install the agent on it. Now within the Wazuh Dashboard I have one active agent as expected, and only a few minutes after adding the agent over 700 events have been logged!

![Security Events Minutes After](https://github.com/quadicyber/Custom-SOC-Home-Lab/assets/104177919/b16587a7-b584-40ff-9351-c95d9dd2c136)

In the next phase, I will begin to generate telemetry and create alerts.

## *Phase Four -- Telemetry*
In this phase, the goal is to generate telemetry containing Mimikatz from the Wazuh Agent Machine, and ingest it into the Wazuh Manager/Dashboard, triggering a Mimikatz custom alert.

After logging into the Wazuh Agent Machine, I went into the configuration file and added event monitoring for Sysmon that was installed in Phase 2. 
Next, I downloaded Mimikatz onto the Wazuh Agent Machine. For this step, the Downloads folder path was excluded from Windows Defender since it will detect it and block it. 

Some configurations were edited within the Wazuh Manager, and on the Wazuh Dashboard a new index pattern was created for archives, so that all logs can be searched irregardless of if Wazuh triggered an alert. This is because by default Wazuh does not show all logs.

Within The Wazuh Agent VM, I went into the event viewer to confirm that Sysmon was logging Mimikatz events by filtering the logs with Event ID '1'. 
This is also confirmed in the Wazuh Manager CLI using the command *cat archives.json | grep -i mimikatz*, where every instance is highlighted in **red text**. Within the Wazuh Dashboard, 2 Mimikatz 'hits' are now showing. 

![Mimikatz CLI](https://github.com/quadicyber/Custom-SOC-Home-Lab/assets/104177919/b7ec7c6b-ae37-4ea6-af91-0fe5201956f2)

![Mimikatz Sysmon Confirmation](https://github.com/quadicyber/Custom-SOC-Home-Lab/assets/104177919/b26aac00-c1b0-4087-97e3-a04facf611c5)
*NOTE: Mimikatz is an application that attackers/red-teamers use to extract credentials.* 

The *data.win.eventdata.originalFileName* is used to create the alert because an attacker could just rename Mimikatz to something else and bypass the alert. Just for fun, I set the alert level to 15, the highest possible level. MITRE ID T1003 is used because it is for credential dumping, which is what Mimikatz is known for.

![Custom Mimikatz Alert](https://github.com/quadicyber/Custom-SOC-Home-Lab/assets/104177919/5217e7da-6cb0-4cd6-86cf-055086f1fc95)

After exiting and restarting Mimikatz in the Wazuh Agent VM, my custom alert is triggered on the Wazuh Dashboard!

![Mimikatz Usage Detected!](https://github.com/quadicyber/Custom-SOC-Home-Lab/assets/104177919/cfe300c7-b47c-4562-82af-af85ee5a1a5e)

## *Phase 5 -- Automation*
In the final phase of this project, the SOAR (The Shuffle) will be connected to The Hive and to the SOC Analyst (which is me in this instance) to send alerts. By then end of this 
phase, the SOC will be fully operational. 

Here is an overview of the Workflow that I will be using for this project. The same logic could be applied to other use cases as well.
1. Mimikatz alert sent to Shuffle
2. Shuffle Receives Mimikatz Alert
   - Extracts SHA256 Hash From File
3. Check Reputation Score with VirusTotal
4. Send Details to TheHive To Create Alert
5. Send Email to SOC Analyst to Begin Investigation

![Final Workflow](https://github.com/quadicyber/Custom-SOC-Home-Lab/assets/104177919/4444bdce-d4c6-4404-bc8e-5d0881181da7)

Steps 1 through 3 are all completed in Shuffle. I started by logging into Shuffle and creating a new workflow, adding triggers to it, and instructed the Wazuh Manager to send events over to Shuffle. After this, I was able to view all of the alert info that Wazuh generated within Shuffle, and I created all of the details that would sent to The Hive in step 4. 

![Wazuh Event Info in Shuffle](https://github.com/quadicyber/Custom-SOC-Home-Lab/assets/104177919/2c59e083-fc5a-4ecb-8e10-78e75e76b902)

In order to test the automation, I temporarily opened all IPv4 traffic to port 9000 on my servers. I then went in and refreshed the workflow, and upon doing so my alert appeared within The Hive on a user account I created that simulates what an SOC Analyst would see. 

![The Hive Case Detailed](https://github.com/quadicyber/Custom-SOC-Home-Lab/assets/104177919/22056dd0-f50b-4fed-8fed-18eedbe43a44)

After this point, due to technical difficulties implementing the email function of The Shuffle that are outside of my control, at this point I was unfortunately unable to implement the email/response function of my SOC.

Even so, the SOC Analyst could still log into the Wazuh Manager manually and send the response commands to it so that the Mimikatz threat on the Wazuh Agent Machine is eradicated. 

I learned a **lot** during this project, particularly during the configuration phase as I got to see a lot of the logic and rules necessary to keep an SOC running behind the scenes. With this project I can now practice my incident response and remediation skills and techniques, so this has been a great investment of my time and effort. 

