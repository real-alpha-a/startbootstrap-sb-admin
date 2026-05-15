STEP 1 — Create Directories
mkdir -p ~/mongo-recovery/{db,logs,run,dumps,recovered,tmp}

Go inside:

cd ~/mongo-recovery
STEP 2 — Download MongoDB Binary

IMPORTANT:
Use same MongoDB major version as production.

Example MongoDB 3.2

Download:

wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1604-3.2.22.tgz

If wget unavailable:

curl -O https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1604-3.2.22.tgz
STEP 3 — Extract MongoDB
tar -xvzf mongodb-linux-x86_64-ubuntu1604-3.2.22.tgz

Rename:

mv mongodb-linux-* mongodb

Now binaries are here:

~/mongo-recovery/mongodb/bin/
STEP 4 — Verify MongoDB

Run:

~/mongo-recovery/mongodb/bin/mongod --version
STEP 5 — Create Config File

Create config:

nano ~/mongo-recovery/mongod.conf

Paste:

systemLog:
  destination: file
  path: /home/YOUR_USERNAME/mongo-recovery/logs/mongod.log
  logAppend: true

storage:
  dbPath: /home/YOUR_USERNAME/mongo-recovery/db

processManagement:
  fork: true
  pidFilePath: /home/YOUR_USERNAME/mongo-recovery/run/mongod.pid

net:
  bindIp: 127.0.0.1
  port: 27018

IMPORTANT:
Replace:

YOUR_USERNAME

with actual Linux username.

Find username:

whoami
STEP 6 — Start MongoDB

Run:

~/mongo-recovery/mongodb/bin/mongod \
  --config ~/mongo-recovery/mongod.conf

If successful:

forked process: xxxx
child process started successfully
STEP 7 — Verify Running

Check process:

ps -ef | grep mongod

Check port:

netstat -plnt | grep 27018

OR:

ss -plnt | grep 27018
STEP 8 — Connect Mongo Shell

Run:

~/mongo-recovery/mongodb/bin/mongo --port 27018

Inside shell:

show dbs
STEP 9 — Copy Dump

Put dump inside:

~/mongo-recovery/dumps

Example:

~/mongo-recovery/dumps/mydb/users.bson
STEP 10 — Restore Collection

Example:

~/mongo-recovery/mongodb/bin/mongorestore \
  --port 27018 \
  --db mydb \
  --collection users \
  ~/mongo-recovery/dumps/mydb/users.bson
STEP 11 — Check for Corruption

Run BSON check:

~/mongo-recovery/mongodb/bin/bsondump \
  ~/mongo-recovery/dumps/mydb/users.bson

If corrupted:

Invalid BSONObj size
STEP 12 — Validate Collection

Connect shell:

~/mongo-recovery/mongodb/bin/mongo --port 27018

Then:

use mydb

db.runCommand({
  validate: "users",
  full: true
})
STEP 13 — Stop MongoDB
~/mongo-recovery/mongodb/bin/mongo \
  --port 27018 \
  --eval "db.shutdownServer()"
If MongoDB Fails to Start

Most common issue:
Old MongoDB binary incompatible with modern Linux.

Example error:

libcrypto.so missing
GLIBC version missing

Then:

use newer compatible Mongo version
OR use older server/VM
OR use static binaries

Tell me exact error if that happens.

Recommended Mongo Versions
Production Version	Recovery Version
2.6	2.6.x
3.0	3.0.x
3.2	3.2.x
3.4	3.4.x

Use same major version first.

VERY IMPORTANT

Do NOT:

use port 27017
use /var/lib/mongodb
run mongod --repair
overwrite original dump
Best Next Step

Now do these:

download Mongo
start isolated Mongo
share:
mongo version
dump structure
any startup error

Then I’ll help you:

restore safely
identify corrupted collections
salvage good documents
recover around bad BSON records
migrate to modern MongoDB safely
unable to establish SSL connection

That error usually comes from MongoDB version vs shell/client mismatch or SSL/TLS mismatch, not from your data.

Since you’re running this manually (no sudo, old Ubuntu, old Mongo), the most common cause is:

You started mongod, but your mongo shell is trying to use SSL (or a newer protocol)

Let’s fix it step by step.

1. First confirm MongoDB is actually running

Run:

ps -ef | grep mongod

Then check port:

ss -plnt | grep 27018

If nothing is listening → Mongo didn’t start correctly.

2. Most common issue: wrong client version

If you downloaded:

newer mongo shell (4.x/5.x/6.x)
but server is 3.2 / 3.0

You will get errors like:

unable to establish SSL connection
handshake failures
connection reset
3. Fix: ALWAYS use matching mongo shell

Inside your extracted folder:

~/mongo-recovery/mongodb/bin/mongo --version

If this is missing or different, you are likely using system mongo.

Check:

which mongo

If output is like:

/usr/bin/mongo

👉 THIS is the problem.

4. Force correct client (VERY IMPORTANT)

Always use full path:

~/mongo-recovery/mongodb/bin/mongo --port 27018

NOT just:

mongo --port 27018   ❌
5. Disable SSL/TLS explicitly (critical fix)

Old MongoDB versions do NOT need SSL.

Try:

~/mongo-recovery/mongodb/bin/mongo --port 27018 --ssl false

If that fails:

~/mongo-recovery/mongodb/bin/mongo --port 27018 --sslDisabledProtocols 'all'
6. Check mongod startup logs (VERY IMPORTANT)

Open:

cat ~/mongo-recovery/logs/mongod.log | tail -50

Look for:

SSL enabled?
TLS required?
bind errors?
7. Force mongod WITHOUT SSL (safe config)

Edit config:

nano ~/mongo-recovery/mongod.conf

Ensure this section exists:

net:
  bindIp: 127.0.0.1
  port: 27018
  ssl:
    mode: disabled

If ssl block exists → REMOVE it.

8. Restart Mongo cleanly

Stop old process:

pkill mongod

Start again:

~/mongo-recovery/mongodb/bin/mongod --config ~/mongo-recovery/mongod.conf
9. Quick isolation test (no SSL, direct)

Try this minimal start:

~/mongo-recovery/mongodb/bin/mongod \
  --dbpath ~/mongo-recovery/db \
  --port 27018 \
  --bind_ip 127.0.0.1 \
  --nojournal

Then connect:

~/mongo-recovery/mongodb/bin/mongo --port 27018
10. If still failing → 2 likely root causes
Case A: Wrong Mongo version mismatch

Example:

mongod = 3.2
mongo shell = 4.x

👉 FIX: use same version package for both

Case B: system mongo interfering

Check:

which mongod
which mongo

If ANY are:

/usr/bin/*

you are mixing installations.

11. Fast diagnostic checklist

Run and share output:

~/mongo-recovery/mongodb/bin/mongod --version
~/mongo-recovery/mongodb/bin/mongo --version
which mongo
which mongod

Also:

cat ~/mongo-recovery/logs/mongod.log | tail -50
Key takeaway

👉 This is NOT a data corruption issue yet
👉 This is almost always:

wrong mongo shell
SSL mismatch
version mismatch

If you paste your:

Mongo version
exact command you ran
log tail

I can pinpoint the exact fix in one shot.

wget 

If you just typed wget and are stuck, here’s exactly how to use it properly for your MongoDB manual install.

1. Basic wget syntax
wget <URL>

Example:

wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1604-3.2.22.tgz

This will download the file into your current directory.

2. If wget is not found

Run:

wget --version

If you get:

command not found

Then use curl instead:

curl -O https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1604-3.2.22.tgz
3. Download MongoDB manually (recommended for your case)

Go to your recovery folder first:

cd ~/mongo-recovery

Then download:

Example MongoDB 3.2 (safe for old systems)
wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1604-3.2.22.tgz
4. If download is slow or fails

Use retry option:

wget -c https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1604-3.2.22.tgz

Or increase retries:

wget --tries=10 --timeout=30 https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1604-3.2.22.tgz
5. After download

Extract:

tar -xvzf mongodb-linux-x86_64-ubuntu1604-3.2.22.tgz
6. If your question is actually “wget is failing”

Tell me the exact error like:

SSL certificate problem
connection refused
404 not found
timeout

Because fixes differ.

If you want, I can also give you:

exact MongoDB version link for your production DB
offline download strategy (if server has no internet)
alternative SCP-based transfer method from your laptop/other server
my current mongo db version is 2.0.4 

Got it — MongoDB 2.0.4 is very old (2012 era). This changes your recovery approach a lot.

That version explains your error:

Invalid BSONObj size -286331154 (0xEEEEEEEE)

That pattern + Mongo 2.0 usually means:

data corruption on disk OR
broken memory pages OR
app writing invalid BSON OR
journal inconsistencies (very common in 2.0 era)
⚠️ Critical Reality First

MongoDB 2.0.4 is NOT compatible with modern MongoDB tools

So:

❌ Do NOT use MongoDB 3.x/4.x/6.x to restore directly
❌ Do NOT use modern mongorestore
❌ Do NOT mix binaries

You MUST use:

✔ MongoDB 2.0.x tools only

Otherwise you will get:

SSL errors
BSON parsing failures
silent corruption during restore
✔ Your Correct Recovery Strategy (NO sudo, NO Docker)

We will do:

Step 1: Download MongoDB 2.0 tools
Step 2: Run isolated mongod in home directory
Step 3: Attach dump safely
Step 4: Identify corrupted collections
Step 5: Extract only good data
STEP 1 — Download MongoDB 2.0.4 binaries

You need legacy archive.

Try this:

cd ~/mongo-recovery
wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-2.0.4.tgz

If this fails (likely), use fallback mirror:

👉 MongoDB legacy archive:
MongoDB Legacy Archive

STEP 2 — Extract MongoDB 2.0
tar -xvzf mongodb-linux-x86_64-2.0.4.tgz
mv mongodb-linux-* mongodb

Check:

~/mongo-recovery/mongodb/bin/mongod --version

You should see:

db version v2.0.4
STEP 3 — Start MongoDB (IMPORTANT)

Run without SSL, without fork confusion

~/mongo-recovery/mongodb/bin/mongod \
  --dbpath ~/mongo-recovery/db \
  --port 27018 \
  --bind_ip 127.0.0.1 \
  --nojournal

Why --nojournal?

Mongo 2.0 journaling is unstable
helps recovery
STEP 4 — Connect properly

⚠️ MUST use same version shell:

~/mongo-recovery/mongodb/bin/mongo --port 27018

If you use system mongo → SSL error happens.

STEP 5 — Test dump structure

Check:

ls ~/mongo-recovery/dumps

Expected:

mydb/users.bson
mydb/orders.bson
STEP 6 — Restore ONE collection first (VERY IMPORTANT)
~/mongo-recovery/mongodb/bin/mongorestore \
  --port 27018 \
  --db mydb \
  --collection users \
  ~/mongo-recovery/dumps/mydb/users.bson
STEP 7 — If your error appears here

If you see:

Invalid BSONObj size -286331154

That means:
👉 corruption is inside BSON file itself

NOT Mongo server problem

STEP 8 — Identify broken records

Use:

~/mongo-recovery/mongodb/bin/bsondump \
  ~/mongo-recovery/dumps/mydb/users.bson > /tmp/users.json

If it crashes:

last line = corruption point
STEP 9 — Salvage strategy (IMPORTANT)

Mongo 2.0 does NOT have good repair tools.

So we do:

Option A (best)

Skip corrupted document blocks using custom parser

Option B

Split file into chunks and restore partial data

Option C

Extract only valid prefix until corruption

STEP 10 — Why your system broke (very likely)

With MongoDB 2.0.4, your setup likely has:

1. Memory corruption pattern (0xEEEEEEEE)
uninitialized memory write
disk overwrite
JVM/native buffer issue
2. Email attachment ingestion problem

Your pipeline:

Outlook → attachment → parse → store in Mongo

Common issue:

oversized document (>16MB equivalent logical issue even in old versions)
binary stored as string/base64
partial write crash
3. Ubuntu 12 + old disk issues
ext3/ext4 old bugs
unclean shutdowns
power loss corruption
🚨 Important Warning

Do NOT run:

mongod --repair

on this version unless you are okay losing data.

✔ What I need from you to go deeper

Send me:

ls ~/mongo-recovery/dumps

size of BSON files:

ls -lh dumps/mydb/
exact error when running mongorestore
If you want next step help, I can guide you on:
extracting only valid records from corrupted BSON
building Python recovery script for Mongo 2.0 BSON format
identifying which collection is broken without restoring
rebuilding full DB safely
migrating Mongo 2.0 → modern MongoDB (step-by-step safe path)

Just tell me 👍
