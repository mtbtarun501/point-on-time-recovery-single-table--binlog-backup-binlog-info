sudo mkdir -p /pointbackup/full
sudo chown -R mysql:mysql /pointbackup/full

sudo mkdir -p /pointbackup/binlogs

sudo mkdir -p /tmp/pitr_logs
sudo chown -R mysql:mysql /tmp/pitr_logs
==============================================================================================================

📦 STEP 2: Take Full XtraBackup

xtrabackup --backup \
  --target-dir=/pointbackup/full \
  --user=root \
  --password=password
==============================================================================================================
📌 STEP 3: Check Binlog Start Info


cat /pointbackup/full/xtrabackup_binlog_info
# Example output: mysql-bin.000012	157
[Write this down — it's your PITR starting point]

[INITITALLY BEFORE ANY TABLE UPDATE DELETE AND DROP]
{sudo cat /pointbackup/full/xtrabackup_binlog_info
mysql-bin.000014	157}
==============================================================================================================
                          [TAKEN FULL BACKUP AT MORNING AND AT 12PM I DROPED SINGLE TABLE IN A DATABASAE]
                          
 STEP 4: Simulate DB Activity After Backup


-- Switch to your test DB
USE recovery_test;

-- Recreate table (if not already)
CREATE TABLE test_table (
  id INT NOT NULL AUTO_INCREMENT,
  message VARCHAR(255),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id)
) ENGINE=InnoDB;

-- Insert 20 rows
INSERT INTO test_table (message)
SELECT CONCAT('msg_', n)
FROM (
  SELECT 1 AS n UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5
  UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9 UNION ALL SELECT 10
  UNION ALL SELECT 11 UNION ALL SELECT 12 UNION ALL SELECT 13 UNION ALL SELECT 14 UNION ALL SELECT 15
  UNION ALL SELECT 16 UNION ALL SELECT 17 UNION ALL SELECT 18 UNION ALL SELECT 19 UNION ALL SELECT 20
) AS nums;

-- Delete 5 rows
DELETE FROM test_table WHERE id IN (20,21,22,23,24,25);

-- Update 5 rows
UPDATE test_table SET message = CONCAT(message, '_UPDATED') WHERE id IN (16,17,18,19,20);

-- DROP the table
DROP TABLE test_table;
==============================================================================================================
                                  [PREPARE THE XTRABACKUP FILE]

🔄 STEP 5: Prepare the Backup (with export)


[xtrabackup --prepare --export --target-dir=/fullbackups/full]
==============================================================================================================


🧱 STEP 6: Recreate and DISCARD the Table


CREATE TABLE test_table (
  id INT NOT NULL AUTO_INCREMENT,
  message VARCHAR(255),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id)
) ENGINE=InnoDB;

ALTER TABLE test_table DISCARD TABLESPACE;
==============================================================================================================

STEP 7: Copy .ibd + .cfg Files to MySQL Datadir

sudo cp /pointbackup/full/recovery_test/test_table.{ibd,cfg} /var/lib/mysql/recovery_test/
sudo chown mysql:mysql /var/lib/mysql/recovery_test/test_table.*
==============================================================================================================

📂 STEP 8: IMPORT TABLESPACE

ALTER TABLE test_table IMPORT TABLESPACE;
✅ Now you have the table as it was at full backup time
==============================================================================================================


**cat /pointbackup/full/xtrabackup_binlog_info

sudo mysqlbinlog --base64-output=DECODE-ROWS --verbose \
  --start-position=157 \
  /var/lib/mysql/mysql-bin.000012 > /tmp/pitr_logs/binlog_replay.sql
🧠 What This Does:
Reads the binary log starting from position 157

Decodes row-based operations (inserts/deletes/updates)

Writes a human-readable SQL version to /tmp/pitr_logs/binlog_replay.sql**

          ------------------------------------------------------------------------
          sudo mysqlbinlog --base64-output=DECODE-ROWS --verbose \
  --start-position=157 \
  /pointbackup/full/mysql-bin.000014 | sudo tee /tmp/pitr_logs/binlog_after_backup.sql > /dev/null
==============================================================================================================


  ✅ NEXT: Search for DROP/DELETE/UPDATE
Once the file is generated, search for:


grep -i -A 10 "DROP TABLE" /tmp/pitr_logs/binlog_after_backup.sql
grep -i -A 10 "DELETE FROM" /tmp/pitr_logs/binlog_after_backup.sql
grep -i -A 10 "UPDATE" /tmp/pitr_logs/binlog_after_backup.sql
That will help us find:

The DROP position to STOP BEFORE replaying

Any other destructive actions to roll back safely



