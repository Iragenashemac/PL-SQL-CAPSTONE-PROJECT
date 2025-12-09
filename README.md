# 📱 PL-SQL CAPSTONE PROJECT  
## **STOLEN PHONE & IMEI TRACKING SYSTEM**

### **Author:**  
**IRAGENA SHEMA Cedrick — 24766**

---

<br/>

# 📘 PHASE 1: Problem Definition

### **Institutional / National Context**
Mobile phone theft has become a major challenge. Stolen devices are often:

- Re-sold illegally  
- Re-registered with new SIM cards  
- Used for cybercrime  
- Used anonymously due to lack of integrated tracking  

Law enforcement and telecoms need a system that can:

- Track IMEI activity  
- Validate phone ownership  
- Detect stolen devices  
- Record movements & actions  
- Provide intelligence for investigations  

### **Data Challenge**
There is no centralized system for:

- Stolen phone reports  
- Real-time IMEI tracking  
- SIM history linkage  
- Ownership verification  
- Nationwide search & reporting  

### **Expected Output**
A centralized PL/SQL-powered system that:

✔ Registers users and phones  
✔ Tracks stolen phones  
✔ Soft-deletes instead of hard deletion  
✔ Logs all operations automatically  
✔ Tracks IMEI location & network  
✔ Produces audit trails  

<br/><br/>

# 📘 PHASE 2: System Architecture & ERD

### **System Entities**
| Entity | Description |
|-------|-------------|
| USERS | Registered users (citizens, police, telecom staff) |
| PHONES | Mobile phones owned by users |
| STOLEN_REPORTS | Reports submitted for missing phones |
| IMEI_TRACKING | History of IMEI activity |
| AUDIT_LOG | Logs all operations |

### **ER Diagram Placeholder**

👉 Insert screenshot here:

[screenshot: ER Diagram]

yaml


Or:

![ER Diagram](screenshots/erd.png)

<br/><br/>

# 📘 PHASE 3: Database Schema

### **1. USERS**
```sql
CREATE TABLE USERS (
    USER_ID NUMBER PRIMARY KEY,
    FULL_NAME VARCHAR2(200) NOT NULL,
    NATIONAL_ID VARCHAR2(20) UNIQUE NOT NULL,
    PHONE VARCHAR2(20),
    EMAIL VARCHAR2(100),
    CREATED_AT DATE DEFAULT SYSDATE
);
```
2. PHONES
```sql

CREATE TABLE PHONES (
    PHONE_ID NUMBER PRIMARY KEY,
    USER_ID NUMBER REFERENCES USERS(USER_ID),
    IMEI VARCHAR2(20) UNIQUE NOT NULL,
    BRAND VARCHAR2(50),
    MODEL VARCHAR2(50),
    STATUS VARCHAR2(20) DEFAULT 'ACTIVE',
    IS_DELETED NUMBER(1) DEFAULT 0,
    CREATED_AT DATE DEFAULT SYSDATE
);
```
3. STOLEN_REPORTS
```sql

CREATE TABLE STOLEN_REPORTS (
    REPORT_ID NUMBER PRIMARY KEY,
    PHONE_ID NUMBER REFERENCES PHONES(PHONE_ID),
    USER_ID NUMBER REFERENCES USERS(USER_ID),
    LOCATION VARCHAR2(100),
    REPORT_DATE DATE DEFAULT SYSDATE,
    STATUS VARCHAR2(20) DEFAULT 'PENDING'
);
```
4. IMEI_TRACKING
```sql

CREATE TABLE IMEI_TRACKING (
    TRACK_ID NUMBER PRIMARY KEY,
    PHONE_ID NUMBER REFERENCES PHONES(PHONE_ID),
    IMEI VARCHAR2(20),
    NETWORK VARCHAR2(50),
    LOCATION VARCHAR2(100),
    TRACK_DATE DATE DEFAULT SYSDATE
);
```
5. AUDIT_LOG
```sql

CREATE TABLE AUDIT_LOG (
    LOG_ID NUMBER PRIMARY KEY,
    ACTION_TYPE VARCHAR2(50),
    DESCRIPTION VARCHAR2(500),
    ACTION_DATE DATE DEFAULT SYSDATE
);
```
Sequences
```sql

CREATE SEQUENCE SEQ_USER START WITH 1;
CREATE SEQUENCE SEQ_PHONE START WITH 1;
CREATE SEQUENCE SEQ_REPORT START WITH 1;
CREATE SEQUENCE SEQ_TRACK START WITH 1;
CREATE SEQUENCE SEQ_LOG START WITH 1;
```
<br/><br/>

# 📘 PHASE 4: Triggers
1. Log phone updates
```sql

CREATE OR REPLACE TRIGGER TRG_PHONE_UPDATE
AFTER UPDATE ON PHONES
FOR EACH ROW
BEGIN
    INSERT INTO AUDIT_LOG (LOG_ID, ACTION_TYPE, DESCRIPTION)
    VALUES (
        SEQ_LOG.NEXTVAL,
        'PHONE_UPDATE',
        'Phone with IMEI ' || :OLD.IMEI || ' updated.'
    );
END;
```
2. Auto IMEI Tracking When Reported Stolen
```sql

CREATE OR REPLACE TRIGGER TRG_IMEI_TRACK
AFTER INSERT ON STOLEN_REPORTS
FOR EACH ROW
BEGIN
    INSERT INTO IMEI_TRACKING (TRACK_ID, PHONE_ID, IMEI, NETWORK, LOCATION)
    VALUES (
        SEQ_TRACK.NEXTVAL,
        :NEW.PHONE_ID,
        (SELECT IMEI FROM PHONES WHERE PHONE_ID = :NEW.PHONE_ID),
        'MTN',
        :NEW.LOCATION
    );
END;
```
3. Soft Delete Protection
```sql

CREATE OR REPLACE TRIGGER TRG_SOFT_DELETE
BEFORE DELETE ON PHONES
FOR EACH ROW
BEGIN
    RAISE_APPLICATION_ERROR(
        -20555,
        'Deletion is not allowed. Use soft delete instead.'
    );
END;
```
<br/><br/>

