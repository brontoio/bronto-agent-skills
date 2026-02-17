---
name: Log Statement IDs
description: a skill to describe log statement IDs, how to generate them, inject them into source code, as well as how to create a statement file that contain the mapping between log statements within the project and their associated statement ID.
---

# Updating Statement IDs

A statement ID is a key-value pair where the key is `stmt_id` and its value is a 16 character string. Updating 
statement IDs in a project consists of adding a statement ID in log statements where they would be missing and 
updating the details in statementIds.json.

A statement ID value can be obtained with 

```shell
python -c 'import uuid; print(uuid.uuid4().hex[:16])'
``` 

Location of log statements in the project can be narrowed down with

```shell
grep -r -i -n -E '.info\(|.debug\(|.warn\(|.warning\(|.error\(|.trace\(|.exception\(|.atInfo\(|.atDebug\(|.atWarn\(|.atWarning\(|.atError\(|.atTrace\(|.exception\(' $ARGUMENTS[0] | grep -v stmt_id
```

This command provides the list of log statement candidates and provides their file and line number. From here the full 
log statement must be checked in order to handle multi-line statements. A statement is missing from a statement ID 
if both the following conditions are true:

- its message does not contain a key-value pair of the form `stmt_id=<VALUE>`
- the log statement does not contain key-value pair attributes with `stmt_id` as key, e.g. 
`extra={{"stmt_id": "1234567890abcdef"}}` in Python or `addKeyValue("stmt_id", "1234567890abcdef")` with the 
Java logging Fluent API.

In order to inject statement IDs, simply update log statements by appending ' stmt_id=<STMT_ID>'. 
The value of STMT_ID should be constant and unique per log statement. Below are examples of log statements, 
based on the Python programming language, before and after injecting statement IDs:

```
logger.info('a simple log statement') --> logger.info('a simple log statement stmt_id=8320e3c149f34c28')

logger.info('a log statement with a placeholder %s', value) --> logger.info('a log statement with a placeholder %s stmt_id=be7e775bbaf949a3', value)

logger.info(expr_representing_a_string) --> logger.info(expr_representing_a_string + ' stmt_id=e0c64be98903425a')

logger.info('a multiline ' + 
'log statement') --> logger.info('a multiline ' + 
'log statement stmt_id=12fd106cdffc4a09')
```

If structured logging is used in the codebase, such as using the `extra` parameter in Python or the Fluent 
logging API in Java, then the same pattern should be used to inject statement IDs, e.g. in Python         

```
logger.info('a simple log statement') --> logger.info('a simple log statement', extra={{'stmt_id': '8320e3c149f34c28'}})
```

and in Java with the Fluent API:

```shell
logger.atInfo().setMessage("a simple log statement").log() --> logger.atInfo().setMessage("a simple log statement").addKeyValue("stmt_id, "8320e3c149f34c28").log()
```

The following applies when adding statement IDs:
- Statement IDs should only be injected into source code located under $ARGUMENTS[0] 
- Only one statement ID should be defined per log statement.
- Statement IDs should be injected in log statement messages, regardless of the severity of the log statement
- Event though the samples above focus on the Python programming language, statement IDs should be injected 
regardless of the programming language used.

Also, for each log statement, there should be an entry in the statementIds.json file. The format of this 
file is

```json
[
  {
    "statement_id": "1234567890abcdef",
    "log_statement": "some log statement expression"
  },
  ...
]
```

where 
- statement_id is the stmt_id value of the log statement
- log_statement is the log statement message itself

Examples
- if the message of the log statement is 

```this is a %s statement, stmt_id=1234567890```

then extracting the statement ID and log statement should lead to 

```json
{
  "statement_id": "1234567890",
  "log_statement": "this is a %s statement, stmt_id=1234567890"
}
```
- if the log message is a concatenation of strings, then please use the evaluated string as log statement, 
i.e. extracting 

```'this is a %s' + ' statement, stmt_id=1234567890'```

should lead to 

```json
{
  "statement_id": "1234567890",
  "log_statement": "this is a %s statement, stmt_id=1234567890"
}
```

- finally, if the log statement contains non evaluated string, please replace these parts with %s, for instance 
extracting 

```'this is a %s ' + str(my_object) + ' statement, stmt_id=1234567890'```

should lead to 

```json
{
  "statement_id": "1234567890",
  "log_statement": "this is a %s %s statement stmt_id=1234567890"
}
```

Write all the extracted log statements and statement IDs to a CSV file named statementIds.csv at 
the root of this project.
