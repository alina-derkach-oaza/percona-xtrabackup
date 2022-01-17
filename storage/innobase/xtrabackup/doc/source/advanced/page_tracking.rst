.. _page_tracking:

====================================
Page Tracking for Incremental Backup 
====================================

As of 8.0.18, the MySQL supports the page tracking functionality for `incremental backup <https://www.percona.com/doc/percona-xtrabackup/8.0/backup_scenarios/incremental_backup.html>`. 

To create an incremental backup the |mysqlbackup| looks for pages that have been modified since the last backup in the InnoDB data files and copies only changed pages. Since there is no need to scan all the pages in the database, the page tracking makes the incremental backup faster if the number of changed pages is not big.

Our tests show that if |1%| of data has been changed after the full backup of |100 GB|, the incremental backup takes only |30 seconds|. In comparison, without page tracking, the incremental backup takes |5 minutes|.

If the number of changed pages increases, the performance reduces. Our tests show, that with page tracking the incremental backup performs better than full scan up to |50%| the size of the server*. 

.. note::

   The result depends on the type of workload. For example, you either insert new pages or randomly update different pages in the database.

Prerequisites
-------------

To start using the page tracking functionality, do the following steps:

1. Install the |mysqlbackup| component and enable it on the server: 

  .. code-block:: bash

     $ INSTALL COMPONENT "file://component_mysqlbackup";

2. Check whether the |mysqlbackup| component is installed succesfuly:

  .. code-block:: bash

     $ SELECT COUNT(1) FROM mysql.component WHERE  component_urn='file://component_mysqlbackup';

Usage
-----

After the |component_mysqlbackup| is loaded and active on the server, enable the page tracking functionality for the full and incremental backups using the following option:  

``--page-tracking``

It serves the dual purpose:

* Reset page tracking at the start of the backup so that the next incremental backup can use page tracking.

* Use page tracking for incremental backup if page tracking data is available from the backup start checkpoint LSN.

Once the page tracking is enabled, the next incremental backup will use page tracking to find changed pages.

You can check the LSN value starting from which changed pages have been tracked using the following option:

  .. code-block:: bash

     $ SELECT mysqlbackup_page_track_get_start_lsn();

To stop page tracking use the following option:

  .. code-block:: bash

     $ SELECT mysqlbackup_page_track_set(false);

.. note::

   To use the aforementioned options for page tracking, run the ``BACKUP_ADMIN`` privilege.

First backup using page tracking
--------------------------------

For the very first backup using page tracking, Percona XtraBackup may have a  delay. You will see the following message: 

  .. code-block:: bash

     $ xtrabackup: pagetracking: Sleeping for 1 second, waiting for checkpoint lsn 17852922 to reach to page tracking start lsn 21353759

This delay is required to ensure that the page tracking start LSN is more than the backup end checkpoint LSN which is required for the next incremental backup. If the server page flushing rate is low or it uses a very big buffer pool, you can start page tracking well ahead of taking the first backup to ensure that page tracking LSN is more than the checkpoint LSN of the server. It is required for the first backup only.

Known issues with page tracking
-------------------------------

1. If the index was recently added to a table after the last LSN checkpoint and it was built in place using an exclusive algorithm, you may get a bad incremental backup using page tracking. Find more details in PS-8032. 

2. The server internally generated small data to maintain the page tracking data. This data can be purged before a full backup. Find more details in Bug#105406. 