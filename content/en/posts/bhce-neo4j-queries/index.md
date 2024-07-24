+++
title = 'BloodHound CE and neo4j queries - statistics, ADCS and more'
date = 2024-07-24T20:30:08+03:00
draft = false
+++

The goal of this blog post is to showcase some queries in neo4j and the BloodHound GUI, including how to hunt for some ADCS escalation paths beyond the defaults available in BloodHound.

Until recently I had exclusively been using the now legacy version of BloodHound with my trusty old friend the `bloodhound-python` ingestor. I decided it's time to explore BloodHound Community Edition and I ended up experimenting quite a lot with custom queries both directly in the neo4j database and the BloodHound GUI itself. I ended up figuring out how Cypher works (more or less), reading a ton of queries other people have written, picking and choosing, adapting and tinkering to achieve my desired results. (*I also finally figured out how to do shortest paths from owned principals, I was ready to pick up a pitchfork and dismiss CE due to the lack of that alone.*)

I won't be going through how to set up BHCE or import data or any of that, deployment is straightforward via Docker. Nor is this meant to teach you Cypher, just to open you up to the possibilities it offers and maybe get you some quick wins.

I'm using the latest version of BHCE and the SharpHound ingestor included with it at the time of writing. The data used for demonstration purposes was collected in my home lab setup of [GOAD](https://github.com/Orange-Cyberdefense/GOAD) + some additions of vulnerable ADCS templates taken from my own [ADCS project](https://github.com/arth0sz/Practice-AD-CS-Domain-Escalation) on GitHub and a few extra misconfigurations here and there. 
## Neo4J queries for statistics

I'm going to start with some neo4j queries that are useful for statistics purposes and for providing a quick and dirty overview of the environment. It's not as pretty and polished as the GUI, but it will give you the results in tables you can export as `.cvs` files and will enable you to quickly summarise different kinds of information and perform some searches that aren't possible in BloodHound itself. 
#### Users by group

The following query will return back all groups along with their unique member counts and the group description. It might end up showing you some over-privileged custom groups and give you a rough target to aim for and dig deeper. 

```crystal
MATCH (g:Group)<-[:MemberOf*1..]-(u:User) RETURN g.name AS groupName, g.description AS
groupDescription, COUNT(DISTINCT u) AS numberOfMembers ORDER BY numberOfMembers DESC
```

![](Pasted%20image%2020240716231148.png)
#### Operating systems

This query will just return all the operating systems in use. Very useful to quickly find out if there's any unsupported systems present in the environment that could provide you with some quick wins.

```crystal
MATCH (c:Computer) RETURN DISTINCT c.operatingsystem as operatingSystem,
COUNT(c.operatingsystem) as osCount
```

![](Pasted%20image%2020240716231305.png)
#### Krbtgt accounts

This query will return only the krbtgt accounts, and show whether the password was changed any time recently. Might give you an idea if that's managed and rotated on a regular basis, or the existence and significance of the account has long been forgotten. 

```crystal
MATCH (u:User) WHERE u.hasspn = True AND u.name STARTS WITH 'KRBTGT' RETURN u.name AS
accountName, datetime({ epochSeconds:toInteger(u.pwdlastset) }) AS passwordLastSet
```

![](Pasted%20image%2020240716231352.png)

You can take the next query and extend this one by adding the date the account was created as well.
#### Domain Admins and password last set

This query will return all Domain Admins in the environment along with the dates the account was created and when their password was last changed. First of all, this gives you a list of all accounts you'd likely want to target. It also provides information on whether there's regular password rotation. This might not mean much, if all the accounts are behind a Privileged Access Management solution, MFA and/or happen to have 16+ character passwords.

```crystal
MATCH (g:Group) WHERE g.name =~ "(?i).*DOMAIN ADMINS.*" WITH g MATCH (g)<-
[r:MemberOf*1..]-(u) RETURN u.name AS User,
datetime({epochSeconds:toInteger(u.pwdlastset)}) as passwordLastSet, datetime({epochSeconds: toInteger(u.whencreated)}) AS dateCreated ORDER BY u.pwdlastset
```

