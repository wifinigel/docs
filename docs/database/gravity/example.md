# Per-client blocking example

In this example, we describe how to set up blocking rule for three specific clients. All remaining (and newly added) clients in the network are "unmanaged", i.e., they use the Pi-hole as usual. The examples shown here built upon each other, i.e., example 5 might make no sense without the context of example 3.

Don't forget to run
```bash
pihole restartdns reload-lists
```
after your database modifications to have FTL flush it's internal domain-blocking cache (separate from the DNS cache).

## Prerequisites

1. Add three groups to our database. The third group will be disabled.
```sql
INSERT INTO "group" (id, name) VALUES (1, 'Group 1');
INSERT INTO "group" (id, name) VALUES (2, 'Group 2');
INSERT INTO "group" (id, name) VALUES (3, 'Group 3');
```

2. Add three clients to our database.
```sql
INSERT INTO client (id, ip) VALUES (1, '192.168.0.101');
INSERT INTO client (id, ip) VALUES (2, '192.168.0.102');
INSERT INTO client (id, ip) VALUES (3, '192.168.0.103');
```

3. Link the three clients to the three groups.
```sql
INSERT INTO client_by_group (client_id, group_id) VALUES (1, 1);
INSERT INTO client_by_group (client_id, group_id) VALUES (2, 2);
INSERT INTO client_by_group (client_id, group_id) VALUES (3, 3);
```

## Example 1: Exclude from blocking
**Task:** Exclude client 1 from Pi-hole's blocking
```sql
DELETE FROM client_by_group WHERE client_id = 1 AND group_id = 0;
```

#### Test
```text
192.168.0.101: dig +short doubleclick.net ---> 216.58.208.46  (not blocked)
192.168.0.102: dig +short doubleclick.net ---> 0.0.0.0        (blocked)
192.168.0.103: dig +short doubleclick.net ---> 0.0.0.0        (blocked)
192.168.0.104: dig +short doubleclick.net ---> 0.0.0.0        (blocked)
```
#### Result
All three client automatically got assigned to the default ("zero") group when they were added. The default group includes all adlists and list domains (if not already changed by the user). When we remove the default group for client `192.168.0.101`, we effectively remove all default associations to adlists and domains leaving this client completely unblocked.

## Example 2: Blocklist management
**Task:** Set up one blocklist for client `192.168.0.101` (groups 1).

### Step 1
Assign adlist with ID 1 to group 1
```sql
INSERT INTO adlist_by_group (adlist_id, group_id) VALUES (1,1);
```

#### Test
```text
192.168.0.101: dig +short doubleclick.net ---> 0.0.0.0  (blocked)
192.168.0.102: dig +short doubleclick.net ---> 0.0.0.0  (blocked)
192.168.0.103: dig +short doubleclick.net ---> 0.0.0.0  (blocked)
192.168.0.104: dig +short doubleclick.net ---> 0.0.0.0  (blocked)
```
#### Result
`192.168.0.101` is now sees `doubleclick.net` as being blocked as it uses an adlist including this domain. All other clients stay unchanged.

## Example 3: Blacklisting
**Task:** Add a single domain that should be **black**listed for client `192.168.0.101` (group 1).

### Step 1
Add the domain to be blocked
```sql
INSERT INTO domainlist (type, domain, comment) VALUES (1, 'blacklisted.com', 'Blacklisted for members of group 1');
```

#### Test
```text
192.168.0.101: dig +short blacklisted.com ---> 103.224.182.245  (not blocked)
192.168.0.102: dig +short blacklisted.com ---> 0.0.0.0          (blocked)
192.168.0.103: dig +short blacklisted.com ---> 0.0.0.0          (blocked)
192.168.0.104: dig +short blacklisted.com ---> 0.0.0.0          (blocked)
```
#### Result
Client `192.168.0.101` is not blocking this domain as we removed the default assignment through group 0 above. All remaining clients are linked through the default group to this domain and see it as being blocked.

