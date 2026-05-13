# Backend Logging Rule

All backend logs must be written to text files. When implementing backend logging:

1. Use file-based logging for all application logs
2. Include timestamps in log entries (ISO format preferred)
3. Create a dedicated logs directory in the backend
4. Use descriptive log file names (e.g., `app.log`, `error.log`, `echo.log`)
5. Ensure the logs directory is created if it doesn't exist
6. Append to log files rather than overwriting
7. Include relevant context in log messages (request ID, user ID, etc. if applicable)

## Example Implementation

```python
import os
from datetime import datetime

log_dir = "logs"
os.makedirs(log_dir, exist_ok=True)
log_file = os.path.join(log_dir, "app.log")

with open(log_file, "a") as f:
    f.write(f"[{datetime.now().isoformat()}] Log message here\n")
```
