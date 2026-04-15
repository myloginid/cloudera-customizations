# CDP 719 Hue/Trino integration 

This documents every code and configuration change we applied to get Hue → Trino working on CDP 719 Python 3 / Java 8 environment. 

## 1. JDBC connector cleanup (code)

1. Edit `desktop/libs/notebook/src/notebook/connectors/jdbc.py` **and** `.../connectors/rdbms.py`. Replace the legacy Python 2 `raise QueryError, e` blocks with the Python 3 safe pattern:
   ```python
   raise QueryError(message).with_traceback(sys.exc_info()[2])
   ```
   This keeps the traceback while avoiding invalid syntax under Python 3.
2. Run syntax checks inside the Hue virtualenv:
   ```bash
   /opt/cloudera/parcels/CDH-7.1.9-1.cdh7.1.9.p1003.76239544/lib/hue/build/env/bin/python3 -m py_compile \
     desktop/libs/notebook/src/notebook/connectors/jdbc.py \
     desktop/libs/notebook/src/notebook/connectors/rdbms.py
   ```
   Fix any errors before restarting Hue.
3. Git-style diff example for the `jdbc.py` change:
   ```diff
   @@
   -    raise QueryError, e
   +    raise QueryError(message).with_traceback(sys.exc_info()[2])
   ```

## 2. Python dependencies

1. Install Py4J (Hue’s JDBC bridge requires it):
   ```bash
   /opt/cloudera/parcels/CDH-7.1.9-1.cdh7.1.9.p1003.76239544/lib/hue/build/env/bin/pip install py4j
   ```
   Internet access is required to do this.

## 3. Trino JDBC driver availability

1. Pick a Trino JDBC jar compiled for Java 8 (≤ version 469). Place it where Hue can read it, e.g. `/opt/cloudera/trino-jdbc-469.jar`. Set chown and chmod so that Hue process can access the driver
2. Ensure Hue’s safety-valve config exposes the jar:
   - Confirm the generated runtime configs (e.g., `/var/run/cloudera-scm-agent/process/.../hue.ini`, `supervisor.conf`, `proc.json`) contain `CLASSPATH=/opt/cloudera/trino-jdbc-469.jar`.
3. Customize hue_service_safety_valve to add JDBC interpreter entry:
   ```ini
   [[interpreters]]
   [[[trino]]]
   name      = Trino JDBC
   interface = jdbc
   options   = '{"url":"jdbc:trino://<host>:<port>","driver":"io.trino.jdbc.TrinoDriver","catalog":"hive","schema":"default","user":"<user>","password":"<strong>"}'
   ```
   Set the correct password. Default catalog should be set to the default catalog name in Trino.

## 4. Hue debuging helpers

1. In the `[desktop]` section of the Hue hue_service_safety_valve config, enable debug output:
   ```ini
   [desktop]
   django_debug_mode=true
   send_debug_messages=true
   http_500_debug_mode=true
   ```
2. Use Cloudera Manager to restart the Hue service once these edits are saved.

## 5. Validation workflow

1. After restarting Hue via CM, re-run the problematic Trino notebook query/logins.
2. Confirm `/var/log/hue/error.log` no longer reports:
   * `py4j` import failures
   * `ClassNotFoundException: io.trino.jdbc.TrinoDriver`
   * `UnsupportedClassVersionError`
   * `Catalog must be specified when session catalog is not set`
3. Tail the logs while reproducing to ensure the debug knobs emit the full stack traces you now need for troubleshooting.

## 6. Summary

| Area | Change |
| --- | --- |
| Code | Replaced Python 2 raise syntax with `.with_traceback()`, compiled both connectors with Python 3. |
| Dependencies | Installed `py4j` in Hue’s virtualenv. |
| Classpath | Added `/opt/cloudera/trino-jdbc-469.jar` via `dbproxy_extra_classpath` + `CLASSPATH`. |
| Interpreter | Documented the Trino JDBC interpreter definition with catalog/schema defaults. |
| Debugging | Enabled Django/Hue debug flags for richer error logs. |
| Validation | Restart Hue via Cloudera Manager and verify `/var/log/hue/error.log`. |

Use this guide as the canonical rebuild plan for future environments.

## 7. Credits
Manish Maheshwari - https://github.com/myloginid
