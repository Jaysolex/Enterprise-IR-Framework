# Evidence Inventory — INC-XXXX

## Chain of Custody

All evidence must be hashed immediately upon collection.
Hash must be verified before and after any analysis.
Do not modify original evidence — work on copies only.

---

## Evidence Log

| Item # | Description | Type | Source System | Collected By | Date/Time | SHA256 Hash | Storage Location |
|--------|-------------|------|--------------|-------------|-----------|-------------|-----------------|
| E001 | | Memory dump | | | | | |
| E002 | | Event log | | | | | |
| E003 | | Network capture | | | | | |
| E004 | | Disk image | | | | | |
| E005 | | Registry hive | | | | | |
| E006 | | Prefetch files | | | | | |
| E007 | | Browser history | | | | | |
| E008 | | UAC collection | | | | | |

---

## Hashing Commands

```bash
# Linux
sha256sum /path/to/evidence > /path/to/evidence.sha256
sha256sum -c /path/to/evidence.sha256

# Windows PowerShell
Get-FileHash C:\IR\evidence.dmp -Algorithm SHA256
```

---

## Evidence Transfer Log

| Item # | From | To | Method | Date | Transferred By | Received By |
|--------|------|----|--------|------|---------------|------------|
| | | | | | | |

---

## Evidence Storage

| Storage Type | Location | Access Control | Encryption |
|-------------|---------|---------------|-----------|
| Primary | | | |
| Backup | | | |
| Cloud | | | |
