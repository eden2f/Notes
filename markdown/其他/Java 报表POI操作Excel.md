# Java 报表POI操作Excel

## Excel简介

一个excel文件就是一个工作簿workbook，一个工作簿中可以创建多张工作表sheet，而一个工作表中包含多个单元格Cell，这些单元格都是由列（Column）行(Row)组成，列用大写英文字母表示，从A开始到Z共26列，然后再从AA到AZ又26列，再从BA到BZ再26列以此类推。行则使用数字表示，例如；A3 表示第三行第一列，E5表示第五行第五列。

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240126231535.png)
	
## POI工具包

JAVA中操作Excel的有两种比较主流的工具包： JXL 和 POI 。jxl 只能操作Excel 95, 97, 2000也即以.xls为后缀的excel。而poi可以操作Excel 95及以后的版本，即可操作后缀为 .xls 和 .xlsx两种格式的excel。

POI全称 Poor Obfuscation Implementation,直译为“可怜的模糊实现”，利用POI接口可以通过JAVA操作Microsoft office 套件工具的读写功能。官网：http://poi.apache.org ，POI支持office的所有版本，并且在接下来的演示中需要从前端页面导入用户上传的版本不确定的excel文件，所以选择POI来讲解。

## 示例环境

* JDK 1.8
* OS: Windows10
* POI 3.16

## Maven依赖
	
``` xml
    <!-- https://mvnrepository.com/artifact/org.apache.poi/poi -->
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi</artifactId>
        <version>3.16</version>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.apache.poi/poi-ooxml -->
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi-ooxml</artifactId>
        <version>3.16</version>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.apache.poi/poi-ooxml-schemas -->
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi-ooxml-schemas</artifactId>
        <version>3.16</version>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.apache.xmlbeans/xmlbeans -->
    <dependency>
        <groupId>org.apache.xmlbeans</groupId>
        <artifactId>xmlbeans</artifactId>
        <version>2.6.0</version>
    </dependency>

    <!-- https://mvnrepository.com/artifact/dom4j/dom4j -->
    <dependency>
        <groupId>dom4j</groupId>
        <artifactId>dom4j</artifactId>
        <version>1.6.1</version>
    </dependency>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
    </dependency>
```
	
## HelloWorld

* 07版本前
  * Excel 的工作簿对应POI的HSSFWorkbook对象；
  * Excel 的工作表对应POI的HSSFSheet对象；
  * Excel 的行对应POI的HSSFRow对象；
  * Excel 的单元格对应POI的HSSFCell对象。
* 07版本及07以后版本
  * Excel 的工作簿对应POI的XSSFWorkbook对象；
  * Excel 的工作表对应POI的XSSFSheet对象；
  * Excel 的行对应POI的XSSFRow对象；
  * Excel 的单元格对应POI的XSSFCell对象。

## Hello POI

``` java
import org.apache.poi.hssf.usermodel.HSSFCell;
import org.apache.poi.hssf.usermodel.HSSFRow;
import org.apache.poi.hssf.usermodel.HSSFSheet;
import org.apache.poi.hssf.usermodel.HSSFWorkbook;

import java.io.FileOutputStream;
import java.io.IOException;

/**
    * Created by huangMP on 2017/8/20.
    * decription :
    */
public class OutputExcelDemo {

    /**
        *  从 工作簿中写入 数据 OutputExcelDemo
        * 07 版本及之前的版本写法
        * @throws IOException
        */
    public void outputExcel() throws IOException {
        // 1. 创建工作簿
        HSSFWorkbook workbook = new HSSFWorkbook();

        // 2. 创建工作类
        HSSFSheet sheet = workbook.createSheet("hello world");

        // 3. 创建行 , 第三行 注意:从0开始
        HSSFRow row = sheet.createRow(2);

        // 4. 创建单元格, 第三行第三列 注意:从0开始
        HSSFCell cell = row.createCell(2);
        cell.setCellValue("Hello World");

        String fileName = "D:\\huangMP\\Desktop\\OutputExcelDemo.xls";
        FileOutputStream fileOutputSteam = new FileOutputStream(fileName);

        workbook.write(fileOutputSteam);
        workbook.close();

        fileOutputSteam.close();
    }
}
```
	
### 测试方法
``` java
@Test
public void outputExcel() throws Exception {
    OutputExcelDemo outputExcelDemo = new OutputExcelDemo();
    outputExcelDemo.outputExcel();
}
```

