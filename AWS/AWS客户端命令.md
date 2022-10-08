[TOC]

# AWS客户端命令

## DMS

**创建endpoint**

```shell
aws dms create-endpoint --endpoint-identifier=test-kinesis-posp-data --engine-name kinesis --endpoint-type target --region ap-east-1 --kinesis-settings ServiceAccessRoleArn=arn:aws:iam::502076313352:role/mySpectrumRole,StreamArn=arn:aws:kinesis:ap-east-1:502076313352:stream/posp-data,MessageFormat=json-unformatted
```

