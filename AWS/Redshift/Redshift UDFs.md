redshift 支持用户自定义函数(UDFs)，其不仅支持sql语法，也支持python脚本(PLPYTHONU)。首先需要授予用户创建函数的权限

```
grant usage on language plpythonu to u
```

