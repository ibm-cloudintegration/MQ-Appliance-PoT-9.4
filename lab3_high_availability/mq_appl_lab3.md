# Lab 3 - IBM MQ Appliance High Availability

In this lab, you will configure two virtual appliances for high availability (HA) and test that HA works as expected.

VMs required:

* **Windows**
* **MQAppl1**
* **MQAppl2**

The lab environment consists of two virtual appliances (MQAppl1 and
MQAppl2) and a Windows environment to perform console operations and
testing. There are other virtual appliances (MQAppl3, MQAppl4, MQAppl5, and MQAppl7)
that will not be used in this lab. You can just ignore them for now. 

The initial setup configuration from Lab 1 has already been applied to MQAppl1 and MQAppl2. This includes pre-defining the IP addresses. The tables below outline the network configuration for each appliance. It is essential to review and understand this setup, as it will be referenced throughout the lab.

### MQAppl1 Addresses

|Virtual Adapter Network Name | Appliance Ethernet Interface | Usage | DHCP | IP Address | Gateway Addresses |
|:---------------------------:|:----------------------------:|:-----:|:----:|:----------:|:-----------------:|
| Network 1 | eth0 | Management and client traffic | No | 10.0.0.11 | 10.0.0.1 |
| HA1 | eth1 | HA primary connection | No | 10.0.1.11 | 10.0.1.1 |
| HA2 | eth2 | HA alternate connection | No | 10.0.2.11 | 10.0.2.1 |
| HArep | eth3 | HA replication connection | No | 10.0.3.11 | 10.0.3.1 |
| DRrep | eth4 | DR replication connection | No| 10.0.4.11 | 10.0.4.1 |

### MQAppl2 Addresses


|Virtual Adapter Network Name | Appliance Ethernet Interface | Usage | DHCP | IP Address | Gateway Addresses |
|:---------------------------:|:----------------------------:|:-----:|:----:|:----------:|:-----------------:|
| Network 1 | eth0 | Management and client traffic | No | 10.0.0.12 | 10.0.0.1 |
| HA1 | eth1 | HA primary connection | No | 10.0.1.12 | 10.0.1.1 |
| HA2 | eth2 | HA alternate connection | No | 10.0.2.12 | 10.0.2.1 |
| HArep | eth3 | HA replication connection | No | 10.0.3.12 | 10.0.3.1 |
| DRrep | eth4 | DR replication connection | No| 10.0.4.12 | 10.0.4.1 |

## Start the environment

1. If either MQAppl1 or MQAppl2 are not started, please start these now.. 
    
    ![](./images/image9.png)

2. In the *Windows* VM, open two new PuTTY windows. One for MQAppl1 on 10.0.0.11, and a second for MQAppl2 on 10.0.0.12. Log in with user / password **admin** / **Passw0rd!**.
	
	![](./images/image11a.png)

	
### Sync the appliance clocks
 
> <span style="color: red">**Important:** <br>For this lab, the appliance clocks need to be synchronized. This should be automatic in your environment, but it is always best to verify prior to setting up the appliances. You should make sure the appliance times are in sync. Use the \"show clock\" to display the time. </span>
	![](./images/image10.png)

## The virtual environment

Prior to MQ 9.2.3, the normal process was to use the command line interface (CLI) to configure HA between the appliances. This is the mechanism used in this lab. 

As of MQ 9.2.3 you now have the option to use the MQ Console user interface (UI) to configure HA between the appliances. An outline of the process is documented at the end of this lab for your review. 

<a name="cli"></a>

> <span style="color: Blue"><Bold>Note:</Bold>Use PuTTY when you are instructed to open a command window to enter CLI commands on an appliance.</span>

### Virtual appliance MQAppl1 

1. The first virtual appliance you will look at is MQAppl1. Make sure you are at the appliance command line. If you are in the mqcli, type "**exit**". Execute the "**show ipaddress**" command. The IP addresses in use for this appliance are as follows:

    ![](./images/image12a.png)
    
1. You need to be at the *mqcli* command line to issue the mq commands. 
	Enter **mqcli**

	![](./images/image12b.png)      
    
### Virtual appliance MQAppl2

1. The second virtual appliance you will look at is MQAppl2. Make sure you are at the appliance command line. If you are in the mqcli, type "**exit**". Execute the "**show ipaddress**" command. The IP addresses in use for this appliance are as follows:

    ![](./images/image13a.png)
    