![](Pasted%20image%2020240716231609.png)
#### Service accounts, password last set, date created

This query does the same as the above, except that it relates to service accounts. Kerberoasting is a very well-known and popular attack vector and this can quickly give you an idea of whether passwords are rotated or not. Passwords that haven't been changed in years combined with a weak password policy may give you a quick win.

```crystal
MATCH (u:User) WHERE u.hasspn=true AND (NOT u.name STARTS WITH 'KRBTGT') RETURN u.name
AS accountName, datetime({epochSeconds: toInteger(u.pwdlastset)}) AS passwordLastSet,
datetime({epochSeconds: toInteger(u.whencreated)}) AS dateCreated ORDER BY u.pwdlastset
```

![](Pasted%20image%2020240716231746.png)
#### Users and descriptions

I like just using neo4j to read user descriptions. Even if there aren't any passwords left in there, it may give you a good overview of employees and their roles in the company, potentially revealing targets to focus on.

```crystal
MATCH (u:User) RETURN u.name as username, u.description as description
```

![](Pasted%20image%2020240716232023.png)

You can add `WHERE u.enabled = True` before the `RETURN` statement in order to limit the results to enabled users only. You can also extend this to show date created and password last set, if you want to.
#### Password never expires

Another thing you might want to look at is accounts with password never expires set. 

```crystal
MATCH (u:User {pwdneverexpires: True}) WHERE NOT u.name starts with 'KRBTGT' RETURN
u.name AS username, u.description AS description, datetime({
epochSeconds:toInteger(u.lastlogon)}) AS lastLogon ORDER BY lastLogon DESC
```

![](Pasted%20image%2020240716232153.png)
## BloodHound queries

### General

After having gone through some statistics, let's take a look at some BloodHound GUI queries to help identify potential paths and targets for escalation.
#### Domain Admins or Administrators with hasSession on non-DC computers

Ideally, users in the Domain Admins or Administrators should not log on to workstations, but this is an avenue always worth exploring. If an over-privileged user has a session on a computer and you manage to compromise that machine, you might be able to hijack that session, steal a token, steal a ticket, get the keys to the kingdom.

The first part of the query will look for and match the Domain Controllers so that we can exclude them as we wouldn't really need to look for paths to DA if we had compromised the DC already.  This will be utilised a few times in various queries.

```crystal
MATCH (c1:Computer)-[:MemberOf*1..]->(g:Group) WHERE g.objectid ENDS WITH '-516' WITH
COLLECT(c1.name) AS domainControllers
MATCH (n:User)-[:MemberOf*1..]->(g:Group) WHERE g.objectid ENDS WITH '-512' OR
g.objectid ENDS WITH '-544'
MATCH p = (c:Computer)-[:HasSession]->(n) WHERE NOT c.name in domainControllers return
p
```

![](Pasted%20image%2020240723221548.png)
#### Unconstrained Delegation Computers Excluding DCs

This query will look for computers with Unconstrained Delegation configured to either give you a potential target to aim for or a vulnerability to fix.

```crystal
MATCH (c1:Computer)-[:MemberOf*1..]->(g:Group) WHERE g.objectid ENDS WITH '-516' WITH
COLLECT(c1.name) AS domainControllers
MATCH (c:Computer {unconstraineddelegation:true}) WHERE NOT c.name IN domainControllers
RETURN c
```

![](Pasted%20image%2020240723221721.png)
#### Users with Constrained Delegation Permissions

I think the query searching for Constrained Delegation is pretty well-known, but it would be remiss of me not to include it after Unconstrained Delegation.

```crystal
MATCH p = ((u:User)-[r:AllowedToDelegate]->(c:Computer)) RETURN p
```

