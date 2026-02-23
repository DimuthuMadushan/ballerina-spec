# HTTP Access Log Rotation Support

- Authors - @DimuthuMadushan  
- Reviewed by - @daneshk @TharmiganK 
- Created date - 2025-02-22  
- Updated date - 2025-02-22  
- Issue - https://github.com/ballerina-platform/ballerina-spec/issues/1438
- State - Submitted  

## Summary

This proposal introduces log rotation capability for Ballerina HTTP access logs. It enables automatic rotation of log files based on different policies. Log rotation prevents unbounded growth of log files in long-running HTTP services, reducing disk space consumption and improving operational manageability in production environments.

## Motivation

Currently, the Ballerina http module writes access logs continuously to a single file without any built-in rotation. In long-running deployments this causes log files to grow indefinitely, leading to:

- **Disk space exhaustion** — Access log files for high-throughput HTTP services can grow rapidly and eventually consume all available disk space.
- **Performance degradation** — Large log files slow down log analysis, audit reviews, and incident troubleshooting in production.
- **Operational overhead** — Users must implement custom scripts or external tools (logrotate, cron jobs) to manage log file growth, adding platform-specific operational complexity.

The ballerina log module already provides log rotation. HTTP access logs are distinct log streams with their own file paths and operational requirements, and they should have the same first-class rotation support.

## Goals

- Provide built-in log rotation for HTTP access logs without requiring external tools.
- Enable automatic cleanup of old backup files with a configurable retention limit.
- Maintain full backward compatibility with the existing accessLog configuration API.
- Ensure thread-safe rotation operations under concurrent HTTP request processing.

## Non-Goals

- Log compression (e.g., gzip) of rotated files
- Remote log shipping or integration with external log management systems.
- Asynchronous/background rotation (rotation happens synchronously before writes)

## Design

The current access log implementation is based on the Java Util Logging (JUL) framework. HttpLogManager, which extends JUL LogManager, is responsible for initializing the logging infrastructure. It creates the required loggers, attaches the configured handlers (console, file, or socket), assigns the appropriate formatters, and sets the log levels. The default JUL FileHandler attached to the loggers handles writing log records to files. 

However, the rotation capabilities of the default FileHandler are limited. It supports only size-based rotation and automatic cleanup of old backup files based on the configured file count limit. (Note: These configurations are not exposed to users in the existing implementation.) It does not support time-based rotation or renaming old log files using a timestamp-based format.

To address these limitations, a custom file handler is proposed. The `HttpRollingFileHandler` extends the JUL Handler class directly and implements three rotation policies: `size-based`, `time-based`, and a combination of `both`. It is designed to be thread-safe for concurrent writes. HttpLogManager is updated to register HttpRollingFileHandler instead of the default FileHandler when rotation configuration is specified.

### 1. Configuration

The rotation configuration shape is intentionally identical to Ballerina log module `RotationConfig` to provide a consistent experience across the Ballerina standard library.

#### Rotation Policy

```ballerina
# Log rotation policies for HTTP log files.
public enum RotationPolicy {
    # Rotate log file when it exceeds the configured size limit.
    SIZE_BASED,
    # Rotate log file when the configured time interval elapses.
    TIME_BASED,
    # Rotate log file when either size or time threshold is reached (whichever comes first).
    BOTH
}
```

#### Rotation Configuration Record

```ballerina
# Log rotation configuration for HTTP access log file destinations.
public type RotationConfig record {|
    # Rotation policy to apply.
    # Default: BOTH
    RotationPolicy policy = BOTH;

    # Maximum file size in bytes before triggering rotation.
    # Applicable for SIZE_BASED and BOTH policies.
    # Default: 10MB (10 * 1024 * 1024 bytes)
    int maxFileSize = 10485760;

    # Maximum age in seconds before triggering rotation.
    # Applicable for TIME_BASED and BOTH policies.
    # Default: 24 hours (86400 seconds)
    int maxAge = 86400;

    # Maximum number of rotated backup files to retain.
    # Oldest backup files are deleted when this limit is exceeded.
    # Default: 10
    int maxBackupFiles = 10;
|};
```

#### Access Log Configuration

The `rotation` field is added as an optional field to the existing `AccessLogConfiguration` records, making rotation entirely opt-in and backward compatible.

```ballerina
# Access log configuration with optional rotation support.
public type AccessLogConfiguration record {|
    # Log file path. Only files with .log extension are supported.
    string path?;
    # Log rotation configuration.
    RotationConfig rotation?;
|};
```

### 2. Usage Examples

#### Example 1: Size-Based Rotation for Access Logs

```toml
[ballerina.http.accessLogConfig]
console = true
path = "./logs/http-access.log"

[ballerina.http.accessLogConfig.rotation]
policy = "SIZE_BASED"
maxFileSize = 52428800    // 50MB
maxBackupFiles = 30
```

#### Example 2: Time-Based Daily Rotation for Access Logs