3. Enter **mqcli**

## Create the HA Group

You will now create the HA group on the two appliances. You should be at the mqcli command line, as shown below:

1. On **MQAppl2**, run the following command:

	```
	prepareha -s SomeSecret -a 10.0.1.11
	```

	![](./images/image15a.png)
	
   > <span style="color: Blue">**Note:** About *prepareha*:  
   This command prepares an appliance to be part of an HA group. You run it on the appliance that you do not run *crthagrp* on.
   <br> '*-a 10.0.1.11*' 
   Specifies the IP address on the HA group primary interface, on the other appliance in the group.
   <br>'*-s SomeSecret*'
   Specifies a string that is used to generate a short-lived password. The password is used to set up the unique key for the two appliances.
	
2. Now go to **MQAppl1** (do not wait for the prepareha command to
    complete) and issue the following command:

	```
	crthagrp -s SomeSecret -a 10.0.1.12
	```

    This may take a few minutes. At the completion of the command execution, you should see the following messages:

	![](./images/image16a.png)
	
	> **Note:** About *crthagrp*: 
	This command creates an HA group of two appliances. The prepareha command must be run on the other appliance before you run *crthagrp*. 
	<br> '*-a 10.0.1.12*'
	Specifies the IP address on the HA group primary interface, on the other appliance in the group.
	<br> '*-s SomeSecret*'
	Specifies a string that is used to generate a short-lived password in the preparha command. It is the unique key for the two appliances. 

	You are now ready to create queue managers and test the HA.

## Create queue managers 

1. On the *MQAppl1* appliance, issue the following command:

	```
	crtmqm -p 1511 -fs 2 -sx HAQM1
	```

	> **Note:** About (-fs) and (-sx) 
    <br>The -fs parameter specifies that the queue manager is created with the file system size fs. 
    <br>The default file system size is set to 64GB on a real appliance, but you set it to only 2GB on the virtual appliance. 
    <br>The -sx parameter specifies that the queue manager is a high availability (HA) queue manager. A queue manager is not configured for HA, and synchronized across the two appliances without specifying this flag. It is still possible to have non-HA queue managers on an appliance in an HA group.

2. After the queue manager has been created, you will see the HA
    configuration taking place as shown below. You should see the
    message indicating that the final HA configuration has succeeded.

	![](./images/image19a.png)

	> <span style="color: Blue">**Note:** <BR> About high availability status:  
    <br>You can view the status of a queue manager in a high availability (HA) group by using the status command on the command line.  
    <br>The status command returns information about the operational status of a specified queue manager in the HA group. The status can include the following information:
	the operational state of the HA group,
	the filesystem size and CPU used by the queue manager,
	the replication status of the queue manager (if synchronization is in progress), the preferred appliance for the queue manager,
	whether a partitioned situation is detected and if it has, the amount of 'out-of-sync' data held. 

3. You should now run the following command to check the status of the
    queue manager:

	```
	status HAQM1
	```

4. You should now see that the queue manager is running, HA is enabled
    and running normally, with "This appliance" as the preferred
    location.

	![](./images/image20a.png)

5. If you do not see output as above, try running the command again as
    synchronization may still be in progress.

6. Now, go to the **MQAppl2** appliance.

1. Run the status command on this appliance. On this appliance the queue manager will be reported as `Running elsewhere`. 

	```
	status HAQM1
	```

	![](./images/image20b.png)


7. On **MQAppl2** create another HA queue manager. Issue the following command:

	```
	crtmqm -p 1512 -fs 2 -sx HAQM2
	```

8. Again, you expect to see the successful creation of the queue
    manager and successful completion of the HA configuration.

	![](./images/image21a.png)
	

10. Staying on *MQAppl2*, run the status command for the HAQM2 queue
    manager. If you contrast this with the HAQM1 status results you see
    the following:

	```
	status HAQM2
	```

    ![](./images/image22b.png)

	You are now ready to start testing HA, but first you need to set up the
MQ Explorer.

<a name="mqexplorer"></a>
## Set up MQ Explorer	

We create a new messaging user called *ibmdemo* using the same process as in Lab 1. 

