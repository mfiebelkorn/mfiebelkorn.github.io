---
layout: post
title:  "SSPI Errors in an AD joined SQL Server"
date:   2016-12-15 12:13:00
comments: false
modified: 2016-12-15
---

Recently, our DevOps Engineer was stuck on an issue that our Development team was reporting. During their unit tests (seven hours worth of code testing usually ran nightly) they were getting a load of errors within Jenkins:

System.Data.Entity.Core.EntityException : The underlying provider failed on Open.
  ----> System.Data.SqlClient.SqlException : The target principal name is incorrect.  Cannot generate SSPI context.

Our DevOps Engineer (being the smart guy he is) passed it off to me (Infrastructure Engineer) because he recognized it as a Kerberos related issue. Since our Active Directory had been acting strangly for the past year ever since we started using a two-way trust with a company we merged with, I had no doubt this wouldn't be easy to solve. 

## Initial Ideas

So being I knew so little about this issue from the surface (I wasn't sure what target principal name they were using, and I didn't even know what SSPI stood for) I asked them to start with basics: Use a fully qualified SQL Connection string (no hostnames and include the port). I think they secretly rolled their eyes at me, but they tried it. It didn't fix anything. But it bought me some time and set a good troubleshooting base by eliminating more obvious isues that could arise from name resolution. 

A few Google searches produced alot of results, too many results. Time to figure out what all of these pieces are and how do they all fit together. 

## Digging into details

Security Support Provider Interface (SSPI) is a set of Windows APIs that allows for delegation and mutual authentication over any generic data transport layer, such as TCP/IP sockets. Therefore, SSPI allows for a computer that is running a Windows operating system to securely delegate a user security token from one computer to another over any transport layer that can transmit raw bytes of data. 

The "Cannot generate SSPI context" error is generated when SSPI uses Kerberos authentication to delegate over TCP/IP and Kerberos authentication cannot complete the necessary operations to successfully delegate the user security token to the destination computer that is running SQL Server.

Kerberos authentication uses an identifier named "Service Principal Name" (SPN). Consider an SPN as a domain or forest unique identifier of some instance in a server resource. You can have an SPN for a web service, for a SQL service, or for an SMTP service. You can also have multiple web service instances on the same physical computer that has a unique SPN.

I had a look to see what SPN's were registered with the server (setspn -l <servername>), and was surprised to see there were no MSSQL SPNs! 

![SPNS](/images/SPNS.PNG)

When an instance of the SQL Server Database Engine starts, SQL Server tries to register the SPN for the SQL Server service. When the instance is stopped, SQL Server tries to unregister the SPN. For a TCP/IP connection the SPN is registered in the format MSSQLSvc/<FQDN>:<tcpport>.Both named instances and the default instance are registered as MSSQLSvc, relying on the <tcpport> value to differentiate the instances.

For other connections that support Kerberos the SPN is registered in the format MSSQLSvc/<FQDN>/<instancename> for a named instance. The format for registering the default instance is MSSQLSvc/<FQDN>.

I looked up the AD Service Account in which the SQL Instance was running under (that service account would be the AD Account in which the registration is supposed to occur). I opened ASDI Editor and found the service account, under properties, security, advanced I added a new entry for principal "SELF" to "Allow" on "This Object Only" the property "Write servicePrincipalName"

![SPNWritePermission](/images/SPNWritePermission.PNG)

After you make this permission change it's required that you restart the SQL Server Instance. I had initially forgot to do this and it lead to some heartache. 

## Final Thoughts 

An [easier] alternative solution would be to give the SQL Service Account in which the instance is running under Domin Admin permissions. The reason that works is that the domain admins account inherently has the specific permission we granted in my solution above. However, this isn't ideal as it leads to an overelevated service account, which is poor practice. I only mention it because it might serve as a "quick'n dirty" way to test permission issues for others. 

After I implemented the solution and asked our Developement team to test, I was digging through SQL Logs and found that it had actually produced a helpful error. However, I missed it because it was only produced when the SQL Service was restarted (a month prior). I had been using Windows Event Viewer to reveiw SQL Logs (because I didn't have SQL Access initially) and the logs had purged themselves so this error was not showing in the Windows Event Viewer. Inside of the SQL Log Viewer the log file is turned over after every SQL Service Restart you can find the log entries from the last Service quickly by choosing the previous archived file.

![SQLLogs](/images/SQLLogs.png)

You can see the two entries above from the last Service restart stating in plain view that there was a problem auto creating the SPNs. After I made the permission change and restarted I found simlar entries in the logs saying that the SPNs were created successfully this time. 

## Sources/Additional Reading

[Microsoft Support - How to use Kerberos Authentication in SQL Server](https://support.microsoft.com/en-us/kb/319723)

[MSDN Library - Register a SPN for Kerberos Authentication](https://support.microsoft.com/en-us/kb/319723)

[Microsoft Support - Troubleshooting SSPI Error Messages](https://support.microsoft.com/en-us/kb/811889)
