```bash
$ vim replace.txt
```

```bash
$ cat replace.txt
literal:sensitive_data==>REDACTED
```

```bash
$ git filter-repo --replace-text replace.txt
```

```bash
$ rm -f replace.txt
```