1. Enter the following command on both MQAppl1 and MQAppl2:
	
	```
	usercreate -u ibmdemo -p Passw0rd! -g mqm
	```
	
	![](./images/image22c.png)
	
	You now need to set up the **SYSTEM.ADMIN.SVRCONN** channel that the MQ Explorer uses for communication.

2. Go to the **MQAppl1** appliance and enter the following commands:

	```
	runmqsc HAQM1
	DEFINE CHANNEL(SYSTEM.ADMIN.SVRCONN) CHLTYPE(SVRCONN)
	SET CHLAUTH(SYSTEM.ADMIN.SVRCONN) TYPE(BLOCKUSER) USERLIST('*whatever')
	ALTER AUTHINFO('SYSTEM.DEFAULT.AUTHINFO.IDPWOS') AUTHTYPE(IDPWOS) ADOPTCTX(YES)
	REFRESH SECURITY TYPE(CONNAUTH)
	END
	```

	![](./images/image24a.png)

3. Go to the **MQAppl2** appliance and run the following commands:
	```
	runmqsc HAQM2
	DEFINE CHANNEL(SYSTEM.ADMIN.SVRCONN) CHLTYPE(SVRCONN)
	SET CHLAUTH(SYSTEM.ADMIN.SVRCONN) TYPE(BLOCKUSER) USERLIST('*whatever')
	ALTER AUTHINFO('SYSTEM.DEFAULT.AUTHINFO.IDPWOS') AUTHTYPE(IDPWOS) ADOPTCTX(YES)
	REFRESH SECURITY TYPE(CONNAUTH)
	END
	```

	![](./images/image24b.png)

	You are now ready to add the appliance HA queue managers to MQ Explorer.

6. Open **MQ Explorer** (from the desktop).

    ![](./images/image25.png)

7. Right-click the **Queue Managers** folder and select **Add Remote
    Queue Manager...**

	![](./images/image26a.png)

8. Enter the name of the MQAppl1 queue manager (**HAQM1**) and select
    **Connect directly**.

	![](./images/image27a.png)

9. Click **Next**.

10. Enter the IP address of the MQAppl1 appliance (**10.0.0.11**) and the
    port number of the listener (**1511**).

    ![](./images/image28a.png)

11. Click **Next** twice.

43. Select the check box next to **Enable user identification**.

44. Enter **ibmdemo** as the **Userid**.

45. Select the **Prompt for password** radio button. 

	Click **Finish**. 
	
	![](./images/image29a.png)
	
46. In the *Password details* popup, enter the password ("Passw0rd!") and then click **OK**. 

	![](./images/image29b.png)
	
49. The queue manager should now be visible in MQ Explorer.

	![](./images/image30d.png)

16. Repeat the steps above to add HAQM2 using the following details:
	-   Queue manager (**HAQM2**)
	-   IP address (**10.0.0.12**)
	-   Listener port (**1512**)
	-   Messaging user and password (**ibmdemo** / **Passw0rd!**)

17. You will now see the two MQ Appliance queue managers in the Queue
    Managers folder.

    ![](./images/image31b.png)

18. In the content pane, you will see that the queue managers are
    identified as Appliance queue managers.

	![](./images/image32b.png)

You are now ready to test the HA Failover.

## Test HA Failover

1. Go back to the PuTTY window for **MQAppl1**.

2. Ensure you are at the mqcli interface and issue the following command:

	```
	sethagrp -s
	```

	> **NOTES:** About *sethagrp -s*. 
 This command pauses and resumes an appliance in a high availability group. When you use the sethagrp command to pause (or suspend) an appliance that is part of a high availability group, any queue managers running on that appliance fail over to the other appliance in the group. 
                  
3. Run a **status HAQM1** command and note what it displays.

    ![](./images/image33a.png)

4. Go back to the MQ Explorer and add HAQM1, but this time use the
    following details to add it as if it was running on MQAppl2 rather
    than MQAppl1
	-   Queue manager (**HAQM1**]
	-   IP address (**_10.0.0.12_**)
	-   Listener port (**_1511_**)
	-   Messaging user and password (**ibmdemo** / **Passw0rd!**)

5. You will now see three queue managers listed, but both queue
    managers are now running on the MQAppl2 appliance.

    ![](./images/image34b.png)

6. The queue manager on MQAppl1 appliance shows as disconnected and
    does not show a status.

    ![](./images/image35b.png)

	Now resume the appliance from standby mode.

