---
theme: seriph
background: slides-bg.png
class: text-center
highlighter: shiki
lineNumbers: false
info: |
  ## DB2 C Legacy
drawings:
  persist: false
defaults:
  foo: trues
transition: slide-left
title: DB2 C Legacy
mdc: true
---

# DB2 Stored Procedure C Legacy - Application developer's perspective


---
transition: fade-out
---
<Toc />

---
---
# PLSQL in DB2 back in glorious C-app days

- 💎: PLSQL is really cool when you have a C (or CGI remmember?) as backend. 
- Apps can not handle exceptions. SQL errors should be written into outparams
- can not read result sets from cursors
- if sqlcode != 0, else commit
- Exit handler does not mitigate the exception. This will throw in JAVA side. 

---
---

# Anatomy of simple stored procedure

```plsql
CREATE or REPLACE PROCEDURE CM.GET_MY_DATA1(
  IN IUSERID,
  OUT O_DATA,
  OUT OSQL_CODE
)
P1:
BEGING

DECLARE EXIT HANDLER FOR SQLEXCEPTION
SET ORET_CODE = SQLCODE;

SELECT foo FROM CM.DATA INTO O_DATA WHERE ID = IUSERID;

END P1
```

---
---
# Back to the Future - JAVA

- Now we can read read resultsets from cursors! 
- We can also catch exceptions
- It's also easy just to use old stored procedures and read SQL errors from OUT-params
- Jdbc driver does the commit / rollback handling. No exception means commit. 
- Exit handler just sets the osql_code into sql error. (this could be read from exception)
- PLSQL devs are used to using EXIT HANDLERS to hide exceptions..
- This is how we're doing it now...
```
int sqlcode = getInteger(2);
if(sqlcode !=0)
  throw new SQLException("my stored procedure failed with error: " + sqlcode);
```

---
---
# The New way of writing PLSQL

```plsql
CREATE or REPLACE PROCEDURE CM.GET_MY_DATA2(
  IN IUSERID,
  OUT O_DATA,
  OUT OSQL_CODE
) DYNAMIC RESULT SETS 1
P1:
BEGING
DECLARE mycursor CURSOR WITH RETURN FOR
  SELECT foo FROM CM.DATA WHERE ID = IUSERID

DECLARE EXIT HANDLER FOR SQLEXCEPTION
SET ORET_CODE = SQLCODE;

OPEN mycursor;
END P1
```

---
---

# Back to the Future Part II

- Same story goes for other old Databases. DB2, Oracle, Microsoft SQL
- They can return resultsets and use outparams at the same time
- New databases are also developed
- New engines (Postgress etc.) do not support out params and cursors at the same time
 - We still could write excatly the same as in the C-apps era (everything into OUT).
 - can not support the nonsense Hybrid model.
---
---

# C-ERA procedure in plpgsql (Postgress)
```plsql
create or replace function cm.get_my_data1(
  in iuserid integer,
  out o_data varchar,
  out o_sqlcode integer)
language plpgsql
as $$
begin
   select foo from cm.data into o_data where id = iuserid;
exception when others then 
  o_sqlcode = SQLSTATE;
end;$$

```
---
---
# Migrating into Postgress - Cleaning up this mess

- Do not use Exit handlers to pass the error codes into applcation side
- remove exit handlers. Just raise an exception if things are not going in your way.
- Remove old application code that requires an error in out-param. It's really toxic legacy. 
- simple fixes into DB2 PLSQL. Simple to implement in postgress as well. 

---
---
# An example migration
-db2
```plsql
CREATE or REPLACE PROCEDURE CM.GET_MY_DATA2(
  IN IUSERID,
) DYNAMIC RESULT SETS 1
P1:
BEGING
DECLARE mycursor CURSOR WITH RETURN FOR
  SELECT foo FROM CM.DATA WHERE ID = IUSERID

OPEN mycursor;
END P1
```
-postgress
```plsql
create or replace function cm.get_my_data1(
  in iuserid integer) returns refcursor
language plpgsql
as $$
declare
   ref1 refcursor := 'get_my_data'
begin
   open ref1 for select foo from cm.data where id = iuserid;
   return (ref1);
end;$$
```