![](Pasted%20image%2020240723222013.png)
#### Shortest Path to Domain Admins

This is one of the main queries we know and love, covering a couple of tweaks I find to be useful. The query includes any and all edges that can be abused to move forward.
##### From Enabled Users

The query is essentially the following at a high level: find the shortest path from users with the following rights, regardless of how many steps are in-between to groups, where the users are enabled and the group is Domain Admins (whose SID ends with `-512`).

```crystal
MATCH p=shortestPath((n:User)-[:Owns|GenericAll|GenericWrite|WriteOwner|WriteDacl|MemberOf|ForceChangePassword|AllExtendedRights|AddMember|HasSession|Contains|GPLink|AllowedToDelegate|TrustedBy|AllowedToAct|AdminTo|CanPSRemote|CanRDP|ExecuteDCOM|HasSIDHistory|AddSelf|DCSync|ReadLAPSPassword|ReadGMSAPassword|DumpSMSAPassword|SQLAdmin|AddAllowedToAct|WriteSPN|AddKeyCredentialLink|SyncLAPSPassword|WriteAccountRestrictions|GoldenCert|ADCSESC1|ADCSESC3|ADCSESC4|ADCSESC5|ADCSESC6a|ADCSESC6b|ADCSESC7|ADCSESC9a|ADCSESC9b|ADCSESC10a|ADCSESC10b|ADCSESC13|DCFor*1..]->(m:Group))
WHERE n.enabled = True AND m.objectid ENDS WITH "-512"
RETURN p
```

That actually returns quite a lot of information, even though this is a small environment, so it might have trouble rendering in an environment with thousands of users.

![](Pasted%20image%2020240722233749.png)

We can narrow things down as follows:
##### From Kerberoastable Users

Change `n.enabled` to `n.hasspn` and you have the shortest path to Domain Admins from Kerberoastable users. Maybe you already know how to compromise one, maybe this will give you a target to aim for.

```crystal
MATCH p=shortestPath((n:User)-[:Owns|GenericAll|GenericWrite|WriteOwner|WriteDacl|MemberOf|ForceChangePassword|AllExtendedRights|AddMember|HasSession|Contains|GPLink|AllowedToDelegate|TrustedBy|AllowedToAct|AdminTo|CanPSRemote|CanRDP|ExecuteDCOM|HasSIDHistory|AddSelf|DCSync|ReadLAPSPassword|ReadGMSAPassword|DumpSMSAPassword|SQLAdmin|AddAllowedToAct|WriteSPN|AddKeyCredentialLink|SyncLAPSPassword|WriteAccountRestrictions|GoldenCert|ADCSESC1|ADCSESC3|ADCSESC4|ADCSESC5|ADCSESC6a|ADCSESC6b|ADCSESC7|ADCSESC9a|ADCSESC9b|ADCSESC10a|ADCSESC10b|ADCSESC13|DCFor*1..]->(m:Group))
WHERE n.hasspn = True AND m.objectid ENDS WITH "-512"
RETURN p
```

![](Pasted%20image%2020240722232604.png)
##### From Owned Users/Principals

One of my favourites in BloodHound that I dearly missed the first time I ever tried BHCE.

If instead you use `WHERE n.system_tags CONTAINS "owned"`, you get the shortest path from owned users. That one took me a while to figure out as there was a subtle, but important change between versions - in the legacy version of BloodHound you could just do `n.owned = True`.