7. Go back to *MQAppl1* and enter the following command:

	```
	sethagrp -r
	```

8. Check the status of the HAQM1 queue manager and note what it
    displays.

    ![](./images/image36a.png)

9. If the queue manager is not yet showing as *Running*, it is still in
    the process of failing back. Run the status command again.

	Is it running on MQAppl1 again? If it is, you can proceed to the
    next step.

10. Go to the MQ Explorer and check that it is also showing the HAQM1 as
    running on the MQAppl1 appliance. Note: if not, you may have to
    reconnect to the queue manager (*right-click* on the queue manager and select **Connect**).

	Do the same failover test, but for the **HAQM2** queue manager on the
**MQAppl2** appliance.

11. Go back to the PuTTY window for MQAppl2 appliance, and run the sethagrp command to
    suspend (**sethagrp -s**) as before.

12. Check the status of the **HAQM2** queue manager to verify that the
    status indicates that it is running elsewhere.

13. Check in the MQ Explorer to verify that you can add (and then see)
    the HAQM2 running on the MQAppl1 (**10.0.0.11**) appliance.

    ![](./images/image38b.png)

14. Finally, go back to the MQAppl2 and use the **sethagrp -r** command
    to resume the appliance.

15. Verify that both queue managers are running on the appropriate
    appliance.

Finally, for this lab you need to process some messages in the HA
environment.

<a name="processmessages"></a>
## Process Messages

In this section, you will create queues and process messages to and from
these queues. You may use any combination of your favorite tools to do
any of this (MQ Explorer, RFHutil, any of the sample programs for MQ).
However, start with the Web console because it is a very useful
interface to the appliance queue managers.

1. Open a **Firefox** browser on Windows.

2. You want to have two tabs in the browser, one for the web console of each appliance: **MQAppl1** and **MQAppl2**.

3. If you receive any exception messages when opening the console URLs,
    add an exception and continue.

4. Select the first tab and log on to the console for *MQAppl1* (**user =
    admin / password = Passw0rd!**).

5. Click the **Manage** icon. In *Manage*, all queue managers defined on *MQAppl1* are displayed. HAQM1 should be *Running* and HAQM2 should be *Running elsewhere*. Click the **High availability** tab.

	![](./images/image140b.png)
	
1. First, notice the *High Availability* status of the *HA group* showing a checkmark to signify that the high availability status of the HA group is good -- that both appliances are up and running as part of the HA group and shown as *Online*. 

	![](./images/image141a.png)

	Also notice that each tile has an elipsis. If you click the elipsis for the local appliance **MQAppl1** or the partner appliance **MQAppl2**, you can suspend that appliance. By clicking the elipsis for the **HA group**, you can delete the group or regenerate SSH keys.

	The next interesting thing you see is the queue manager status in the bottom half of the window. The regular display of the queue managers is shown. You see both queue managers in a running and highly available state.

7. Click the hyperlink for queue manager **HAQM1**. You are taken to the *Overview* tab for the queue manager. Select the Queue objects tab for the queue manager. You now need to create the queue you will use for testing. Click **Create**. 

	![](./images/image142a.png)

13. Click the **Local** tile and **Next**. Name the queue **Q1** and leave the default object type set to local. Click **Create**.

    ![](./images/image143a.png)

14. You are returned to the list of queues and **Q1** is now in the list. This summary screen of queues shows the *Type* of queue, *Depth* percentage, and *Maximum depth* which shows the number of messages on the queue over the max depth. 

    ![](./images/image144a.png)
    
1. On the far right side of the display is an elipsis for each queue. Here you can view the messages, create messages, clear the queue, or view the configuration (properties) of the queue. Click the elipsis for **Q1** and select **View configuration**. 

	![](./images/image145a.png)

1. Click **Edit**. In edit mode there are two tabs - *Properties* and *Security* where you can maintain authority records for the queue. Stay on *Properties* tab. You may need to scroll down within the *General* properties to find *Default persistence*. Click the drop-down under *Default persistence* and set it to **Persistent**. Then click **Save**.

    ![](./images/image146a.png)
    
    Click the breadcrumb for **Queues** to return to list of queues.
    
    ![](./images/image148a.png)
    
1. You now need to put some messages into the Q1 queue. Use the MQ Explorer for this, because you can easily do it without having to perform any additional configuration. Open the MQ Explorer content pane for the HAQM queues.

