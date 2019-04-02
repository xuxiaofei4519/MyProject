---
title: Hbase通用查询-java实现
date: 2019-04-01 11:04:00
copyright: true
tags:
 - Hbase
categories:
 - Hbase
---

{% cq %} 
数据组同事休产假，因为之前学过一段时间大数据组件，突然临时接手，做了相关Hbase查询的需求。之前的查询接口不能满足日常的查询需求，因此对接口进行升级改造，使其更通用。
{% endcq %}
<!-- more -->

#### 基础类
```java
/**
 * hbase条件类
 * @author xuxiaofei
 */
@EqualsAndHashCode(callSuper = true)
@Data
public class HbaseCodition extends BasicHbaseBo {
    /**
     * 运算符
     */
    private HbaseOpEnum operator;

    /**
     * 过滤类型
     */
    private HbaseFilterTypeEnum filterType;

    /**
     * 比较器类型
     */
    private HbaseComparatorEnum comparator;

    /**
     * 条件value，数组类型是为了满足一些Filter支持多个条件值的情况，
     * 一般情况为一个条件value
     */
    private String[] conditionValue;

}

/**
 * @author xuxiaofei
 */
@Data
public class BasicHbaseBo implements Serializable {
    /**
     * 列族
     */
    private String family;

    /**
     * 列名
     */
    private String qualifier;

    /**
     * 版本号
     */
    private String version;
}


public class PageData<E> implements Serializable {
	private final int DEFAULT_PAGESIZE = 10;
	private final int DEFAULT_CURRENTPAGE = 0;
	private final int DEFAULT_COUNT = 0;

	//总记录数
	private int total = DEFAULT_COUNT;
	//当前页
	private int currentPage;
	//每页数量
	private int pageSize;
	//开始索引
	private int startIndex;
	//结束索引
	private int endIndex;
	// 当前页的记录列表
	private List<E> rows;

	public PageData(int currentPage, int pageSize) {
		this.currentPage = currentPage;
		this.pageSize = pageSize;
        
		if (currentPage < 1) {
			currentPage = DEFAULT_CURRENTPAGE;
		}
		if (pageSize < 0) {
			pageSize = DEFAULT_PAGESIZE;
		}
		this.startIndex = currentPage * pageSize;
		this.endIndex = startIndex + pageSize;
	}
}

```
#### 根据rowkey列表查询
```java
    /**
     * 根据rowkey批量查询对应行数据.
     */
    private BaseResp<HBaseQueryResp> queryByRowKeyList(List<byte[]> rowkeyList, String tableName, BaseResp<HBaseQueryResp> queryResp){
        if (rowkeyList == null || rowkeyList.size() <= 0){
            queryResp.setCode(ErrorCodeEnum.PDPT10002.getCode());
            queryResp.setMessage("查询rowkey列表为空，不能继续查询数据");
            return queryResp;
        }
        try (Table table = getConnection().getTable(TableName.valueOf(tableName))){
            List<Get> getList = new ArrayList<>();
            for (byte[] rowkey : rowkeyList){
                Get get = new Get(rowkey);
                getList.add(get);
            }

            Result[] result = table.get(getList);
            List<HBaseRow> rows = new ArrayList<>();
            for (Result rs: result){
                HBaseRow hbaseRow = new HBaseRow();
                String rowKey = Bytes.toString(rs.getRow());
                Map<String, Object> columMap = new HashMap<>();
                for (Cell cell : rs.listCells()) {
                    //获取列信息
                    String column = Bytes.toString(cell.getQualifierArray(), cell.getQualifierOffset(), cell.getQualifierLength());
                    Object value;
                    //根据列名判断是否是数字类型，用来做金额判断等
                    if (column.endsWith("_number")){
                        value = Bytes.toLong(cell.getValueArray(), cell.getValueOffset(), cell.getValueLength());
                    } else {
                        value = Bytes.toString(cell.getValueArray(), cell.getValueOffset(), cell.getValueLength());
                    }
                    columMap.put(column, value);
                }
                hbaseRow.setRowKey(rowKey);
                hbaseRow.setColums(columMap);
                log.info("HBaseQueryService.queryByRowKeyList hbaseRow=" + JSON.toJSONString(hbaseRow), SerializerFeature.WriteMapNullValue);
                rows.add(hbaseRow);
            }
            HBaseQueryResp hbaseQueryResp = queryResp.getData();
            hbaseQueryResp.setRows(rows);
            queryResp.setCode(ErrorCodeEnum.PDPT00000.getCode());
            queryResp.setMessage("查询成功");
            queryResp.setData(hbaseQueryResp);
            return queryResp;
        }catch (Exception e){
            log.error("HBaseQueryService.queryByRowKeyList exception:", e);
            queryResp.setCode(ErrorCodeEnum.PDPT10000.getCode());
            queryResp.setMessage("HBaseQueryService.queryByRowKeyList 查询异常");
            return queryResp;
        }
    }
```