```crystal
MATCH p=shortestPath((n:User)-[:Owns|GenericAll|GenericWrite|WriteOwner|WriteDacl|MemberOf|ForceChangePassword|AllExtendedRights|AddMember|HasSession|Contains|GPLink|AllowedToDelegate|TrustedBy|AllowedToAct|AdminTo|CanPSRemote|CanRDP|ExecuteDCOM|HasSIDHistory|AddSelf|DCSync|ReadLAPSPassword|ReadGMSAPassword|DumpSMSAPassword|SQLAdmin|AddAllowedToAct|WriteSPN|AddKeyCredentialLink|SyncLAPSPassword|WriteAccountRestrictions|GoldenCert|ADCSESC1|ADCSESC3|ADCSESC4|ADCSESC5|ADCSESC6a|ADCSESC6b|ADCSESC7|ADCSESC9a|ADCSESC9b|ADCSESC10a|ADCSESC10b|ADCSESC13|DCFor*1..]->(m:Group))
WHERE n.system_tags CONTAINS "owned" AND m.objectid ENDS WITH "-512"
RETURN p
```

![](Pasted%20image%2020240722232820.png)

Here I marked a few users as owned at random. You can also just leave it as `((n)` which will match any principal instead (or match computers `((n:Computer)` for example.) 

> Another thing to keep in mind: the letters you see in these queries `u, n, c` can really be anything. 
> I like to use the first letter of what I'm matching (*c for computer*), but you can use any letter you like or full words as well as long as you're consistent within the query.

##### From Computers Excluding DCs

This next query will show you paths to Domain Admins from Computers. 

```crystal
MATCH (c1:Computer)-[:MemberOf*1..]->(g:Group) WHERE g.objectid ENDS WITH '-516' WITH COLLECT(c1.name) AS domainControllers
MATCH p=shortestPath((n:Computer)-[:Owns|GenericAll|GenericWrite|WriteOwner|WriteDacl|MemberOf|ForceChangePassword|AllExtendedRights|AddMember|HasSession|Contains|GPLink|AllowedToDelegate|TrustedBy|AllowedToAct|AdminTo|CanPSRemote|CanRDP|ExecuteDCOM|HasSIDHistory|AddSelf|DCSync|ReadLAPSPassword|ReadGMSAPassword|DumpSMSAPassword|SQLAdmin|AddAllowedToAct|WriteSPN|AddKeyCredentialLink|SyncLAPSPassword|WriteAccountRestrictions|GoldenCert|ADCSESC1|ADCSESC3|ADCSESC4|ADCSESC5|ADCSESC6a|ADCSESC6b|ADCSESC7|ADCSESC9a|ADCSESC9b|ADCSESC10a|ADCSESC10b|ADCSESC13|DCFor*1..]->(m:Group))
WHERE NOT n.name IN domainControllers AND m.objectid ENDS WITH "-512"
RETURN p
```

![](Pasted%20image%2020240722234920.png)
### ADCS

I have a strange love for ADCS and the various ways it can be misconfigured, so I was interested to see what this version of BloodHound can find in that regard. This is not only useful for attacking ADCS, but also for auditing the security of the ADCS currently in use in the given environment.

When you review the default queries in BloodHound CE, you will see that there are queries available for ESC1 and ESC2. I'm not going to get into depth on the requirements for the escalation paths as SpecterOps and various security researches have already done that, you can read the thorough breakdowns by SpecterOps for the first two in BloodHound [here](https://posts.specterops.io/adcs-attack-paths-in-bloodhound-part-1-799f3d3b03cf) and [here](https://posts.specterops.io/adcs-attack-paths-in-bloodhound-part-1-799f3d3b03cf). 

The latter actually talks about how edges have been incorporated into BloodHound (a bit more on that later) for various escalation paths, which includes ESC3, but I wanted to craft my own query for it as well. The following query matches all principles that have enrolment rights (+ a very quick check for other excessive rights) on certificate templates published to the EnterpriseCA, where manager approval is disabled, there are no authorized signatures required or the PKI schema version is 1, and the certificate template has Certificate Request Agent as EKU (Extended Key Usage). 
#### ESC3

```crystal
MATCH p = ()-[:Enroll|GenericAll|AllExtendedRights]->(ct:CertTemplate)-[:PublishedTo]->(:EnterpriseCA) Where "1.3.6.1.4.1.311.20.2.1" IN ct.ekus AND ct.requiresmanagerapproval = False
AND (ct.authorizedsignatures = 0 OR ct.schemaversion = 1) return p	 
```

