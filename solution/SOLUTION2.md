## Problem 2 – Backup and recovery

### Risk
- **DB:** One daily backup, PITR disabled. Effective RPO is up to 24 hours versus the 15 minutes target.
- **Files:** No soft delete or versioning, so deleted files are permanently lost. Restoring an old DB backup can leave records pointing to missing files.
- **Untested recovery:** A restore has never been done, so we cannot know if the 2 hours RTO is achievable (invalid backups, permission errors, no runbook)

### Changes before the incident
- Enable PITR with 7 days transaction log retention (RPO of minutes).
- Enable soft delete on the bucket with 7-day retention.
- Enable audit logs (GCS Data Access, pgAudit) and alert on large deletions.
- Least privilege: no manual deletions in production, only a temporary, approved and logged emergency access.
- Write a recovery runbook and run a restore drill every quarter.
- Apply these changes in a maintenance window (pgAudit requires a DB restart).

### Recovery steps
1. Stop writes: API in maintenance mode, worker scaled to 0. Do not modify the original instance.
2. Use the audit logs to find exactly when the deletion began and what was deleted.
3. PITR clone to a new instance, just before the deletion.
4. Restore only the deleted records to production and the deleted files from soft delete. Fallback: switch the app to the clone (~15 min loss, within RPO).
5. Verify with a script that every restored record points to an existing file. On fallback, update the DB host in Secret Manager and restart the pods.
6. Run a smoke test, reopen the service, monitor error rates and remove the delete permissions that caused the issue.

### Verification
Quarterly restore drill on a non-production clone: measure data loss after PITR (< 15 min), time the full runbook (target under 1 hour), restore a file deleted 6 days earlier, and check the script reports 0 missing files.

*Proposed only – not tested.*