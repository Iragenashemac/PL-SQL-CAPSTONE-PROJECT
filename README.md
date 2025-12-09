# PL-SQL-CAPSTONE-PROJECT : STOLEN PHONE AND IMEI TRACKING

NAMES: IRAGENA SHEMA Cedrick — 24766



<br/>
## 📘 PHASE 1  Problem Definition
Institutional / National Context

Mobile phone theft has become a major security and economic challenge. Stolen devices are often resold, re-registered, or used to commit cybercrimes. Law enforcement agencies need a smart system to:

Track IMEI activity

Validate phone ownership

Detect stolen phones

Record phone movement and audits

Help telecoms and police collaborate efficiently

Data Challenge

Although IMEI exists, there is no central integrated system to monitor:

Stolen phone reports

Device movement

IMEI search history

SIM history

Owner verification

Expected Output

A centralized PL/SQL-powered system that:

✔ Registers stolen phones
✔ Tracks IMEI automatically
✔ Prevents deletion through soft-delete
✔ Logs all activities
✔ Provides fast search & retrieval
✔ Produces reliable audit trails

<br/><br/>

## 📘 PHASE 2 — System Architecture & ERD
System Entities
Entity	Description
USERS	Registered users (citizens, police, telecom staff)
PHONES	Mobile phones owned by users
STOLEN_REPORTS	Theft reports submitted
IMEI_TRACKING	IMEI tracking history
AUDIT_LOG	Activity logs for accountability
ER Diagram

👉 Screenshot Placeholder (insert here):

![ER Diagram](screenshots/erd.png)


<br/><br/>

## 📘 PHASE 3 — Database Schema
1. USERS Table
CREATE TABLE USERS (
    USER_ID NUMBER PRIMARY KEY,
    FULL_NAME VARCHAR2(200) NOT NULL,
    NATIONAL_ID VARCHAR2(20) UNIQUE NOT NULL,
    PHONE VARCHAR2(20),
    EMAIL VARCHAR2(100),
    CREATED_AT DATE DEFAULT SYSDATE
);

2. PHONES Table
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

3. STOLEN_REPORTS Table
CREATE TABLE STOLEN_REPORTS (
    REPORT_ID NUMBER PRIMARY KEY,
    PHONE_ID NUMBER REFERENCES PHONES(PHONE_ID),
    USER_ID NUMBER REFERENCES USERS(USER_ID),
    LOCATION VARCHAR2(100),
    REPORT_DATE DATE DEFAULT SYSDATE,
    STATUS VARCHAR2(20) DEFAULT 'PENDING'
);

4. IMEI_TRACKING Table
CREATE TABLE IMEI_TRACKING (
    TRACK_ID NUMBER PRIMARY KEY,
    PHONE_ID NUMBER REFERENCES PHONES(PHONE_ID),
    IMEI VARCHAR2(20),
    NETWORK VARCHAR2(50),
    LOCATION VARCHAR2(100),
    TRACK_DATE DATE DEFAULT SYSDATE
);

5. AUDIT_LOG Table
CREATE TABLE AUDIT_LOG (
    LOG_ID NUMBER PRIMARY KEY,
    ACTION_TYPE VARCHAR2(50),
    DESCRIPTION VARCHAR2(500),
    ACTION_DATE DATE DEFAULT SYSDATE
);

Sequences
CREATE SEQUENCE SEQ_USER START WITH 1;
CREATE SEQUENCE SEQ_PHONE START WITH 1;
CREATE SEQUENCE SEQ_REPORT START WITH 1;
CREATE SEQUENCE SEQ_TRACK START WITH 1;
CREATE SEQUENCE SEQ_LOG START WITH 1;


<br/><br/>

## 📘 PHASE 4 — TRIGGERS
Trigger: Log all phone updates
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

Trigger: Track IMEI automatically
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

Trigger: Soft Delete Protection
CREATE OR REPLACE TRIGGER TRG_SOFT_DELETE
BEFORE DELETE ON PHONES
FOR EACH ROW
BEGIN
    RAISE_APPLICATION_ERROR(-20555, 'Deletion is not allowed. Use soft delete instead.');
END;


<br/><br/>

## 📘 PHASE 5 — PACKAGE (SPEC + BODY)
PACKAGE SPEC: PKG_SPMS_OPS
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

PACKAGE BODY (FINAL FIXED VERSION)
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


<br/><br/>

## 📘 PHASE 6 — Testing & Execution
Test Data Insertion
BEGIN
    PKG_SPMS_OPS.REGISTER_USER('John Doe', '119988776655', '0788888888', 'john@gmail.com');
    PKG_SPMS_OPS.REGISTER_PHONE(1, '356789012345678', 'Samsung', 'S21');
    PKG_SPMS_OPS.REPORT_STOLEN(1, 1, 'Kigali City');
END;
/

Expected Output Screenshots

📌 Insert here:

![Test Output](screenshots/output1.png)
![IMEI Tracking](screenshots/output2.png)


<br/><br/>

## 📘 PHASE 7 — Results Interpretation
✔ The system successfully:

Registers users

Links phones to owners

Logs phone theft reports

Creates automatic IMEI tracking entries

Logs all actions

Prevents data loss through soft delete

## 📘 PHASE 8 — Deployment Guide
1. Running in SQL Developer
@tables.sql
@sequences.sql
@triggers.sql
@package_spec.sql
@package_body.sql

2. Calling procedures
EXEC PKG_SPMS_OPS.GET_PHONE_STATUS('356789012345678');

## 📘 PHASE 9 — Summary
No	Component	Description
1	Tables	Core system data storage
2	Triggers	Automation & audit
3	Package	Central business logic
4	Soft-delete	Prevents loss of evidence
5	IMEI tracking	Core intelligence module
## 📘 PHASE 10 — References

Oracle Official PL/SQL Documentation

GSMA IMEI Structure Documentation

Rwanda National Cyber Security Authority Reports
