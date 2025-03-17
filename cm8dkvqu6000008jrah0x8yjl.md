---
title: "pt-table-sync"
seoTitle: "Percona pt table sync"
seoDescription: "Fix database inconsistency and data discrepancies "
datePublished: Mon Mar 17 2025 21:30:02 GMT+0000 (Coordinated Universal Time)
cuid: cm8dkvqu6000008jrah0x8yjl
slug: pt-table-sync
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/pGHRUV5tPVI/upload/741a29b51a73ca443321ddad7d69bca5.jpeg
tags: mysql, percona, pt-table-sync

---

I recently had a situation when a lot of rows from one of our replicas was missing.  
One of the solutions that came in mind is to use the snapshot of the source and reconfigure it as replica. While this would have worked, it seemed like an extreme approach for what might be a localized issue.

After digging into the logs after a while, the problem seems to be only isolated to a single table. So, I tried `pt-table-sync`.

### What is pt-table-sync?

**pt-table-sync** is a command-line utility that helps synchronize data across different MySQL servers. It can be used in various scenarios, such as:

* Synchronizing data between a **source and its replica**
    
* Resolving inconsistencies between two **source servers**
    
* Comparing and fixing differences between **any two MySQL data sources**
    

The synchronization command looks like this:

```bash
pt-table-sync --execute h=x.x.x.x,D=database,t=table,u=user,p=pass h=localhost,u=user,p=pass
```

Here’s what each parameter does:

* `h=x.x.x.x` → The **host** of the source database (where data is correct)
    
* `D=database` → The **database name**
    
* `t=table` → The **table name** that needs synchronization
    
* `u=user` / `p=pass` → **MySQL credentials** for authentication
    
* `h=localhost` → The **replica server** where data needs to be fixed
    

`pt-table-sync` generally sync data from the replica to the source, i.e. from the 2nd data source to the first as changes to the replica are usually the source of the problems in the first place. i.e. the changes made on the source will be replicated down the replica via the normal replication process.

### How pt-table-sync Works

The synchronization process in **pt-table-sync** consists of three main operations:

* **UPDATE** →For stale data NOOP operation is triggered on source, that only affects the replica.
    
* **DELETE** → DELETE statements on the source for rows that don't exist there but exist in replica.
    
* **INSERT** → For missing data, that exist on source but not on replica. It’s retriggered so it can pass via binary logs
    

By default, **pt-table-sync** generates and executes these statements to bring the replica in sync with the source. If you want to preview these operations before executing them, you can use:

```bash
pt-table-sync --print h=x.x.x.x,D=database,t=table,u=user,p=pass h=localhost,u=user,p=pass
```

This prints out the SQL statements that would be executed, allowing you to review changes before applying them.