Authors
=======
Guillaume Daumas : guillaume.daumas@univ-tlse3.fr

Ahmad Samer Wazan : ahmad-samer.wazan@irit.fr

Rémi Venant: remi.venant@gmail.com



Intro
=====

This code implements a role based approach for distributing Linux capabilities into Linux users. It allows assigning Linux capabilities to Linux users without the need to inject the Linux capabilities into executable files. Our code is a PAM-based module that leverages a new capability set added to Linux kernel, called Ambient Set. Using this module, administrators can group a set of Linux capabilities in roles and give them to their users. For security reasons, users don’t get the attributed roles by default, they should activate them using the command sr (substitute role). Our module is compatible with pam_cap.so. So administrators can continue using pam_cap.so along with our module.

Tested Platforms
===========
Our module has been tested only on Ubuntu and Debian platforms.

Installation
===========

How to Build
------------

	1. git clone https://github.com/guillaumeDaumas/switchRole
    
    2. cd switchRole
    
    3. ./configure.sh
    
    4. make
    
    5. sudo make install
    
    6. restart your system.

Usage
-----

After the installation you will find a file called capabilityRole.conf in the /etc/security directory. You should configure this file in order to define the set of roles and assign them to users or group of users on your system.

**assume Roles**

Once configuration is done, a user can assume a role using the ‘sr’ tool  that is installed with our package. In your shell type for example :

`sr -r role1` 

After that a new shell is openned. This shell contains the capabilities of the role that has been taken by the user. You can verify by reading the capabilities of your shell (cat /proc/$$/status). When you make an exit() you retrun to your initial shell without any privilege. 

**No Root mode**

You have the possibility to launch a full capabale shell that doesn't give any special treatment to uid 0. The root user is considered as any other normal user and you can in this case grant him a few privileges in the capabilityRole.conf distributed by our module :

`sr -n -r role1`

We use the securebits to provide this functionality. Any set-uid-root program will be run without having any special effect. So in the shell, you can't for example use the ping command without a role that has cap_net_raw privilege.


Motivation scenarios
===========

Scenario 1
-----
A user contacts his administrator to give him a privilege that allows him running an HTTP server that is developed using Python. His script needs the privilege CAP_NET_BIND_SERVICE to bind the server socket to 80 port.  Without our module, the administrator has two options: (1)  Use setcap command to inject the privilege into Python interpreter or (2) use pam_cap.so to attribute the CAP_NET_BIND_SERVICE to the user and then inject this privilege in the inheritable and effective sets of the interpreter. Both solutions have security problems because in the case of option (1), the Python interpreter can be used by any another user with this privilege. In the case of option (2) other python scripts run by the legitimate user will have the same privilege.


Here a simple python script that needs to bind a server on the port 80 [9] (the user running the script needs CAP_NET_BIND_SERVICE to do that).

![Screenshot](scenarioPython/codeServer.png)


If we try to execute the script without any privilege, we get the expected 'Permission denied'.
![Screenshot](scenarioPython/connectionFailed.png)


The first solution consists in using the setcap command in order to attribute the cap_net_bind_service capability to the python interpreter. Doing this create a security problem; now users present in the same system have the same privilege. 
![Screenshot](scenarioPython/connectionWithSetcap.png)

The second solution is to use pam_cap.so module, as follows:

