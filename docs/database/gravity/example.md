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

## Example 1. Exclude from blocking
**Task:** Exclude clients from Pi-hole's blocking - already done!

#### Test
```text
192.168.0.101: dig +short doubleclick.net ---> 216.58.208.46  (not blocked)
192.168.0.102: dig +short doubleclick.net ---> 216.58.208.46  (not blocked)
192.168.0.103: dig +short doubleclick.net ---> 216.58.208.46  (not blocked)
192.168.0.104: dig +short doubleclick.net ---> 0.0.0.0        (blocked)
```

#### Result
All three client have been assigned to "empty" groups. "Empty" in this regard means that the corresponding group is not assigned to *any* black-, white or adlists. Hence, these clients do not get anything blocked at all. Add adlists and black- / whitelist domains if you want to change this (see below).

## Example 2: Blocklist management
Task: Set up **full gravity blocking** for clients `192.168.0.102` and `192.168.0.102` (groups 2+3).

### Step 1
Assign all adlists to group 2
```sql
INSERT INTO adlist_by_group (adlist_id, group_id) SELECT id, 2 FROM adlist;
INSERT INTO adlist_by_group (adlist_id, group_id) SELECT id, 3 FROM adlist;
```
We used the `SELECT` statement as shortcut for getting all adlist IDs.
We could also have done this step-by-step for all adlists and groups:
```sql
INSERT INTO adlist_by_group (adlist_id, group_id) VALUES (1,2);
INSERT INTO adlist_by_group (adlist_id, group_id) VALUES (2,2);
INSERT INTO adlist_by_group (adlist_id, group_id) VALUES (3,2);
[...]
INSERT INTO adlist_by_group (adlist_id, group_id) VALUES (1,3);
INSERT INTO adlist_by_group (adlist_id, group_id) VALUES (2,3);
INSERT INTO adlist_by_group (adlist_id, group_id) VALUES (3,3);
[...]
```

#### Test
```text
192.168.0.101: dig +short doubleclick.net ---> 216.58.208.46  (not blocked)
192.168.0.102: dig +short doubleclick.net ---> 0.0.0.0        (blocked)
192.168.0.103: dig +short doubleclick.net ---> 0.0.0.0        (blocked)
192.168.0.104: dig +short doubleclick.net ---> 0.0.0.0        (blocked)
```

#### Result
- `192.168.0.101` - Not blocked as group 1 is not associated to any adlist (as before).
- `192.168.0.102` - Blocked as group 2 is associated to all adlists.
- `192.168.0.103` - Blocked as group 3 is associated to all adlists.
- `192.168.0.104` - Blocked as not managed by any group - all adlist domains are used.

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
192.168.0.102: dig +short blacklisted.com ---> 103.224.182.245  (not blocked)
192.168.0.103: dig +short blacklisted.com ---> 103.224.182.245  (not blocked)
192.168.0.104: dig +short blacklisted.com ---> 0.0.0.0          (blocked)
```

#### Result
- `192.168.0.101` - Not blocked as group 1 is not associated to the new blacklisted entry.
- `192.168.0.102` - Not blocked as group 2 is not associated to the new blacklisted entry.
- `192.168.0.103` - Not blocked as group 3 is not associated to the new blacklisted entry.
- `192.168.0.104` - Blocked as not managed by any group - all blacklisted domains are used.

### Step 2
Assign this domain to group 1
```sql
INSERT INTO domainlist_by_group (domainlist_id, group_id) VALUES (1, 1);
```
(the `domainlist_id` might be different for you, check with `SELECT last_insert_rowid();` after step 1)

#### Test
```text
192.168.0.101: dig +short blacklisted.com ---> 0.0.0.0          (blocked)
192.168.0.102: dig +short blacklisted.com ---> 103.224.182.245  (not blocked)
192.168.0.103: dig +short blacklisted.com ---> 103.224.182.245  (not blocked)
192.168.0.104: dig +short blacklisted.com ---> 0.0.0.0          (blocked)
```

#### Result
- `192.168.0.101` - Blocked as group 1 is associated to the new blacklisted domain.
- `192.168.0.102` - Not blocked as group 2 is not associated to the new blacklisted entry.
- `192.168.0.103` - Not blocked as group 3 is not associated to the new blacklisted entry.
- `192.168.0.104` - Blocked as not managed by any group - all blacklisted domains are used.

### Step
Remove default assignment to all clients not belonging to a group
```sql
DELETE FROM domainlist_by_group  WHERE domainlist_id = 1 AND group_id = 0;
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

