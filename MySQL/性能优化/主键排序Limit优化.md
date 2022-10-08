# limit对order by id的性能影响

```
SELECT
	id AS id,
	order_no AS orderNo,
	payment_order_no AS paymentOrderNo,
	order_status AS orderStatus,
	order_type AS orderType,
	order_amount AS orderAmount,
	sub_total_amount AS subTotalAmount,
	trans_type AS transType,
	app_source AS appSource,
	country_code AS countryCode,
	currency,
	member_id AS memberId,
	delivery_time AS deliveryTime,
	delivery_method AS deliveryMethod,
	cannel_time AS cannelTime,
	activity_id AS activityId,
	activity_type AS activityType,
	activity_status AS activityStatus,
	completed_time AS completedTime,
	shipping_address_id AS shippingAddressId,
	shipping_fee AS shippingFee,
	shipping_no AS shippingNo,
	shipping_company AS shippingCompany,
	shipping_status AS shippingStatus,
	palmbuy_coupon_id AS palmbuyCouponId,
	palmbuy_coupon_amount AS palmbuyCouponAmount,
	discount_amount AS discountAmount,
	ip_address AS ipAddress,
	device_finger AS deviceFinger,
	imei,
	delete_flag AS deleteFlag,
	phone,
	comment_status AS commentStatus,
	create_time AS createTime,
	update_time AS updateTime
FROM
	shopping_order_data.palmbuy_order
WHERE
	(
		activity_id = 'S1110124440'
		AND order_type = 3
		AND activity_status IS NOT NULL
	)
ORDER BY
	id DESC
LIMIT 5;
```

![image-20211224145859258](https://gitee.com/dba_one/wiki_images/raw/master/images/image-20211224145859258.png)

通过检索慢查询日志发现上述SQL执行较慢，统计信息显示SQL平均需要扫描100+行，平均执行时间要8秒，但SQL只需要返回5行数据，因此说明SQL执行计划存在问题，查询执行计划如下：

| id   | select_type | table         | type  | key     | key_len | rows | filtered | Extra       |
| ---- | ----------- | ------------- | ----- | ------- | ------- | ---- | -------- | ----------- |
| 1    | SIMPLE      | palmbuy_order | index | PRIMARY | 4       | 590  | 0.7      | Using where |

执行计划显示SQL通过主键索引完成的查询，可实际上存在activity_id和order_type的组合索引，这里为什么没有走呢？通过FORCE INDEX来强制SQL走二级索引来对比两者的性能

| id   | select_type | table         | type  | key                       | key_len | rows  | filtered | Extra                                 |
| ---- | ----------- | ------------- | ----- | ------------------------- | ------- | ----- | -------- | ------------------------------------- |
| 1    | SIMPLE      | palmbuy_order | range | idx_activity_id_ordertype | 213     | 18950 | 100      | Using index condition; Using filesort |

最后发现，走主键查询需要8秒，走索引查询仅需0.22秒。两者性能差距非常大，那为什么MySQL优化器宁愿选择主键也不选择索引呢？

**优化器的索引选择**

选择索引是MySQL优化器的工作，其目的就是为了寻找代价最低的执行计划方案，其中扫描行数是影响的因素之一，扫描行数越少，资源消耗也就越少。当然优化器也会将是否使用临时表，是否排序等因素结合进行判断。

因此，对比两种情况的执行计划会发现走主键扫描行数只需要590，而走二级索引需要18950，并且走索引还需要额外的排序。也就解释了为什么执行计划最终选择了走主键而不是二级索引。

那是不是走主键真的只需要扫描590行呢？实际上会对整个主键索引进行扫描，远不止590行，这里的590只是MySQL根据查询条件、索引和limit综合考虑出来的预估行数。



**优化思路**

根据上面的描述，可以知道执行代价是优化器选择的标准，那是不是可以人为提升走主键的代价，从而干预影响优化器的选择

1、增加Limit

根据上面的描述，可以知道执行代价是优化器选择的标准，那是不是可以人为提升走主键的代价，从而干预影响优化器的选择。只需要增大limit的值，例如改为limit 500，优化器再次评估时就会发现走主键比走二级索引代价更高。但这种方式并不能一劳永逸，可能随着数据变化，再次选择走主键扫描。

2、子查询

Limit也是影响索引选择的一大因素，因此可以通过子查询利用二级索引先把数据查询出来，再进行Limit，从而避免Limit的影响

```
select 
	id AS id,
	order_no AS orderNo,
	payment_order_no AS paymentOrderNo,
	order_status AS orderStatus,
	order_type AS orderType,
	order_amount AS orderAmount,
	sub_total_amount AS subTotalAmount,
	trans_type AS transType,
	app_source AS appSource,
	country_code AS countryCode,
	currency,
	member_id AS memberId,
	delivery_time AS deliveryTime,
	delivery_method AS deliveryMethod,
	cannel_time AS cannelTime,
	activity_id AS activityId,
	activity_type AS activityType,
	activity_status AS activityStatus,
	completed_time AS completedTime,
	shipping_address_id AS shippingAddressId,
	shipping_fee AS shippingFee,
	shipping_no AS shippingNo,
	shipping_company AS shippingCompany,
	shipping_status AS shippingStatus,
	palmbuy_coupon_id AS palmbuyCouponId,
	palmbuy_coupon_amount AS palmbuyCouponAmount,
	discount_amount AS discountAmount,
	ip_address AS ipAddress,
	device_finger AS deviceFinger,
	imei,
	delete_flag AS deleteFlag,
	phone,
	comment_status AS commentStatus,
	create_time AS createTime,
	update_time AS updateTime
FROM
	shopping_order_data.palmbuy_order
WHERE id in (select id from  shopping_order_data.palmbuy_order where 
	(
		activity_id = 'S1110124440'
		AND order_type = 3
		AND activity_status IS NOT NULL
	)) order by id desc
LIMIT 5;
```







