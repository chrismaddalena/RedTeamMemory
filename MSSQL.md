# Microsoft SQL Server

## Clients

### Heidi SQL

- OS: Windows
- Requirements: Standalone and portable
- Install: https://www.heidisql.com/

Run it and add a new connection. Very simple.

### mssql-cli

- OS: Any
- Requirements; Python
- Install: `pip install mssql-cli`, https://github.com/dbcli/mssql-cli

A really nice command line implementation with auto-complete, syntax highlighting, and pretty output.

`mssql-cli -S server -U user -P password`

A `-d` option specifies a database and `--auto-vertical-output` can help if running queries that will return results wider than the terminal.

### Impacket's mssqlclient.py

- OS: Any
- Requirements; Python
-Install: https://github.com/CoreSecurity/impacket/blob/master/examples/mssqlclient.py

It works and comes with Impacket. It can be kind of fussy in some cases, but supports useful `xp_cmdshell` tools.

`mssqlclient.py domain/user:password@server`

## Basic Enumeration

### Queries

* Get user information: `select SUSER_NAME()`
* List all databases: `SELECT name FROM master.dbo.sysdatabases` or `EXEC sp_databases`
* List all tables:
    * SQL Server 2005+: `SELECT * FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE='BASE TABLE'`
    * Alternative: `SELECT * FROM <DATABASE_NAME>.INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE='BASE TABLE'`
    * SQL Server 2000: `SELECT * FROM sysobjects WHERE xtype='U'`
* List linked servers:
    * `EXEC sp_linkedservers`
    * `SELECT name, is_linked FROM sys.servers`
    * `SELECT * FROM master..sys.servers`

### Advanced Queries

* Execute query on a remote SQL Server: `SELECT * FROM OPENQUERY("REMOTE_SERVER_NAME", 'QUERY')`
* Similar to above: `EXECUTE('query') AT "REMOTE_SERVER_NAME"`

### Interesting Stored Procedures

* SQL Server 2016's `sp_execute_external_script`: Potentially execute Python scripts.
    *   `EXECUTE sp_execute_external_script @language=N'Python',@script=N'print(1+1)'`

### Using Metasploit

A number of Metasploit modules exist for MSSQL. Here are some highlights worth remembering:

* auxiliary(scanner/mssql/mssql_login) : Check logins against a tagret SQL Server.
* auxiliary(admin/mssql/mssql_exec) : Attempts code execution against target SQL Server (xp_cmdshell).
* exploit(windows/mssql/mssql_linkcrawler) : Automatically explore server links.

## Notes

* Double quotes should only be used around the names of databases and the like.
* Single quotes can be nested by increasing escaped single quotes:
    * Ex: `EXECUTE('First ''Second(''''Third'''')''')`
* PowerUpSQL: https://github.com/NetSPI/PowerUpSQL