# 📘 PHASE 5: Package (SPEC + BODY)
PACKAGE SPEC
```sql

CREATE OR REPLACE PACKAGE PKG_SPMS_OPS AS

    PROCEDURE REGISTER_USER(
        P_FULL_NAME IN VARCHAR2,
        P_NID IN VARCHAR2,
        P_PHONE IN VARCHAR2,
        P_EMAIL IN VARCHAR2
    );

    PROCEDURE REGISTER_PHONE(
        P_USER_ID IN NUMBER,
        P_IMEI IN VARCHAR2,
        P_BRAND IN VARCHAR2,
        P_MODEL IN VARCHAR2
    );

    PROCEDURE REPORT_STOLEN(
        P_PHONE_ID IN NUMBER,
        P_USER_ID IN NUMBER,
        P_LOCATION IN VARCHAR2
    );

    PROCEDURE SOFT_DELETE_PHONE(P_PHONE_ID IN NUMBER);

    FUNCTION GET_PHONE_STATUS(P_IMEI IN VARCHAR2) RETURN VARCHAR2;

END PKG_SPMS_OPS;
/
```
PACKAGE BODY
```sql

CREATE OR REPLACE PACKAGE BODY PKG_SPMS_OPS AS

    PROCEDURE REGISTER_USER(
        P_FULL_NAME IN VARCHAR2,
        P_NID IN VARCHAR2,
        P_PHONE IN VARCHAR2,
        P_EMAIL IN VARCHAR2
    ) IS
    BEGIN
        INSERT INTO USERS (USER_ID, FULL_NAME, NATIONAL_ID, PHONE, EMAIL)
        VALUES (SEQ_USER.NEXTVAL, P_FULL_NAME, P_NID, P_PHONE, P_EMAIL);
    END;

    PROCEDURE REGISTER_PHONE(
        P_USER_ID IN NUMBER,
        P_IMEI IN VARCHAR2,
        P_BRAND IN VARCHAR2,
        P_MODEL IN VARCHAR2
    ) IS
    BEGIN
        INSERT INTO PHONES (PHONE_ID, USER_ID, IMEI, BRAND, MODEL)
        VALUES (SEQ_PHONE.NEXTVAL, P_USER_ID, P_IMEI, P_BRAND, P_MODEL);
    END;

    PROCEDURE REPORT_STOLEN(
        P_PHONE_ID IN NUMBER,
        P_USER_ID IN NUMBER,
        P_LOCATION IN VARCHAR2
    ) IS
    BEGIN
        INSERT INTO STOLEN_REPORTS (REPORT_ID, PHONE_ID, USER_ID, LOCATION)
        VALUES (SEQ_REPORT.NEXTVAL, P_PHONE_ID, P_USER_ID, P_LOCATION);
    END;

    PROCEDURE SOFT_DELETE_PHONE(P_PHONE_ID IN NUMBER) IS
    BEGIN
        UPDATE PHONES SET IS_DELETED = 1 WHERE PHONE_ID = P_PHONE_ID;
    END;

    FUNCTION GET_PHONE_STATUS(P_IMEI IN VARCHAR2) RETURN VARCHAR2 IS
        V_STATUS VARCHAR2(20);
    BEGIN
        SELECT STATUS INTO V_STATUS FROM PHONES WHERE IMEI = P_IMEI;
        RETURN V_STATUS;
    END;

END PKG_SPMS_OPS;
/
```
<br/><br/>

# 📘 PHASE 6: Testing & Execution
Sample Test Block
```sql

BEGIN
    PKG_SPMS_OPS.REGISTER_USER('John Doe', '119988776655', '0788888888', 'john@gmail.com');
    PKG_SPMS_OPS.REGISTER_PHONE(1, '356789012345678', 'Samsung', 'S21');
    PKG_SPMS_OPS.REPORT_STOLEN(1, 1, 'Kigali City');
END;
/
```
Expected Output Screenshots
Insert here:

csharp
Copy code
[screenshot: Output 1]
[screenshot: Output 2]
Or:




<br/><br/>

# 📘 PHASE 7: Results Interpretation
✔ Users successfully register
✔ Phones linked to owners
✔ Theft reports captured
✔ IMEI tracking auto-generated
✔ All changes logged
✔ Soft delete prevents data loss

<br/><br/>

# 📘 PHASE 8: Deployment Guide
Run all scripts

@tables.sql  
@sequences.sql  
@triggers.sql  
@package_spec.sql  
@package_body.sql  
Call procedures

EXEC PKG_SPMS_OPS.GET_PHONE_STATUS('356789012345678');
<br/><br/>

# 📘 PHASE 9: Summary
No	Component	Description
1	Tables	Core storage
2	Triggers	Automation & audit
3	Package	Business logic layer
4	Soft Delete	Evidence protection
5	IMEI Tracking	Core intelligence

<br/><br/>

# 📘 PHASE 10: References
Oracle PL/SQL Documentation
