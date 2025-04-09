## Task 12: Performance Tuning, Snapshot Backup, and Recovery

**Objectives:**
- Optimize performance through query adjustments and index configuration.
- Monitor performance using Elasticsearch’s built‑in monitoring tools and Python scripts.
- Implement backup (snapshot) and recovery procedures.
- Stress test and document performance improvements.

**Lab Steps:**

1. **Performance Monitoring and Profiling:**  
   - Use Kibana’s “Monitoring” app to observe key metrics.
   - Repeat queries using the profile API as shown in Task 8.
   - Write a Python script (as in Task 8) that logs query timings.
2. **Index Tuning Experiments:**  
   - Adjust settings such as `refresh_interval` and `merge policies` on an existing index.
   - Use the Reindex API to simulate schema changes:
     ```json
     POST /_reindex
     {
       "source": { "index": "products" },
       "dest": { "index": "products_v2" }
     }
     ```
3. **Snapshot and Restore:**  
   - **Setup a Snapshot Repository:**
     ```json
     PUT /_snapshot/my_backup
     {
       "type": "fs",
       "settings": {
         "location": "/mount/backups/my_backup",
         "compress": true
       }
     }
     ```
   - **Take a Snapshot:**
     ```json
     PUT /_snapshot/my_backup/snapshot_1?wait_for_completion=true
     {
       "indices": "products,articles",
       "ignore_unavailable": true,
       "include_global_state": false
     }
     ```
   - **Simulate Data Loss and Recovery:**  
     Delete an index:
     ```http
     DELETE /products
     ```
     Restore the snapshot:
     ```json
     POST /_snapshot/my_backup/snapshot_1/_restore
     {
       "indices": "products",
       "include_global_state": false
     }
     ```
4. **Reflection and Documentation:**  
   – Record performance benchmarks before and after tuning.  
   – Summarize best practices for backup and disaster recovery.