### Step 2
Assign this domain to group 1
```sql
INSERT INTO domainlist_by_group (domainlist_id, group_id) VALUES (1, 1);
```
(the `domainlist_id` might be different for you, check with `SELECT last_insert_rowid();` after step 1)

#### Test
```text
192.168.0.101: dig +short blacklisted.com ---> 0.0.0.0  (blocked)
192.168.0.102: dig +short blacklisted.com ---> 0.0.0.0  (blocked)
192.168.0.103: dig +short blacklisted.com ---> 0.0.0.0  (blocked)
192.168.0.104: dig +short blacklisted.com ---> 0.0.0.0  (blocked)
```
#### Result
All clients see this domain as being blocked. Client 1 due to a direct assignment through group 1, all remaining clients through the default group 0 (unchanged).

### Step 3
Remove default assignment to all clients not belonging to a group
```sql
DELETE FROM domainlist_by_group WHERE domainlist_id = 1 AND group_id = 0;
```
(the `domainlist_id` might be different for you, see above)

#### Test
```text
192.168.0.101: dig +short blacklisted.com ---> 0.0.0.0          (blocked)
192.168.0.102: dig +short blacklisted.com ---> 103.224.182.245  (not blocked)
192.168.0.103: dig +short blacklisted.com ---> 103.224.182.245  (not blocked)
192.168.0.104: dig +short blacklisted.com ---> 103.224.182.245  (not blocked)
```
#### Result
While client 1 keeps its explicit assignment through group 1, the remaining clients lost their unassignments when we deleted the default group assignment.

## Example 4: Whitelisting
**Task:** Add a domain that should be **white**listed for client `192.168.0.102` (group 2).

### Step 1
Add the domain to be unblocked
```sql
INSERT INTO domainlist (type, domain, comment) VALUES (0, 'doubleclick.net', 'Whitelisted for members of group 2');
```

#### Test
```text
192.168.0.101: dig +short doubleclick.net ---> 0.0.0.0        (blocked)
192.168.0.102: dig +short doubleclick.net ---> 216.58.208.46  (not blocked)
192.168.0.103: dig +short doubleclick.net ---> 216.58.208.46  (not blocked)
192.168.0.104: dig +short doubleclick.net ---> 216.58.208.46  (not blocked)
```
#### Result
Client `192.168.0.101` is not whitelisting this domain as we removed the default assignment through group 0 above. All remaining clients are linked through the default group to this domain and see it as being whitelisted. Note that this is completely analog to [step 1](#step-1_1) of [example 3](#example-3-blacklisting).

### Step 2
Remove default group assignment
```sql
DELETE FROM domainlist_by_group WHERE domainlist_id = 2 AND group_id = 0;
```

#### Test
```text
192.168.0.101: dig +short doubleclick.net ---> 0.0.0.0  (blocked)
192.168.0.102: dig +short doubleclick.net ---> 0.0.0.0  (blocked)
192.168.0.103: dig +short doubleclick.net ---> 0.0.0.0  (blocked)
192.168.0.104: dig +short doubleclick.net ---> 0.0.0.0  (blocked)
```
#### Result
All requests are blocked as the new whitelist entry is not associated to any group and, hence, is not used by any client.

### Step 3
Assign this domain to group 2
```sql
INSERT INTO domainlist_by_group (domainlist_id, group_id) VALUES (2, 2);
```
(the `domainlist_id` might be different for you, check with `SELECT last_insert_rowid();` after step 1)

#### Test
```text
192.168.0.101: dig +short doubleclick.net ---> 0.0.0.0        (blocked)
192.168.0.102: dig +short doubleclick.net ---> 216.58.208.46  (not blocked)
192.168.0.103: dig +short doubleclick.net ---> 0.0.0.0        (blocked)
192.168.0.104: dig +short doubleclick.net ---> 0.0.0.0        (blocked)
```

#### Result
Client 2 got the whitelist entry explicitly assigned to. Accordingly, client 2 does not see the domain blocked whereas all remaining client still see this domain as blocked (unchanged).