## 应用

### 从工作簿(Excel)文件读取信息

``` java
    /**
        * 从 工作簿中读取 数据 ReadExcelDemo
        * @throws IOException
        */
    public void readExel() throws IOException {

        String fileName = "D:\\huangMP\\Desktop\\OutputExcelDemo.xls";
        FileInputStream fileInputStream = new FileInputStream(fileName);

        // 1. 创建工作簿
        HSSFWorkbook workbook = new HSSFWorkbook(fileInputStream);

        // 2. 创建工作类
        HSSFSheet sheet = workbook.getSheetAt(0);

        // 3. 创建行 , 第三行 注意:从0开始
        HSSFRow row = sheet.getRow(2);

        // 4. 创建单元格, 第三行第三列 注意:从0开始
        HSSFCell cell = row.getCell(2);
        String cellString = cell.getStringCellValue();

        System.out.println("第三行第三列的值为 : " + cellString );

        workbook.close();
        fileInputStream.close();
    }
```

### 格式化Excel

* 在POI中可以利用格式化对象来格式化excel文档；也即设置excel内容的样式。POI中主要的格式化对象常用的有合并单元格、设置单元格字体、边框，背景颜色等。
* 常用设置
* 合并单元格
	* 在POI中有一个CellRangeAddress对象,中文直译是 单元格范围地址，主要用于在单元格的合并上，这个对象的构造方法CellRangeAddress(int firstRow, int lastRow, int firstCol, int lastCol) 有4个参数，分别表示（起始行号，终止行号， 起始列号，终止列号）, 设置这个对象中要合并的单元格范围后，工作表对象sheet调用方法addMergedRegion(CellRangeAddress region) ，将上述设置的CellRangeAddress对象作为参数传入即可合并单元格。
	
	* 设置单元格样式
		* 首先要设置单元格样式则要先初始化POI中的单元格样式对象HSSFCellStyle，然后在样式对象中设置不同的样式（内容位置、字体、背景、颜色、边框等）。单元格样式是由工作簿workbook创建的，一个工作簿可以创建多个样式。
			* 设置单元格内容位置；设置水平位置 setAlignment(short align) ，设置垂直位置setVerticalAlignment(short align)
			* 设置单元格字体；POI中的字体对象为HSSFFont，字体是由工作簿创建，可以用于多个单元格上。
			* 设置单元格背景色。

#### 示例

``` java
public void testExcelStyle() throws IOException {
	// 1. 创建工作簿
	HSSFWorkbook workbook = new HSSFWorkbook();
	// 1.1 创建单元格对象 合并第三行第三列到5列
	// 构造参数 起始行号 结束行号 起始列号 结束列号
	CellRangeAddress cellRangeAddress = new CellRangeAddress(
			2, 2, 2, 4 );
	// 1.2 创建单元格样式
	HSSFCellStyle style = workbook.createCellStyle();
	style.setAlignment(HSSFCellStyle.ALIGN_CENTER);
	style.setAlignment(HSSFCellStyle.VERTICAL_CENTER);

	// 1.3 创建字体
	HSSFFont font = workbook.createFont();
	font.setBoldweight(HSSFFont.BOLDWEIGHT_BOLD);
	font.setFontHeightInPoints((short)16);
	// 将字体加载到样式中
	style.setFont(font);

	// 1.4 设置背景色为黄色
	// 1.4.1 设置填充模式
	style.setFillPattern(HSSFCellStyle.SOLID_FOREGROUND);
	style.setFillBackgroundColor(HSSFColor.YELLOW.index);
	style.setFillForegroundColor(HSSFColor.GREEN.index);

	// 2. 创建工作类
	HSSFSheet sheet = workbook.createSheet("hello world");
	// 2.1 加入合并单元格对象
	sheet.addMergedRegion(cellRangeAddress);

	// 3. 创建行 , 第三行 注意:从0开始
	HSSFRow row = sheet.createRow(2);

	// 4. 创建单元格, 第三行第三列 注意:从0开始
	HSSFCell cell = row.createCell(2);
	cell.setCellValue("Hello World");
	// 4.1 单元格添加样式
	cell.setCellStyle(style);

	String fileName = "D:\\huangMP\\Desktop\\HelloExcelStyle.xls";
	FileOutputStream fileOutputSteam = new FileOutputStream(fileName);

	workbook.write(fileOutputSteam);
	workbook.close();

	fileOutputSteam.close();

}
```