18. Drill down into **HAQM1**, and then into **Queues**. Right-click **Q1** and select **Put Test Message...**.

    ![](./images/image147a.png)

19. Enter some test messages by entering some text and clicking **Put message**.

	![](./images/image149a.png)

20. Repeat this for as many messages as you wish to put on the queue (the lab test scenario has 13 messages).

21. When you are finished, click **Close**.

22. Go back to the Web console dashboard and refresh the page. 

23. You should now see that there are 5 messages on the queue (or as many messages as you put there).

    ![](./images/image150a.png)

	You now need to test HA to ensure that the messages fail over to the queue and queue manager on the other appliance.

24. Go to the *MQAppl1* appliance.

25. Suspend the appliance using the **sethagrp -s** command as before.

26. Use the **dspmq** command to verify that the HAQM1 is running elsewhere (as before, if this takes a little time, continue to run the command until the results are as shown below).

    ![](./images/image53a.png)

27. Go back to the browser, but this time log on to the Web console for the MQAppl2 appliance (this is the other tab in the browser). Click the **Manage** icon. You see that both HA queue managers are now runnng on MQAppl2. Click the **HAQM1** hyperlink.

	![](./images/image152a.png)

31. Navigate to the **Queues** tab and you can see, the 5 messages are all present on Q1 (on *MQAppl2*).

    ![](./images/image153a.png)

1.	From here, you can browse the messages on the queue. Click on **Q1**.

33. Verify that the messages all appear as you expect.

	![](./images/image155.png)

34. Click the breadcrumb for **Queues** to return to list of queues.

1. You now want to put the queue managers back the way they were. Go to the *MQAppl1* appliance.

36. Issue the **sethagrp -r** command to resume the appliance.

37. Go back to the Web console dashboard for MQAppl1. Refresh the page and click the manage icon if necessary to see that HAQM1 is now running on this appliance again and HAQM2 is still running on the other appliance.

	![](./images/image156a.png)	
38. Click the hyperlink for **HAQM1**, select the Queues tab. You will see that the 5 messages for Q1 are back where they belong, on the queue belonging to this queue manager on this appliance.

    ![](./images/image157a.png)
    


## Congratulations

This concludes the HA lab.
You have now successfully completed the setup and testing of the HA
environment between two MQ Appliances. This officially ends this lab. If
time allows, you may continue with the extra credit section.

<a name="extracredit"></a>	
## Extra Credit

If you have time to spare and want to try more testing, you may wish to
test failing over the other queue manager. Alternatively, you may wish
to test using other sample MQ utilities such as amqsputc and amqsgetc.

Bear in mind that if you want to run these utilities from the browser /
MQ Explorer image you will also need to do additional set up.

### Set up SYSTEM.DEF.SVRCONN

You can use the default svrconn channel for your client
    communication with the queue managers, but you will need to
    configure the channel authentication as you did previously for the
    SYSTEM.ADMIN.SVRCONN.

1. Using **runmqsc** for **HAQM1** and **HAQM2** on each of the appliances, perform the following:

	```
	SET CHLAUTH(SYSTEM.DEF.SVRCONN) TYPE(BLOCKUSER) USERLIST('*whatever')
    SET CHLAUTH(SYSTEM.DEF.SVRCONN) TYPE(USERMAP) CLNTUSER('ibmdemo') usersrc(MAP) MCAUSER('ibmdemo') 
	ALTER AUTHINFO('SYSTEM.DEFAULT.AUTHINFO.IDPWOS') AUTHTYPE(IDPWOS) ADOPTCTX(YES)

	REFRESH SECURITY TYPE(CONNAUTH)

	END
	```

### Set up variables

You also need to set the connection information for the MQSERVER and
MQSAMP\_USER\_ID variables.

Depending on whether you are running your test in a Windows or a
    Linux environment, they will have a slightly different format.

1. For Windows:
	
	```
	set MQSERVER=SYSTEM.DEF.SVRCONN/TCP/ipaddress(port) 

	set MQSAMP_USER_ID=ibmdemo
	```

2. For Linux:

	```
	export MQSERVER=SYSTEM.DEF.SVRCONN/TCP/'ipaddress(port)'

	export MQSAMP\_USER\_ID=ibmdemo
	```

