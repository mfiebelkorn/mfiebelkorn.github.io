---
layout: post
title:  "MSSQL Bulk Admin Importing"
date:   2017-05-24 09:05:00
comments: false
modified: 2017-05-24
---

This week our developers are putting the finishing touches on a new feature in which they are needing to do BULKINSERTS. Since our production environment is multi-node, I setup a UNC Path on our DFS Namespace so that all the nodes would have access to the same location from which to import from. 

After ensuring that all permissions were correct (Share Permissions, NTFS Permissions, SQL Permissions), we were still getting something that looked alot like a permission error:
~~~~
msg 4861, Level 16, State 1, Line 1
Cannot bulk load because the file “\\SomeShare\SomeFolder\SomeFile.txt” could not be opened. Operating system error code 5(Access is denied).
~~~~

I had assumed that the NTFS Identity being used was that of the SQL Service Account, but that was an incorrect assumption. It turns out that the SQL Server is passing the credentials by way of a double hop! This requires Kerberos and delegation!
![DoubelHop](/images/SQLPermissionHop.png)

It turns out that MSSQL Uses NTLM by default instead of Kerberos, you can verify this by running the following query:
SELECT net_transport, auth_scheme
FROM sys.dm_exec_connections
WHERE session_id = @@spid

In order to ensure Kerberos delegation is permitted, we need to check SPNs (Service Principal Names). Executing setspn -L <sql service account> displayed no SPNs! So there is a good place to start, I ran the following:
~~~~
setspn -A MSSQLSvc/<SQLServer>:1433 <domain>/<sql service account>
~~~~

Then executed the query above to verify, it now shows Kerberos! Great! One last thing to do, allow delegation on the service account. One quick note on this, if you try this without the SPN set, it will not show up as an option. 
![DelegationPermission](/images/DelegationPermission.PNG)

Everything worked great! Until we promoted this change to our Systest environment which used SQL AlwaysOn. When registering the SPN in an AlwaysOn Cluster, you must use the AlwaysOn Listner Name. Additionally, for the SPN to work across all replica's, you need to use the same service account for all instances in the Cluster. We initailly set the SPN on the active node's service account, everything worked, so then at that point our DBAs changed the service account on the secondary nodes to be the same as the primary node. That's not the best order of operations, but it was done in a trial/error troubelshooting session, so we didn't complain too much. The SPN difference was subtle, but caused me enough grief to make a learnign experience out of it:
~~~~
setspn -A MSSQLSvc/<AlwaysOn Listener Name>:1433 <domain>/<sql service account>
~~~~

## Sources/Additional Reading
[MS Documentation](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/listeners-client-connectivity-application-failover#SPNs)

[Arjan Fraaij Blog Post](http://blog.arjanfraaij.com/2010/12/bulk-admin-operating-system-error-code.html)
