# Excel 导出行数限制 Invalid row number (65536) outside allowable range (0..65535)

在 Excel 早期版本中，默认的工作薄扩展名为".xls"，：最大256(IV，2的8次方)列，最大65536(2的16次方)行;即横向256个单元格，竖向65536个单元格。
自 Office 2007 版本起，Excel 默认的工作薄扩展名为".xlsx"，最大16384(XFD，2的14次方)列，最大1048576(2的20次方)行;即横向16384个单元格，竖向1048576个单元格。
**使用 EasyExcel 导出".xls"文件超出行数上限报错如下**
```java
java.lang.IllegalArgumentException: Invalid row number (65536) outside allowable range (0..65535)
	at org.apache.poi.hssf.usermodel.HSSFRow.setRowNum(HSSFRow.java:252)
	at org.apache.poi.hssf.usermodel.HSSFRow.<init>(HSSFRow.java:86)
	at org.apache.poi.hssf.usermodel.HSSFRow.<init>(HSSFRow.java:70)
	at org.apache.poi.hssf.usermodel.HSSFSheet.createRow(HSSFSheet.java:259)
	at org.apache.poi.hssf.usermodel.HSSFSheet.createRow(HSSFSheet.java:83)
	at com.alibaba.excel.util.WorkBookUtil.createRow(WorkBookUtil.java:70)
	at com.alibaba.excel.write.executor.ExcelWriteAddExecutor.addOneRowOfDataToExcel(ExcelWriteAddExecutor.java:60)
	at com.alibaba.excel.write.executor.ExcelWriteAddExecutor.add(ExcelWriteAddExecutor.java:51)
	at com.alibaba.excel.write.ExcelBuilderImpl.addContent(ExcelBuilderImpl.java:61)
	at com.alibaba.excel.ExcelWriter.write(ExcelWriter.java:161)
	at com.alibaba.excel.ExcelWriter.write(ExcelWriter.java:146)
```