You can change these variables to suit whichever particular appliance
and queue manager you are running a test for.

If you are unfamiliar with the sample programs and wish to use them,
please speak to the instructor.

<a name="configure"></a>
## Configure HA Using MQ Console


This section is included to show how to create an HA group and HA queue
managers using the MQ Console.

1. Open Firefox and open two tabs, one for the *MQAppl1* bookmark and one for the *MQAppl2* bookmark.

2. Sign in with **admin** / **Passw0rd!** in each tab so you are ready to respond to prompts within the timeout period.

1. In the *MQAppl1* tab, click the **Manage** icon on the left side bar.

	![](./images/image158b.png)

3. Click the **High Availability** tab. 

	![](./images/image160b.png)

1. Switch to the *MQAppl2* tab, click the **Manage** icon on the left side bar, and then click the **High Availability** tab. 

	![](./images/image160a.png)
	
1. Still on *MQAppl2*, click the **Set up high availability group** button.

	![](./images/image161a.png)

1. Enter **10.0.1.11** for the *IP address of the partner HA primary link*, then click the **Next** button to test the connection.

	![](./images/image162a.png)
	
1. If the ping is not successful, you will get an error message. When successful, the console will display a temporary key that must be used to complete the high availability setup on the second appliance. Be sure to make note of the *temporary key* as you need to enter it on the partner appliance *MQAppl1*.

	![](./images/image164a.png)
	
	> <span style="color: red">**Important:** <br>Note that this temporary key gets uniquely generated every time, and you only have 5 minutes to enter this temporary key in the Console of the second appliance before the process timesout.</span>


1. Switch back to the *MQAppl1* tab, and click the **Set up high availability group** button.

2. Enter **10.0.1.12** for the *IP address of the partner HA primary link*, then click the **Next** button to test the connection.

	![](./images/image164b.png)
	
	
1. When successful, the creation process moves to *Step 2*. Enter the *temporary key* and click **Next**. 

	![](./images/image165a.png)
		
1. Once the group creation is complete (be patient, it will take a couple minutes), you will receive a green *Success* message box popup and then the *HA group* and both appliances appear online with green checkmarks. Change to *MQAppl2* and you will see the same display on that appliance.

	![](./images/image167b.png)
	
	![](./images/image168a.png)

	Notice that no queue managers have been created yet. You must have the HA group defined before creating queue managers in the HA group.

10. Move to the *MQAppl1* browser tab. Click **Queue managers**. You see no queue managers in the *Manage* display. Click **Create** to create a new HA queue manager. 

    ![](./images/image169a.png)
    
11. Fill in the panel with the following values:
	-   Name: **HAQM1**
	-   Port: **1511**
	-   Define queue manager size: **2 GB**
	-	Auto start queue manager: **Checked**

	Click **Next**.

    ![](./images/image170b.png)     

12. On the second page of *Create a queue manager*, make sure to toggle the **High Availability** switch to **On**. Notice that you have the opportunity right then to define a floating IP address for the HA queue manager. Click the drop-down under **Local network interface** and select **eth0** which is the interface which applications connect to.

    ![](./images/image171a.png)
    
12. Ater you make the *eth0* selection, the IP adress to be used for the floating IP address is displayed and can be set. Using the arrow, increase the value to **18** (making the floating IP address 10.0.0.18).
    
    ![](./images/image171e.png) 
       
13. Click **Next**.

1.  IBM MQ Appliance 9.3 enabled disk encryption. Click the toggle switch to *On* for *Enable file system encryption*.

	![](./images/image172b.png)
	
1. Enter a passphrase in *Enter file system passphrase* and confirm it with the same value. You can click the *Show passphrase* icon to view what you entered.

	![](./images/image172c.png)

	> **NOTE:** Pick a passphrase that you can remember. *'mqappl'* was used in this case. Be sure to store the passphrase in a secure manner for future reference.
	
13. Click **Create**.
		
14. The queue manager status temporarily shows *Deploying* while it is being created and replicated to the second appliance. This will take a few minutes. 

	![](./images/image172d.png)

1.  The queue manager status will change to *Stopped*. Then you will receive a green success message and the status changes to *Running*. 

    ![](./images/image172e.png)

## Summary

This lab has focused on how to configure High Availability with the MQ Appliance. 

Congratulations, you have now completed this lab exercise. 