#### RowKeys查询方法
```java
    /**
     * 查询满足条件的rowkey列表
     */
    private Map<String, Object> queryRowKeyByCondition(String tableName, String rowKeyPrefix,  String startRow, String endRow, List<HbaseCodition> conditionList, PageData pageData, FilterList.Operator operator) throws Exception {
        Map<String, Object> result = new HashMap<>(2);
        try (Table table = getConnection().getTable(TableName.valueOf(tableName))){
            Scan scan = new Scan();
            if (StringUtils.isNotBlank(startRow)) {
                scan.withStartRow(Bytes.toBytes(startRow), true);
            }
            //结果不包括enRow这行
            if (StringUtils.isNotBlank(endRow)) {
                scan.withStopRow(Bytes.toBytes(endRow), true);
            }
            // rowKey前缀过滤
            if (StringUtils.isNotBlank(rowKeyPrefix)){
                scan.setRowPrefixFilter(Bytes.toBytes(rowKeyPrefix));
            }
            //条件列表
            FilterList filterList = new FilterList(operator);
            for (HbaseCodition condition : conditionList){

                String[] conditionValue = condition.getConditionValue();
                String family = condition.getFamily();
                String qualifier = condition.getQualifier();
                String version = condition.getVersion();

                //匹配运算符
                CompareFilter.CompareOp compareOp;
                switch (condition.getOperator()){
                    case LESS:
                        compareOp = CompareFilter.CompareOp.LESS;
                        break;
                    case LESS_OR_EQUAL:
                        compareOp = CompareFilter.CompareOp.LESS_OR_EQUAL;
                        break;
                    case EQUAL:
                        compareOp = CompareFilter.CompareOp.EQUAL;
                        break;
                    case NOT_EQUAL:
                        compareOp = CompareFilter.CompareOp.NOT_EQUAL;
                        break;
                    case GREATER_OR_EQUAL:
                        compareOp = CompareFilter.CompareOp.GREATER_OR_EQUAL;
                        break;
                    case GREATER:
                        compareOp = CompareFilter.CompareOp.GREATER;
                        break;
                    case NO_OP:
                        compareOp = CompareFilter.CompareOp.NO_OP;
                        break;
                    default:
                        compareOp = CompareFilter.CompareOp.NO_OP;
                }
                //匹配比较器
                ByteArrayComparable comparator;
                switch (condition.getComparator()){
                    case BINARY:
                        comparator = new BinaryComparator(Bytes.toBytes(conditionValue[0]));
                        break;
                    case BINARYPREFIX:
                        comparator = new BinaryPrefixComparator(Bytes.toBytes(conditionValue[0]));
                        break;
                    case SUBSTRING:
                        comparator = new SubstringComparator(conditionValue[0]);
                        break;
                    case REGEXSTRING:
                        comparator = new RegexStringComparator(conditionValue[0]);
                        break;
                    case LONG:
                        comparator = new LongComparator(new BigDecimal(conditionValue[0]).toBigInteger().longValue());
                        break;
                    default:
                        comparator = new BinaryComparator(Bytes.toBytes(conditionValue[0]));

                }

                //匹配过滤器类型
                switch (condition.getFilterType()){
                    case SINGLECOLUMNVALUE:
                        SingleColumnValueFilter singleColumnValueFilter = new SingleColumnValueFilter(Bytes.toBytes(family), Bytes.toBytes(qualifier), compareOp, comparator);
                        // 当参考列不存在时返回结果不包含该行
                        singleColumnValueFilter.setFilterIfMissing(true);
                        filterList.addFilter(singleColumnValueFilter);
                        break;
                    case FAMILY:
                        FamilyFilter familyFilter = new FamilyFilter(compareOp, comparator);
                        filterList.addFilter(familyFilter);
                        break;
                    case COLUMNPREFIX:
                        ColumnPrefixFilter columnPrefixFilter = new ColumnPrefixFilter(Bytes.toBytes(qualifier));
                        filterList.addFilter(columnPrefixFilter);
                        break;
                    case MULTIPLECOLUMNPREFIX:
                        String[] qualifierArray = qualifier.split("\\|");
                        List<byte[]> qualifierList = new ArrayList<>();
                        for (String str : qualifierArray) {
                            qualifierList.add(Bytes.toBytes(str));
                        }
                        byte[][] bytes = (byte[][]) qualifierList.toArray();
                        MultipleColumnPrefixFilter multipleColumnPrefixFilter = new MultipleColumnPrefixFilter(bytes);
                        filterList.addFilter(multipleColumnPrefixFilter);
                        break;
                    case ROW:
                        RowFilter rowFilter = new RowFilter(compareOp, new BinaryPrefixComparator(Bytes.toBytes(conditionValue[0])));
                        filterList.addFilter(rowFilter);
                        break;
                    default:
                        throw new Exception("不支持的过滤器类型：" + condition.getFilterType());
                }
            }
            if (filterList.size() > 0){
                scan.setFilter(filterList);
            }
            ResultScanner scanner = table.getScanner(scan);
            List<byte[]> rowKeyList = new ArrayList<>();
            int total = 0;
            for (Result r : scanner) {
                if (total >= pageData.getStartIndex() && total < pageData.getEndIndex() ) {
                    rowKeyList.add(r.getRow());
                }
                total++;
            }
            result.put("total", total);
            result.put("rowKeyList", rowKeyList);
            return result;
        }catch (Exception e){
            log.error("HBaseQueryService.queryByConditionList exception", e);
            throw e;
        }
    }
```