```toml
[ballerina.http.accessLogConfig]
console = true
path = "./logs/http-access.log"

[ballerina.http.accessLogConfig.rotation]
policy = "TIME_BASED"
maxAge = 86400        // Rotate daily
maxBackupFiles = 7    // Keep one week of access 
```

#### Example 3: Combined Policy

```toml
[ballerina.http.accessLogConfig]
console = true
path = "./logs/http-access.log"

[ballerina.http.accessLogConfig.rotation]
policy: "BOTH",
maxFileSize: 52428800,   // 50MB
maxAge: 86400,           // 24 hours
maxBackupFiles: 30       // One month of backups
```

### 3. Rotated File Naming

Rotated files are renamed with a timestamp suffix in the format `yyyy-MM-dd_HH-mm-ss` since the timestamp-based naming carries operational context over numbered suffixes (Ex: app.log.1, app.log.2).

```
http-access.log          ← current active file
http-access-2025-02-19_10-30-00.log   ← rotated backup
http-access-2025-02-18_10-30-00.log   ← older backup
```

### 4. Implementation Details

The rotation logic is implemented in a new native Java class HttpRollingFileHandler which extends java.util.logging.Handler directly. This avoids all dependency on any third-party logging framework, using only Java standard library APIs. The implementation is similar to default JUL File Handler with additional rotation policy support.

```
io.ballerina.stdlib.http.api.logging/
├── HttpRollingFileHandler.java     ← core rolling handler (extends java.util.logging.Handler)
```

#### Backward Compatibility

The rotation field is added as an optional field to the existing AccessLogConfiguration record. All existing applications that configure accessLog with a file path but no rotation field will continue to work without any change. Log files will be written without rotation, exactly as before. This implementation does not enforce a specific file extension in order to maintain backward compatibility. The file extension can be .log, .txt, etc.


## Alternatives

### Alternative 1: Expose Existing File Size–Based Rotation Configurations in Java Default File Handler

**Approach**

The current access log implementation uses the default java.util file handler. The default file handler supports size-based log rotation; however, these configurations are not exposed to users. These configurations can be exposed through AccessLogConfiguration, similar to the approach described above.

```ballerina
# Log rotation configuration for HTTP access log file destinations.
public type RotationConfig record {|
    # Maximum file size in bytes before triggering rotation.
    # Default: 10MB (10 * 1024 * 1024 bytes)
    int maxFileSize = 10485760;

    # Maximum number of rotated backup files to retain.
    # Oldest backup files are deleted when this limit is exceeded.
    # Default: 10
    int maxBackupFiles = 10;
|};
```

**Pros**
- No additional implementation effort (only requires exposing the configurations).
- Built-in file locking management for multi-process safety.
- Built-in byte counting tracks bytes written accurately.
- Can be extended if time-based rotation is introduced in the future.

**Cons**
- No time-based rotation.
- No timestamp on rotated files (added numbered suffix).

## Testing

### Unit Tests

- **Size-based rotation** — Write logs exceeding `maxFileSize`, verify rotation occurs and backup file is created with correct timestamp suffix.
- **Time-based rotation** — CConfigure a short `maxAge` (5 seconds), verify rotation after the interval elapses.
- **Combined policy** — Verify rotation fires when either condition (size or time) is met first.
- **Backup cleanup** — Verify oldest backup files are deleted when count exceeds `maxBackupFiles`.
- **Concurrent writes** — Multiple writing simultaneously to verify no log records are lost and rotation occurs exactly once per threshold crossing.
- **Error handling** — Graceful handling of rotation failures (insufficient permissions, disk full).

### Integration Tests

- **Access log rotation** — Verify access log entries are written to the rotated file correctly after rotation.
- **Config.toml** — Verify rotation configuration is correctly picked up via `Config.toml`.
- **Service lifecycle** — Verify log files are flushed and closed cleanly on listener graceful shutdown.

### Performance Tests

- **During rotation** — Measure latency of the rotation operation itself.
- **High concurrency** — Throughput under concurrent HTTP load with rotation enabled.

## Assumptions

- The Ballerina service process has write permissions for the configured log directory.
- The system clock is reasonably accurate for time-based rotation.
- maxBackupFiles is set to a value appropriate for available disk space.
- Backup files are not modified by external processes between rotations.

## Dependencies

### External Dependencies

- `java.util.logging` — Handler base class, JUL integration
- `java.nio.file` — File operations, atomic move
- `java.nio.channels` — `FileLock` for multi-process safety
- `java.util.concurrent.atomic` — `AtomicLong` for byte counting

## References

### Related Proposals
- [Log file rotation support for Ballerina log](https://github.com/daneshk/ballerina-spec/blob/master/beps/lib-log/1410_log_file_rotation.md)
- [Apache HTTP Server — rotatelogs](https://httpd.apache.org/docs/current/programs/rotatelogs.html)
- [Log4j RollingFileAppender](https://logging.apache.org/log4j/2.x/manual/appenders/rolling-file.html)
- [Spring Boot logging — file rotation](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.logging.file-rotation)