- `192.168.0.101` - Blocked as group 1 is associated to the new blacklisted domain.
- `192.168.0.102` - Not blocked as group 2 is not associated to the new blacklisted entry.
- `192.168.0.103` - Not blocked as group 3 is not associated to the new blacklisted entry.
- `192.168.0.104` - Not blocked as we deleted the default assignment of this domain to untagged clients.

## Example 4: Whitelisting
**Task:** Add a domain that should be **white**listed for client `192.168.0.102` (group 2).

## Step 1
Add the domain to be unblocked
```sql
INSERT INTO domainlist (type, domain, comment) VALUES (0, 'doubleclick.net', 'Whitelisted for members of group 2');
```
#### Test
```text
192.168.0.101: dig +short doubleclick.net ---> 216.58.208.46  (not blocked)
192.168.0.102: dig +short doubleclick.net ---> 0.0.0.0        (blocked)
192.168.0.103: dig +short doubleclick.net ---> 0.0.0.0        (blocked)
192.168.0.104: dig +short doubleclick.net ---> 216.58.208.46  (not blocked)
```

#### Result
- `192.168.0.101` - Not blocked as group 1 is not associated to any adlist (as before).
- `192.168.0.102` - Blocked as group 2 is associated to all adlists but not to the new whitelist entry.
- `192.168.0.103` - Blocked as group 3 is associated to all adlists but not to the new whitelist entry.
- `192.168.0.104` - Not blocked as not managed by any group - all whitelisted domains are used.

### Step 2
Assign this domain to group 2
```sql
INSERT INTO domainlist_by_group (domainlist_id, group_id) VALUES (2, 2);
```
(the `domainlist_id` might be different for you, check with `SELECT last_insert_rowid();` after step 1)

#### Test
```text
192.168.0.101: dig +short doubleclick.net ---> 216.58.208.46  (not blocked)
192.168.0.102: dig +short doubleclick.net ---> 216.58.208.46  (not blocked)
192.168.0.103: dig +short doubleclick.net ---> 0.0.0.0        (blocked)
192.168.0.104: dig +short doubleclick.net ---> 216.58.208.46  (not blocked)
```

#### Result
- `192.168.0.101` - Not blocked as group 1 is not associated to any adlist (as before).
- `192.168.0.102` - Not blocked as group 2 is associated to the whitelist entry for this domain.
- `192.168.0.103` - Blocked as group 3 is associated to all adlists but not to the new whitelist entry.
- `192.168.0.104` - Not blocked as not managed by any group - all whitelisted domains are used.

### Step 3
Remove default assignment to all clients not belonging to a group
```sql
DELETE FROM domainlist_by_group  WHERE domainlist_id = 2 AND group_id = 0;
```
(the `domainlist_id` might be different for you, see above)

#### Test
```text
192.168.0.101: dig +short doubleclick.net ---> 216.58.208.46  (not blocked)
192.168.0.102: dig +short doubleclick.net ---> 216.58.208.46  (not blocked)
192.168.0.103: dig +short doubleclick.net ---> 0.0.0.0        (blocked)
192.168.0.104: dig +short doubleclick.net ---> 0.0.0.0        (blocked)
```

#### Result
- `192.168.0.101` - Not blocked as group 1 is not associated to any adlist (as before).
- `192.168.0.102` - Not blocked as group 2 is associated to the whitelist entry for this domain.
- `192.168.0.103` - Blocked as group 3 is associated to all adlists but not to the new whitelist entry.
- `192.168.0.104` - Blocked as we deleted the default assignment of this domain to untagged clients.
