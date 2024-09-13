---
title: "Schema change without locking."
seoTitle: "Schema change without locking."
seoDescription: "Performing schema changes on production databases comes with significant overhead, such as locking entire tables and potentially causing replication lag."
datePublished: Fri Sep 13 2024 14:03:24 GMT+0000 (Coordinated Universal Time)
cuid: cm10sgs3p000209kwfhgi5hfz
slug: schema-change-without-locking
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/EqC7y72PLAY/upload/01e09f0ffaead64fd1f1fe0491277cdd.jpeg
tags: mysql, databases, percona

---

Performing schema changes on production databases comes with significant overhead, such as locking entire tables and potentially causing replication lag in master-slave architectures. For MySQL and its variants, there's a powerful tool that mitigates these issues: pt-online-schema-change.

## What is pt-online-schema-change?

pt-online-schema-change is a tool that alters a table's structure without blocking reads or writes, making it ideal for use in production environments.

## How It Works

1. **Table Copying**: It creates an empty copy of the table with the desired alterations.
    
2. **Data Migration**: It copies rows from the original table to the new one in small, manageable chunks.
    
3. **Real-time Updates**: During the copy process, any modifications to the original table are reflected in the new table using triggers.
    
4. **Atomic Swap**: Once copying is complete, it uses an atomic `RENAME TABLE` operation to switch the old and new tables.
    
5. **Cleanup**: Finally, it drops the original table.
    

## Key Features and Safety Measures

* **Chunk-based Processing**: Data is copied in small chunks, optimized for performance (configurable with `--chunk-time`).
    
* **Trigger Mechanism**: Ensures data consistency during the migration process.
    
* **Foreign Key Handling**: Supports methods to update foreign key references after the schema change.
    
* **Safety Checks**:
    
    * Requires a PRIMARY KEY or UNIQUE INDEX in most cases.
        
    * Detects and respects replication filters.
        
    * Pauses operations if replicas lag behind.
        
    * Monitors server load and can pause or abort if thresholds are exceeded.
        
    * Sets conservative lock wait timeouts to minimize disruption.
        
    * Careful handling of foreign key constraints.
        

## Considerations

* By default, it doesn't modify the table unless the `--execute` option is specified.
    
* It may rename foreign keys and indexes slightly to avoid naming collisions.
    
* Existing triggers on the table will prevent the tool from working.
    

## Example Usage

```bash
#!/bin/bash

USERNAME="root"
PASSWORD="your_password_here"
HOST="localhost"
DATABASE="critical_db"
TABLE="critical_table"
ALTER_STATEMENT="ADD COLUMN guest VARCHAR(45), ADD COLUMN name VARCHAR(100)"

if [[ "$1" == "--execute" ]]; then
    EXECUTE=true
else
    EXECUTE=false
fi

echo "Performing dry run..."
pt-online-schema-change \
    --user="$USERNAME" \
    --password="$PASSWORD" \
    --host="$HOST" \
    D="$DATABASE",t="$TABLE" \
    --alter="$ALTER_STATEMENT" \
    --preserve-triggers \
    --dry-run

if [ "$EXECUTE" = true ]; then
    echo "Applying changes..."
    pt-online-schema-change \
        --user="$USERNAME" \
        --password="$PASSWORD" \
        --host="$HOST" \
        D="$DATABASE",t="$TABLE" \
        --alter="$ALTER_STATEMENT" \
        --preserve-triggers \
        --execute
fi
```

To use this script:

1. Save it to a file (e.g., `alter_`[`table.sh`](http://table.sh))
    
2. Make it executable: `chmod +x alter_`[`table.sh`](http://table.sh)
    
3. Run a dry run: `./alter_`[`table.sh`](http://table.sh)
    
4. Execute the changes: `./alter_`[`table.sh`](http://table.sh) `--execute`
    

## Conclusion

pt-online-schema-change offers a robust solution for performing schema changes in production environments, minimizing downtime and reducing the risks associated with traditional ALTER TABLE operations.