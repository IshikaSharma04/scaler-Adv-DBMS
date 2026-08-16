# Advanced DBMS Lab Task

**Role Number:** 10352  
**Name:** Ishika Sharma

---

## 1. SQLite3 Exploration

First, I created a database file named `sample.db` and wrote a quick python script to fill a `users` table with 1,000,000 rows so I'd have a good amount of data to test with.

**File Size:**
When I ran `ls -lh sample.db`, the output showed the file size was around 43 MB.

**Page Size:**
I ran `sqlite3 sample.db "PRAGMA page_size;"` and got `4096` as the output, which means the default page size is 4 KB.

**Page Count:**
I ran `sqlite3 sample.db "PRAGMA page_count;"` and it returned `10894`.

### Playing around with `mmap_size`

I checked the default mmap size (`sqlite3 sample.db "PRAGMA mmap_size;"`) and it was 0, meaning it was disabled. 

Then I changed it to 256 MB by running:
`sqlite3 sample.db "PRAGMA mmap_size=268435456; PRAGMA mmap_size;"`

### Query Times

I timed a basic `SELECT *` query to see if mmap made a difference.

Without mmap enabled:
`time sqlite3 sample.db "PRAGMA mmap_size=0; SELECT * FROM users;" > /dev/null`
It took about 0.339 seconds.

With mmap enabled (256 MB):
`time sqlite3 sample.db "PRAGMA mmap_size=268435456; SELECT * FROM users;" > /dev/null`
It took about 0.322 seconds.

So turning on mmap did speed up the query a little bit. I also used `ps aux | grep sqlite` in another terminal tab while the queries were running to see the process in action.

---

## 2. PostgreSQL (PSQL) Setup

I used Homebrew to install PostgreSQL (`brew install postgresql`), started the background service, and created a `sampledb` database. To make the comparison fair, I filled a `users` table with 1,000,000 rows using Postgres's `generate_series()` function.

**Page Size:**
`psql -d sampledb -c "SHOW block_size;"`
Output: `8192` (so Postgres uses 8 KB blocks).

**Page Count:**
I made sure to run `VACUUM ANALYZE users;` first to update the stats, then ran:
`psql -d sampledb -c "SELECT relpages FROM pg_class WHERE relname = 'users';"`
Output: `14286`

**Query Time:**
`time psql -d sampledb -c "SELECT * FROM users;" > /dev/null`
It took 3.572 seconds.

---

## 3. Comparison & My Observations

Here is a quick summary of the differences I noticed between SQLite and PostgreSQL for 1 million rows:

* **Page Size:** SQLite uses 4 KB (4096 bytes) while Postgres uses 8 KB (8192 bytes).
* **Page Count:** SQLite had 10,894 pages, while Postgres had 14,286 pages.
* **Disk Space:** SQLite took up roughly 43 MB, but Postgres took up way more space (around 111 MB).
* **Speed:** SQLite was much faster for the `SELECT *` dump (0.339s compared to 3.572s for Postgres).

**Why did this happen?**

1. Postgres takes up a lot more disk space because it has to store extra hidden system columns for its MVCC implementation (like transaction IDs) for every single row. This adds a lot of overhead compared to SQLite's simpler storage model.
2. The query was faster in SQLite because it runs directly inside the same process. Postgres is a client-server system, so taking all that data and sending it over a local socket to print to the terminal adds a huge amount of overhead, which explains the big time difference.
3. Lastly, the `mmap` experiment showed that memory mapping lets SQLite read data straight from memory instead of relying on standard OS read calls, which gives a nice little performance boost. Postgres handles this differently with its own shared buffers anyway.
