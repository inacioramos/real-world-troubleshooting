# 🗄️ Case Study: Production Database Corruption Recovery (Firebird SQL)

<p align="center">
  <img src="https://img.shields.io/badge/Status-Resolved-success?style=for-the-badge" alt="Status Resolved">
  <img src="https://img.shields.io/badge/Severity-Critical-critical?style=for-the-badge" alt="Severity Critical">
  <img src="https://img.shields.io/badge/Category-Database_Administration-orange?style=for-the-badge" alt="Category Database Administration">
</p>

> [!NOTE]
> 📝 **Contexto em Português:** Este caso documenta a recuperação crítica de um banco de dados relacional Firebird em ambiente de produção que sofria com corrupção estrutural e erros de *Access Violation*. A resolução envolveu o isolamento das páginas corrompidas e comandos avançados de manutenção (`gfix`, `gbak`) para garantir a integridade dos dados e mitigar o tempo de inatividade do cliente.

## 📋 Overview

This technical case study details a critical infrastructure intervention to salvage a corrupted relational database operating in a live commercial production environment. It highlights database maintenance proficiency, data rescue techniques, and disaster recovery execution under operational constraints.

* **Database Engine:** Firebird SQL
* **Error Token:** `Access Violation` / Database Corruption Flags
* **Impact:** **Critical** 🚨 (Potential total loss of historical business data and operational downtime)

---

## 🔍 Problem Description

During standard operations under high transactional load, the client's commercial automation system experienced abrupt crashes, throwing unhandled memory allocation errors and locking the database file handles.

Initially, any attempt to verify parameters or test connections triggered an OS-level file lock exception:

![Facilite Database IO Lock Error](facilite-io-error.png)

The system flagged that `FACILITE.FDB` was indefinitely trapped
---

## 👣 Steps Taken for Investigation & Diagnosis

To isolate the failure without risking further data degradation, the following sandbox diagnostic protocol was executed:

1. **Environment Isolation:** Made a secure binary copy of the corrupted `.fdb` file to an isolated technical staging environment.
2. **Integrity Validation:** Ran validation passes using the Firebird command-line tool (`gfix`) to check the internal structure.
3. **Log Analysis:** Evaluated the database engine logs to locate the exact corrupted memory pages and corrupted relational headers causing the system to throw the *Access Violation* exception.

---

## 📊 Discovered Results

### ❌ Actual Result (The Breakdown)
The structural degradation of the database was severe. When using database administration suites like **IB Expert** and **IBOConsole** to force sweeps or backup pipelines, the engine crashed entirely due to structural page corruption.

1. **Engine Consistency Check Failure:**

![IB Expert Firebird Consistency Bugcheck](ibexpert1-bugcheck.png)

The database server triggers a low-level engine panic: `internal Firebird consistency check (can't continue after bugcheck)`.

2. **Memory Exception upon Indexing:**
   
![IBOConsole Delphi Access Violation Exception](iboconsole-access-violation.png)

While processing millions of rows inside transactional tables (such as `COMPLITCOMANDAPRODUTO`), the application hits a corrupted index pointer, forcing the operating system to throw a fatal memory read/write exception: `Access violation at address 772A12D8 in module 'ntdll.dll'`.

### ✅ Expected Result
The database must maintain full transactional integrity (ACID properties), allowing applications to perform CRUD operations smoothly without encountering low-level page allocation failures.

---

## 🧠 Root Cause Analysis (RCA)

The *Access Violation* errors were triggered because the internal database pointers were referencing bad or unreadable data pages (likely caused by a sudden hardware power failure or improper system shutdown on the server). When the application attempted to index or load data residing on those broken fragments, the Firebird server process suffered an unhandled memory read exception.

---

## 🛠️ Recovery & Resolution Applied

A multi-stage disaster recovery pipeline was deployed using low-level Firebird command-line utilities to isolate corrupt fragments and rebuild the core architecture directly on the staging environment:

### 1. Corruption Mapping & Database Repair (`gfix`)
First, the engine was forced to inspect the file headers, targeting internal pages. The utility exposed a severe physical checksum failure on a specific block:

```bash
gfix -m -user SYSDBA -pass masterkey "C:\Sinco\BackupAtualizacoes\BD\FACILITE.FDB"
```

![Firebird Gfix Bad Checksum Page Isolation](firebird-checksum-error.png)

As shown in the terminal layout above, the utility successfully isolated the structural corruption flag pointing to the specific internal database page **7742122**.

Following the initial isolation, a secondary diagnostic sweep was executed to map the absolute volume of structural failures inside the storage engine layers:

![Firebird Gfix Validation Summary Output](firebird-validation-summary.png)

The diagnostic validation summary reported **56 record-level errors**, **1 data page error**, and **1 database page error**. These specific fragments were systematically marked to safeguard the remaining healthy database structures.

---

### 2. Metadata Extraction & Clean Rebuild (`gbak`)

With the corruption sectors contained, a metadata backup structure was initiated. Initial syntax constraints under live production conditions generated standard parameter execution errors:

![Firebird Gbak CLI Syntax Error Attempt](firebird-gbak-syntax-error.png)

After adjusting the parameters to bypass the broken sectors (`-g -ignore`), the extraction layer successfully read through the database blocks, generating a clean backup snapshot (`.fbk`) and restoring a brand new container.

The execution sequence completed successfully by processing, activating, and cleanly rebuilding every core indexing schema, structural table, and relational constraint:

![Firebird Gbak Database Recovery Success](firebird-gbak-success.png)

The automated operational pipeline finished cleanly, with the database engine logging its absolute clearance confirmation flag: `gbak:finishing, closing, and going home`.

> **Final Verification Status:** The newly compiled database file was validated with a subsequent `gfix -v`, showing **100% data integrity** and zero operational losses. The client system returned to production with total stability.

---

## 🎯 Hard Skills Demonstrated in this Case

* **Disaster Recovery (DR) & Data Resiliency:** Executing low-level recovery tools under a strict operational deadline to prevent data loss.
* **Root Cause Analysis on Database Layers:** Tracing memory violation errors (*Access Violation*) back to physical database structure inconsistencies.
* **Database Maintenance Commands:** Practical proficiency with specialized engine tools (`gfix`, `gbak`) for structural verification and rebuilding.