The administrator sets cap_net_bind_service in the /etc/security/capability.conf file (pam_cap's configuration file).

![Screenshot](scenarioPython/pam_capConf.png)

As you see, the inheriable set of the shell has now the new capability.

![Screenshot](scenarioPython/bashPamCap.png)

The administrator has to use setcap command to inject cap_net_bind_service in the Effective and Inheritable set of the interpreter. After that the user can run the script.

![Screenshot](scenarioPython/connectionWithPamCap.png)

However, in this case all scripts run by the same user will have the same privilege :(

Our solution provides a better alternative. Suppose that the capabilityRole.conf contains the following configuration:

                                  role1 cap_net_bind_service guillaume none 
Then the user needs only to assume role1 using our sr tool and then run his (her) script. (S)he can use other shell to run the other non-privileged scripts.


![Screenshot](scenarioPython/connectionWithRole.png)



And as we can see here, python binary doesn't have any capabilities.

Scenario 2 
-----
Suppose a developer wants to test a program that (s)he has developed in order to reduce the downloading rate on his server. The developer should use the LD_PRELOAD environment variable to load his shared library that intercepts all network calls made by the servers’ processes. With the current capabilities tools, the administrator of the server can use setcap command or pam_cap.so to give the developer cap_net_raw. However, the developer cannot achieve his test because, for security reasons, LD_PRELOAD doesn't work when capabilities are set in the binary file.

This is an example program which tries to open a raw socket (cap_net_raw needed) [10]:

![Screenshot](scenarPreload/codeSocket.png)

This is a code which tries to intercept the socket() call [10].

![Screenshot](scenarPreload/codeCapture.png)

As we can see, we need cap_net_raw to open this socket.

![Screenshot](scenarPreload/socketWithoutWithSetcap.png)

But, when the LD_PRELOAD is configured, Linux kernel disable the interception of socket() call. The following figure shows how the interception works correctly after having removed the capability from the binary file (setcap -r), and not correctly when the capacity is re-added to the binary file.

![Screenshot](scenarPreload/preloadWithoutWithSetcap.png)

With our module, no need set the capability in the binary file: you can open the socket and intercept the connection.

![Screenshot](scenarPreload/preloadWithRole.png)



Scenario 3 
-----
an administrator wants to attribute a user the privilege to run an apache server. Without our module, the administrator can use either setcap command to inject the necessary privilege in the binary of apache server (option1) or use pam_cap.so module and setcap command (option2). Both options have problems: All systems' users will get this privilege in the case of option 1. The configuration of the binary apache will be lost after updating the apache package. 

To achieve this objective using our module, an admininstrator should follow the following steps:

1- Grant the privilege cap_bet_bind_service, cap_dac_override to the user by editing capabilityRole.conf file. Note that cap_dac_override is not mandatory if the administraor changes the ownership of the log files.

2- Define a script (lets call it runapache.sh) that has the following commands: source /etc/apache2/envvars and /usr/sbin/apache2
						   
3-User can assume the role and run the apache service using the command sr:
sr -r role1  -c 'runapache.sh'

4-verify that the apache process has only the cap_net_bind_service and cap_dac_override in its effective using this command: cat /proc/PID/status



Scenario 4 
-----
Two developers create a shared folder in which they stored a common program that they develop together. This program requires cap_net_raw privilege. The developers have to mount their shared folder using NFS v3.  This scenario is not feasible with the current tools because NFS v3 doesn’t support extended attributes. 



NoRoot Scenario 
-----
With -n option the kernel considers the root user as any normal user. 

Here, the sr tool is launched with the root user but without -n : 

![Screenshot](scenarioNoRoot/rootWithoutNoroot.png)

as you see, the root user obtains the full list of privileges.

Here is the result when we add the -noroot option : 

![Screenshot](scenarioNoRoot/rootNorootRole1.png)

As you see now, the root user didn't obtain the full list of privileges.

Under this mode, set-uid-root programs will not have the full list of privileges and they can not be launched.

For example, lets check result of the ping program. Suppose that we assign the role2 to root user as follows:

![Screenshot](scenarioNoRoot/rootConfRole2.png)

When the root user assume the role2 with -noroot option, he can't use `ping 0` because cap_net_raw is not present its role.

![Screenshot](scenarioNoRoot/rootRole2NoRootCantPing.png)

If we modify the configuration to assign role1 that conains cap_net_raw privilege to root user, we see that he can now use ping:

![Screenshot](scenarioNoRoot/rootConfRole1.png)


![Screenshot](scenarioNoRoot/rootRole1NoRootPing.png).

Service Managment Scenario 
-----

An administrator wants to launch a set of services like apache and ssh by giving them only the privileges that they need. (S)he may use the command setcap or pam_cap.so to define the necessary privileges in the binary of each service. However all configuration will be lost after an update.

To launch the apache service using our module, an admininstrator should follow the following steps:

1- Grant the privilege cap_bet_bind_service to the user by editing capabilityRole.conf file,

2- Define a script (lets call it runapache.sh) that has the follwing commands: source /etc/apache2/envvars
						   /usr/sbin/apache2
						   
3-as a root, run the follwing command:

sr -r role1 -u apacheuser -n -c 'runapache.sh'

4-verify that the apache process has only the cap_net_bind_service in its effective using this command: cat /proc/PID/status

How sr works
===========
You might be interested to know how we implement the sr tool. So here is the algorithm: 

![Screenshot](algorithm2.jpg)

In terms of capabilities calucations by Linux Kernel, here is what happens:

![Screenshot](calculations.jpg)


References
==========

[1] PAM repository : https://github.com/linux-pam/linux-pam

[2] libcap repository : https://github.com/mhiramat/libcap



Very helpfull site, where you can find some informations about PAM, libcap and the capabilities:


[3] Original paper about capabilities : https://pdfs.semanticscholar.org/6b63/134abca10b49661fe6a9a590a894f7c5ee7b.pdf

[4] Article about the capabilities : https://lwn.net/Articles/632520/

[5] Article about Ambient : https://lwn.net/Articles/636533/

[6] Simple article with test code for Ambient : https://s3hh.wordpress.com/2015/07/25/ambient-capabilities/

[7] Article about how PAM is working : https://artisan.karma-lab.net/petite-introduction-a-pam

[8] A very helpfull code about how to create a PAM module : https://github.com/beatgammit/simple-pam



Source of the scenarios code:

[9] Where I have found the simple Python code for HTTP server : https://docs.python.org/2/library/simplehttpserver.html

[10] Where I have found the simple PRELOAD code : https://fishi.devtail.io/weblog/2015/01/25/intercepting-hooking-function-calls-shared-c-libraries/