![](Pasted%20image%2020240716232555.png)

In this case, the query will return that the Domain Users in the `essos.local` and `sevenkingdoms.local` can abuse this escalation path, which is confirmed by the built in edge in BHCE as follows below.

We can add `WHERE n.admincount = False` to the query as well to exclude any accounts with elevated rights from the visualization. The same will be utilised in the next query.

```crystal
MATCH p = (n)-[:Enroll|GenericAll|AllExtendedRights]->(ct:CertTemplate)-[:PublishedTo]->(:EnterpriseCA) WHERE n.admincount = False AND "1.3.6.1.4.1.311.20.2.1" IN ct.ekus AND ct.requiresmanagerapproval = False
AND (ct.authorizedsignatures = 0 OR ct.schemaversion = 1) return p	
```

![](Pasted%20image%2020240723231841.png)
#### Generic query for ADCS edges

This following query will display all available ADCS edges in BloodHound so long as the ingestor has managed to collect the required information. This leaves us again with the *Domain Users* for the aforementioned domains who can abuse both ESC1 and ESC3. We also see an edge for ESC4, which is the next one I will cover.

```crystal
MATCH p= ((n)-[:ADCSESC1|ADCSESC3|ADCSESC4|ADCSESC5|ADCSESC6a|ADCSESC6b|ADCSESC7|ADCSESC9a|ADCSESC9b|ADCSESC10a|ADCSESC10b|ADCSESC13]->()) WHERE n.admincount = False
RETURN p
```


![](Pasted%20image%2020240716232856.png)
#### ESC4 

Very briefly, ESC4 is when a user has write privileges over a certificate template. We can just search for edges that will give a user those rights. The first part of the query is another way to exclude admin users from our search. It will match any users in Domain Admins or the Administrators group and collect them under the name `adminsGroup`, which we can then use to exclude them: `WHERE NOT u.name in adminsGroups`. The query gives us the exact same result as the built-in edge we just saw.

```crystal
MATCH (n:User)-[:MemberOf*1..]->(g:Group) WHERE g.objectid ENDS WITH '-512' OR g.objectid ENDS WITH '-544' WITH COLLECT(n.name) AS adminsGroups
MATCH p = (u:User)-[:WriteDacl|GenericAll|AllExtendedRights|GenericWrite|Owns]->(ct:CertTemplate)-[:PublishedTo]->(:EnterpriseCA) WHERE NOT u.name in adminsGroups
RETURN p
```

![](Pasted%20image%2020240716232634.png)
#### ESC7

The last escalation path I will cover is ESC7 which is a simple one to find. If we have a user with `ManageCA` permissions, they can grant themselves the `Manage Certificates` right as well and proceed with the escalation path abuse.

```crystal
MATCH (n:User)-[:MemberOf*1..]->(g:Group) WHERE g.objectid ENDS WITH '-512' OR g.objectid ENDS WITH '-544' WITH COLLECT(n.name) AS adminsGroups
MATCH p = (u:User)-[:ManageCA]->(:EnterpriseCA) WHERE NOT u.name in adminsGroups
RETURN p
```

![](Pasted%20image%2020240716232727.png)
## Final Words

If you've made it this far, thanks for reading and hope you've found something useful here. There are countless queries you can utilise in neo4j to get summaries, overviews and statistics depending on your wants and needs, so the few listed here are just the beginning. 

There are plenty of good and well-known extensive lists of BloodHound queries available as well, though not all have been updated to fit the Community Edition. You won't find the exact things I've listed here as I've adapted things to fit my own goals and everyone utilises Cypher a bit differently. 

You can look up various Cypher guides and tutorials, but I think if you read enough queries and try to apply them to data you know and understand, you'll figure out the syntax much faster. 







