## The one-line patch

In `libavformat/dashenc.c`, find:
```c
snprintf(os->temp_path, sizeof(os->temp_path), use_rename ? "%s.tmp" : "%s", os->full_path);
```

Change to:
```c
snprintf(os->temp_path, sizeof(os->temp_path), use_rename ? "%s" : "%s", os->full_path);
```
