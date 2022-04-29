## 一丶简介

EasyExcel 是一个基于 Java 的简单、省内存的读写 Excel 的开源项目。在尽可能节约内存的情况下支持读写百 M 的 Excel。



## 二、传统解析的弊端

Java 解析、生成 Excel 比较有名的框架有 Apache poi、jxl。但他们都存在一个严重的问题就是非常的耗内存，poi 有一套 SAX 模式的 API 可以一定程度的解决一些内存溢出的问题，但 POI 还是有一些缺陷，比如 07 版 Excel 解压缩以及解压后存储都是在内存中完成的，内存消耗依然很大。easyexcel 重写了 poi 对 07 版 Excel 的解析，能够原本一个 3M 的 excel 用 POI sax 依然需要 100M 左右内存降低到几 M，并且再大的 excel 不会出现内存溢出，03 版依赖 POI 的 sax 模式。在上层做了模型转换的封装，让使用者更加简单方便。



## 三、快速感受

**excel 内容**

![img](https://static001.geekbang.org/infoq/f1/f1cae2554d486bdabb34c1152bf32421.png)



**读 Excel**

```java
/**	简单的读取excel文件*/
@Test
public void read() {    
	String fileName = "demo.xlsx";    
	// 这里 需要指定读用哪个class去读，然后读取第一个sheet 文件流会自动关闭    
	// 参数一：读取的excel文件路径    
	// 参数二：读取sheet的一行，将参数封装在DemoData实体类中    
	// 参数三：读取每一行的时候会执行DemoDataListener监听器    
	EasyExcel.read(fileName, DemoData.class, new DemoDataListener()).sheet().doRead();
}
```

复制代码



**写 Excel**

```java
/**	简单的写入数据到excel*/
@Test
public void simpleWrite() {    
    String fileName = "demo.xlsx";    
    // 这里 需要指定写用哪个class去读，然后写到第一个sheet，名字为模板 然后文件流会自动关闭    
    // 如果这里想使用03 则 传入excelType参数即可    
    // 参数一：写入excel文件路径    
    // 参数二：写入的数据类型是DemoData    
    // data()方法是写入的数据，结果是List<DemoData>集合    
    EasyExcel.write(fileName, DemoData.class).sheet("模板").doWrite(data());
}
```

复制代码



**Web 上传与下载**

```java
/**	excel文件的下载*/
@GetMapping("download")
public void download(HttpServletResponse response) throws IOException {    
	response.setContentType("application/vnd.ms-excel");    
	response.setCharacterEncoding("utf-8");    
	response.setHeader("Content-disposition", "attachment;filename=demo.xlsx");    
	EasyExcel.write(response.getOutputStream(), DownloadData.class).sheet("模板").doWrite(data());
}
/**	excel文件的上传*/
@PostMapping("upload")
@ResponseBody
public String upload(MultipartFile file) throws IOException {    
	EasyExcel.read(file.getInputStream(), DemoData.class, new DemoDataListener()).sheet().doRead();    
	return "success";
}
```

复制代码



## 四、详解读取 Excel

### 简单读取

**对象**

```java
// 如果没有特殊说明，下面的案例将默认使用这个实体类
public class DemoData {    
	private String string;    
	private Date date;    
	private Double doubleData;    
	// getting setting
}
```

复制代码



**监听器**

```java
// 如果没有特殊说明，下面的案例将默认使用这个监听器
public class DemoDataListener extends AnalysisEventListener<DemoData> {
    List<DemoData> list = new ArrayList<DemoData>();        
    /**     * 如果使用了spring,请使用这个构造方法。每次创建Listener的时候需要把spring管理的类传进来     */    
    public DemoDataListener() {}
    /**     * 这个每一条数据解析都会来调用     *     * @param data     * @param context     */    
    @Override    
    public void invoke(DemoData data, AnalysisContext context) {        
    	System.out.println("解析到一条数据:{}", JSON.toJSONString(data));        
        list.add(data);    
    }
    /**     * 所有数据解析完成了 都会来调用     *     * @param context     */    @Override    public void doAfterAllAnalysed(AnalysisContext context) {        
        System.out.println(JSON.toJSONString(list));    
    }
}
```

复制代码



**代码**

```java
@Test
public void simpleRead() {    
	// 写法1：    
	String fileName = "demo.xlsx";    
	// 这里 需要指定读用哪个class去读，然后读取第一个sheet 文件流会自动关闭    
	EasyExcel.read(fileName, DemoData.class, new DemoDataListener()).sheet().doRead();
    // 写法2：    
    fileName = "demo.xlsx";    
    ExcelReader excelReader = EasyExcel.read(fileName, DemoData.class, new DemoDataListener()).build();    
    ReadSheet readSheet = EasyExcel.readSheet(0).build();    
    excelReader.read(readSheet);    
    // 这里千万别忘记关闭，读的时候会创建临时文件，到时磁盘会崩的    
    excelReader.finish();
}
```

复制代码



### 指定列的下标或名称

**对象**

```java
public class DemoData {    
    /**     * 强制读取第三个 这里不建议 index 和 name 同时用，要么一个对象只用index，要么一个对象只用name去匹配     */    
    @ExcelProperty(index = 2)    
    private Double doubleData;    
    /**     * 用名字去匹配，这里需要注意，如果名字重复，会导致只有一个字段读取到数据     */    
    @ExcelProperty("字符串标题")   
    private String string;    
    @ExcelProperty("日期标题")    
    private Date date;
}
```

复制代码



**代码**

```java
@Testpublic void indexOrNameRead() {    
	String fileName = "demo.xlsx";    
	// 这里默认读取第一个sheet    
	EasyExcel.read(fileName, DemoData.class, new DemoDataListener()).sheet().doRead();
}
```

复制代码



### 读取多个 sheet

**代码**

```java
@Test
public void repeatedRead() {    
    String fileName = "demo.xlsx";
    // 读取全部sheet    
    // 这里需要注意 DemoDataListener的doAfterAllAnalysed 会在每个sheet读取完毕后调用一次。然后所有sheet都会往同一个DemoDataListener里面
    EasyExcel.read(fileName, DemoData.class, new DemoDataListener()).doReadAll();
    // 读取部分sheet    
    fileName = "demo.xlsx";    
    ExcelReader excelReader = EasyExcel.read(fileName).build();    
    // 这里为了简单 所以注册了 同样的head 和Listener 自己使用功能必须不同的Listener    
    // readSheet参数设置读取sheet的序号    
    ReadSheet readSheet1 =EasyExcel.readSheet(0).head(DemoData.class).registerReadListener(new DemoDataListener()).build();    
    ReadSheet readSheet2 =EasyExcel.readSheet(1).head(DemoData.class).registerReadListener(new DemoDataListener()).build();    // 这里注意 一定要把sheet1 sheet2 一起传进去，不然有个问题就是03版的excel 会读取多次，浪费性能    
    excelReader.read(readSheet1, readSheet2);    
    // 这里千万别忘记关闭，读的时候会创建临时文件，到时磁盘会崩的    
    excelReader.finish();
}
```

复制代码



### 自定义格式转换

**对象**

```java
@Data
public class ConverterData {    
	/**     * converter属性定义自己的字符串转换器     */    
    @ExcelProperty(converter = CustomStringConverter.class)    private String string;    
    /**     * 这里用string 去接日期才能格式化     */    
    @DateTimeFormat("yyyy年MM月dd日 HH时mm分ss秒")    
    private String date;    
    /**     * 我想接收百分比的数字     */    
    @NumberFormat("#.##%")   
    private String doubleData;
}
```

复制代码



**自定义转换器**

```java
public class CustomStringStringConverter implements Converter<String> {    
    @Override    
    public Class supportJavaTypeKey() {        
        return String.class;    
    }
    @Override    
    public CellDataTypeEnum supportExcelTypeKey() {        
        return CellDataTypeEnum.STRING;    
    }
    /*** 这里读的时候会调用*     * @param cellData     *            NotNull     * @param contentProperty     *            Nullable     * @param globalConfiguration     *            NotNull     * @return     */    
    @Override    
    public String convertToJavaData(CellData cellData, ExcelContentProperty contentProperty,GlobalConfiguration globalConfiguration) {       
        return "自定义：" + cellData.getStringValue();    
    }
    /**     * 这里是写的时候会调用 不用管     *     * @param value     *            NotNull     * @param contentProperty     *            Nullable     * @param globalConfiguration     *            NotNull     * @return     */    
    @Override    
    public CellData convertToExcelData(String value, ExcelContentProperty contentProperty,GlobalConfiguration globalConfiguration) {        
        return new CellData(value);    
    }
}
```

复制代码



**代码**

```java
@Test
public void converterRead() {    
    String fileName = "demo.xlsx";    
    // 这里 需要指定读用哪个class去读，然后读取第一个sheet     
    EasyExcel.read(fileName, ConverterData.class, new ConverterDataListener())        
    // 这里注意 我们也可以registerConverter来指定自定义转换器， 但是这个转换变成全局了， 所有java为string,excel为string的都会用这个转换器。           // 如果就想单个字段使用请使用@ExcelProperty 指定converter        
    // .registerConverter(new CustomStringStringConverter())        // 读取sheet        
    .sheet().doRead();
}
```

复制代码



### 多行头

**代码**

```java
@Test
public void complexHeaderRead() {    
	String fileName = "demo.xlsx";    
	// 这里 需要指定读用哪个class去读，然后读取第一个sheet     
	EasyExcel.read(fileName, DemoData.class, new DemoDataListener()).sheet()        
	// 这里可以设置1，因为头就是一行。如果多行头，可以设置其他值。不传入默认1行        
	.headRowNumber(1).doRead();
}
```

复制代码



### 读取表头数据

**监听器**

```java
/** * 这里会一行行的返回头 * 监听器只需要重写这个方法就可以读取到头信息 * @param headMap * @param context */
@Override
public void invokeHeadMap(Map<Integer, String> headMap, AnalysisContext context) {    
    LOGGER.info("解析到一条头数据:{}", JSON.toJSONString(headMap));
}
```

复制代码



**代码**

```java
@Test
public void headerRead() {    
    String fileName = "demo.xlsx";    
    // 这里 需要指定读用哪个class去读，然后读取第一个sheet    
    EasyExcel.read(fileName, DemoData.class, new ReadDataListener()).sheet().doRead();
}
```

复制代码



### 异常处理

**监听器**

```java
/*** 监听器实现这个方法就可以在读取数据的时候获取到异常信息*/
@Override
public void onException(Exception exception, AnalysisContext context) {    
    LOGGER.error("解析失败，但是继续解析下一行:{}", exception.getMessage());    
    // 如果是某一个单元格的转换异常 能获取到具体行号    
    // 如果要获取头的信息 配合invokeHeadMap使用    
    if (exception instanceof ExcelDataConvertException) {        
        ExcelDataConvertException excelDataConvertException = (ExcelDataConvertException)exception;        
        LOGGER.error("第{}行，第{}列解析异常", excelDataConvertException.getRowIndex(),                excelDataConvertException.getColumnIndex());    
    }
}
```

复制代码



### web 读取

**代码**

```java
@PostMapping("upload")
@ResponseBody
public String upload(MultipartFile file) throws IOException {    
    EasyExcel.read(file.getInputStream(), UploadData.class, new UploadDataListener(uploadDAO)).sheet().doRead();    
    return "SUCCESS";
}
```

复制代码



## 五、详解写入 Excel

**数据写入公用方法**

```java
private List<DemoData> data() {    
	List<DemoData> list = new ArrayList<DemoData>();    
    for (int i = 0; i < 10; i++) {        
        DemoData data = new DemoData();        
        data.setString("字符串" + i);       
        data.setDate(new Date());        
        data.setDoubleData(0.56);        
        list.add(data);    
    }    
    return list;
}
```

复制代码



### 简单写入

**对象**

```java
public class DemoData {    
    @ExcelProperty("字符串标题")    
    private String string;    
    @ExcelProperty("日期标题")    
    private Date date;    
    @ExcelProperty("数字标题")    
    private Double doubleData;    
    // getting setting
}
```

复制代码



**代码**

```java
@Testpublic void simpleWrite() {    
	// 写法1    
	String System.currentTimeMillis() + ".xlsx";    
	// 这里 需要指定写用哪个class去写，然后写到第一个sheet，名字为模板 然后文件流会自动关闭    
	// 如果这里想使用03 则 传入excelType参数即可    
	EasyExcel.write(fileName, DemoData.class).sheet("写入方法一").doWrite(data());
    // 写法2，方法二需要手动关闭流    
    fileName = System.currentTimeMillis() + ".xlsx";    
    // 这里 需要指定写用哪个class去写    
    ExcelWriter excelWriter = EasyExcel.write(fileName, DemoData.class).build();    
    WriteSheet writeSheet = EasyExcel.writerSheet("写入方法二").build();    
    excelWriter.write(data(), writeSheet);    
    /// 千万别忘记finish 会帮忙关闭流   
    excelWriter.finish();
}
```

复制代码



### 导出指定的列

**代码**

```java
@Test
public void excludeOrIncludeWrite() {    
    String fileName = "excludeOrIncludeWrite" + System.currentTimeMillis() + ".xlsx";
    // 忽略 date 不导出    
    Set<String> excludeColumnFiledNames = new HashSet<String>();    
    excludeColumnFiledNames.add("date");    
    // 这里 需要指定写用哪个class去写，然后写到第一个sheet，名字为模板 然后文件流会自动关闭    
    EasyExcel.write(fileName, DemoData.class).excludeColumnFiledNames(excludeColumnFiledNames).sheet("忽略date")        .doWrite(data());
    fileName = "excludeOrIncludeWrite" + System.currentTimeMillis() + ".xlsx";    
    // 根据用户传入字段 假设我们只要导出 date    
    Set<String> includeColumnFiledNames = new HashSet<String>();    
    includeColumnFiledNames.add("date");    
    // 这里 需要指定写用哪个class去写，然后写到第一个sheet，名字为模板 然后文件流会自动关闭    
    EasyExcel.write(fileName, DemoData.class).includeColumnFiledNames(includeColumnFiledNames).sheet("导出date")        .doWrite(data());
}
```

复制代码



### 指定写入的列

**对象**

```java
public class IndexData {    
    /**    * 导出的excel第二列和第四列将空置    */    
    @ExcelProperty(value = "字符串标题", index = 0)    
    private String string;    
    @ExcelProperty(value = "日期标题", index = 2)    
    private Date date;    
    @ExcelProperty(value = "数字标题", index = 4)    
    private Double doubleData;
}
```

复制代码



**代码**

```java
@Test
public void indexWrite() {    
    String fileName = "indexWrite" + System.currentTimeMillis() + ".xlsx";    
    // 这里 需要指定写用哪个class去写，然后写到第一个sheet，名字为模板 然后文件流会自动关闭    
    EasyExcel.write(fileName, IndexData.class).sheet("模板").doWrite(data());
}
```

复制代码



### 复杂头写入

**对象**

```java
public class ComplexHeadData {    
    /**    * 主标题 将整合为一个单元格效果如下：    * —————————————————————————    * |          主标题        |    * —————————————————————————    * |字符串标题|日期标题|数字标题|    * —————————————————————————    */    
    @ExcelProperty({"主标题", "字符串标题"})    
    private String string;    
    @ExcelProperty({"主标题", "日期标题"})   
    private Date date;    
    @ExcelProperty({"主标题", "数字标题"})    
    private Double doubleData;
}
```

复制代码



**代码**

```java
@Testpublic void complexHeadWrite() {    
    String fileName = "complexHeadWrite" + System.currentTimeMillis() + ".xlsx";    
    // 这里 需要指定写用哪个class去写，然后写到第一个sheet，名字为模板 然后文件流会自动关闭    
    EasyExcel.write(fileName, ComplexHeadData.class).sheet("模板").doWrite(data());
}
```

复制代码



### 自定义格式转

```java
public class ConverterData {    
    /**     * 自定义的转换     */    
    @ExcelProperty(value = "字符串标题", converter = CustomStringConverter.class)    
    private String string;    
    /**     * 我想写到excel 用年月日的格式     */    
    @DateTimeFormat("yyyy年MM月dd日 HH时mm分ss秒")    
    @ExcelProperty("日期标题")    
    private Date date;    
    /**     * 我想写到excel 用百分比表示     */    
    @NumberFormat("#.##%")    
    @ExcelProperty(value = "数字标题")    
    private Double doubleData;    
    // getting setting
}
```

复制代码



**代码**

```java
@Testpublic void converterWrite() {    
    String "converterWrite" + System.currentTimeMillis() + ".xlsx";    
    // 这里 需要指定写用哪个class去写，然后写到第一个sheet，名字为模板 然后文件流会自动关闭    
    EasyExcel.write(fileName, ConverterData.class).sheet("模板").doWrite(data());
}
```

复制代码



### 图片导出

**对象**

```java
@Data@ContentRowHeight(200)@ColumnWidth(200 / 8)public class ImageData {    
    // 图片导出方式有5种    
    private File file;    
    private InputStream inputStream;    
    /**     * 如果string类型 必须指定转换器，string默认转换成string，该转换器是官方支持的     */   
    @ExcelProperty(converter = StringImageConverter.class)    
    private String string;    
    private byte[] byteArray;    /**     * 根据url导出 版本2.1.1才支持该种模式     */    private URL url;}
```

复制代码



**代码**

```java
@Test
public void imageWrite() throws Exception {    
    String fileName = "imageWrite" + System.currentTimeMillis() + ".xlsx";    
    // 如果使用流 记得关闭    
    InputStream inputStream = null;    
    try {        
        List<ImageData> list = new ArrayList<ImageData>();        
        ImageData imageData = new ImageData();        
        list.add(imageData);        
        String imagePath = "converter" + File.separator + "img.jpg";        
        // 放入五种类型的图片 根据实际使用只要选一种即可        
        imageData.setByteArray(FileUtils.readFileToByteArray(new File(imagePath)));        
        imageData.setFile(new File(imagePath));        
        imageData.setString(imagePath);        
        inputStream = FileUtils.openInputStream(new File(imagePath));        
        imageData.setInputStream(inputStream);        
        imageData.setUrl(new URL(            "https://raw.githubusercontent.com/alibaba/easyexcel/master/src/test/resources/converter/img.jpg"));        		    				EasyExcel.write(fileName, ImageData.class).sheet().doWrite(list);    
    } 
    finally {        
        if (inputStream != null) {            
            inputStream.close();        
        }    
    }
}
```

复制代码



### 列宽、行高

**对象**

```java
@Data
@ContentRowHeight(10)
@HeadRowHeight(20)
@ColumnWidth(25)
public class WidthAndHeightData {    
    @ExcelProperty("字符串标题")    
    private String string;    
    @ExcelProperty("日期标题")    
    private Date date;    
    /**     * 宽度为50,覆盖上面的宽度25     */    
    @ColumnWidth(50)    
    @ExcelProperty("数字标题")    
    private Double doubleData;
}
```

复制代码



**代码**

```java
@Test
public void widthAndHeightWrite() {    
    String fileName = "widthAndHeightWrite" + System.currentTimeMillis() + ".xlsx";    
    // 这里 需要指定写用哪个class去写，然后写到第一个sheet，名字为模板 然后文件流会自动关闭    
    EasyExcel.write(fileName, WidthAndHeightData.class).sheet("模板").doWrite(data());
}
```

复制代码



### 合并单元格

**代码**

```java
 @Test public void mergeWrite() {     
     String fileName = "mergeWrite" + System.currentTimeMillis() + ".xlsx";     
     // 每隔2行会合并 把eachColumn 设置成 3 也就是我们数据的长度，所以就第一列会合并。当然其他合并策略也可以自己写     
     LoopMergeStrategy loopMergeStrategy = new LoopMergeStrategy(2, 0);     
     // 这里 需要指定写用哪个class去写，然后写到第一个sheet，名字为模板 然后文件流会自动关闭     
     EasyExcel.write(fileName, DemoData.class).registerWriteHandler(loopMergeStrategy).sheet("合并单元格")         .doWrite(data()); 
 }
```

复制代码



### 动态表头

**代码**

```java
@Test
public void dynamicHeadWrite() {    
    String fileName = "dynamicHeadWrite" + System.currentTimeMillis() + ".xlsx";    
    EasyExcel.write(fileName)        
    // 这里放入动态头        
    .head(head()).sheet("模板")        
    // 当然这里数据也可以用 List<List<String>> 去传入        
    .doWrite(data());
}
	// 动态表头的数据格式List<List<String>>
    private List<List<String>> head() {    
        List<List<String>> list = new ArrayList<List<String>>();    
        List<String> head0 = new ArrayList<String>();    
        head0.add("字符串" + System.currentTimeMillis());    
        List<String> head1 = new ArrayList<String>();   
        head1.add("数字" + System.currentTimeMillis());    
        List<String> head2 = new ArrayList<String>();    
        head2.add("日期" + System.currentTimeMillis());    
        list.add(head0);    
        list.add(head1);    
        list.add(head2);    
        return list;
    }
```

复制代码



### web 数据写出

**代码**

```java
@GetMapping("download")
public void download(HttpServletResponse response) throws IOException {    
    response.setContentType("application/vnd.ms-excel");    
    response.setCharacterEncoding("utf-8");    
    // 这里URLEncoder.encode可以防止中文乱码 当然和easyexcel没有关系    
    String fileName = URLEncoder.encode("数据写出", "UTF-8");    
    response.setHeader("Content-disposition", "attachment;filename=" + fileName + ".xlsx");    			    			    		EasyExcel.write(response.getOutputStream(), DownloadData.class).sheet("模板").doWrite(data());
}
```

复制代码



## 六、详解填充样板写入

这里的案例填充都是模板向下，如果需要横向填充只需要模板设置好就可以。

### 简单的填充

**Excel 模板**

![img](https://static001.geekbang.org/infoq/6d/6d6aec4ee6da6bc8b6c2a831d9333f3b.png)



**Excel 最终效果**

![img](https://static001.geekbang.org/infoq/d3/d3ef6d0c93dc37ad3fcdd5428bb27327.png)



**对象**

```java
public class FillData {    
    private String name;    
    private double number;    
    // getting setting
}
```

复制代码



**代码**

```java
@Test
public void simpleFill() {    
    // 模板注意 用{} 来表示你要用的变量 如果本来就有"{","}" 特殊字符 用"\{","\}"代替    
    String templateFileName = "simple.xlsx";
    // 方案1 根据对象填充    
    String fileName = System.currentTimeMillis() + ".xlsx";    
    // 这里 会填充到第一个sheet， 然后文件流会自动关闭    
    FillData fillData = new FillData();    
    fillData.setName("知春秋");    
    fillData.setNumber(25);    
    EasyExcel.write(fileName).withTemplate(templateFileName).sheet().doFill(fillData);
    // 方案2 根据Map填充    
    fileName = System.currentTimeMillis() + ".xlsx";    
    // 这里 会填充到第一个sheet， 然后文件流会自动关闭    
    Map<String, Object> map = new HashMap<String, Object>();    
    map.put("name", "知春秋");    
    map.put("number", 25);    
    EasyExcel.write(fileName).withTemplate(templateFileName).sheet().doFill(map);
}
```

复制代码



### 复杂的填充

使用 List 集合的方法批量写入数据，点表示该参数是集合

**Excel 模板**

![img](https://static001.geekbang.org/infoq/e8/e86fb32547bc717742625e7c82d1df98.png)



**Excel 最终效果**

![img](https://static001.geekbang.org/infoq/c9/c98255b4ded1bad9475df1506b0f4a0b.png)



**代码**

```java
@Test
public void complexFill() {    
    // 模板注意 用{} 来表示你要用的变量 如果本来就有"{","}" 特殊字符 用"\{","\}"代替    
    // {} 代表普通变量 {.} 代表是list的变量    
    String templateFileName = "complex.xlsx";
    String fileName = System.currentTimeMillis() + ".xlsx";    
    ExcelWriter excelWriter = EasyExcel.write(fileName).withTemplate(templateFileName).build();    
    WriteSheet writeSheet = EasyExcel.writerSheet().build();    
    // 这里注意 入参用了forceNewRow 代表在写入list的时候不管list下面有没有空行 都会创建一行，然后下面的数据往后移动。默认 是false，会直接使用下一行，如果没有则创建。    
    // forceNewRow 如果设置了true,有个缺点 就是他会把所有的数据都放到内存了，所以慎用    
    // 简单的说 如果你的模板有list,且list不是最后一行，下面还有数据需要填充 就必须设置 forceNewRow=true 但是这个就会把所有数据放到内存 会很耗内存    // 如果数据量大 list不是最后一行 参照下一个   
    FillConfig fillConfig = FillConfig.builder().forceNewRow(Boolean.TRUE).build();    
    excelWriter.fill(data(), fillConfig, writeSheet);    
    excelWriter.fill(data(), fillConfig, writeSheet);    
    // 其他参数可以使用Map封装    
    Map<String, Object> map = new HashMap<String, Object>();    
    excelWriter.fill(map, writeSheet);    
    excelWriter.finish();
}
```

复制代码



## 七、API

### 详细参数介绍

#### 关于常见类解析

Ø EasyExcel 入口类，用于构建开始各种操作

Ø ExcelReaderBuilder ExcelWriterBuilder 构建出一个 ReadWorkbook WriteWorkbook，可以理解成一个 excel 对象，一个 excel 只要构建一个

Ø ExcelReaderSheetBuilder ExcelWriterSheetBuilder 构建出一个 ReadSheet WriteSheet 对象，可以理解成 excel 里面的一页,每一页都要构建一个

Ø ReadListener 在每一行读取完毕后都会调用 ReadListener 来处理数据

Ø WriteHandler 在每一个操作包括创建单元格、创建表格等都会调用 WriteHandler 来处理数据

Ø 所有配置都是继承的，Workbook 的配置会被 Sheet 继承，所以在用 EasyExcel 设置参数的时候，在 EasyExcel…sheet()方法之前作用域是整个 sheet,之后针对单个 sheet



#### 读

##### 注解

Ø ExcelProperty 指定当前字段对应 excel 中的那一列。可以根据名字或者 Index 去匹配。当然也可以不写，默认第一个字段就是 index=0，以此类推。千万注意，要么全部不写，要么全部用 index，要么全部用名字去匹配。千万别三个混着用，除非你非常了解源代码中三个混着用怎么去排序的。

Ø ExcelIgnore 默认所有字段都会和 excel 去匹配，加了这个注解会忽略该字段

Ø DateTimeFormat 日期转换，用 String 去接收 excel 日期格式的数据会调用这个注解。里面的 value 参照 java.text.SimpleDateFormat

Ø NumberFormat 数字转换，用 String 去接收 excel 数字格式的数据会调用这个注解。里面的 value 参照 java.text.DecimalFormat

Ø ExcelIgnoreUnannotated 默认不加 ExcelProperty 的注解的都会参与读写，加了不会参与



##### 参数

**通用参数**

Ø ReadWorkbook,ReadSheet 都会有的参数，如果为空，默认使用上级。

Ø converter 转换器，默认加载了很多转换器。也可以自定义。

Ø readListener 监听器，在读取数据的过程中会不断的调用监听器。

Ø headRowNumber 需要读的表格有几行头数据。默认有一行头，也就是认为第二行开始起为数据。

Ø head 与 clazz 二选一。读取文件头对应的列表，会根据列表匹配数据，建议使用 class。

Ø clazz 与 head 二选一。读取文件的头对应的 class，也可以使用注解。如果两个都不指定，则会读取全部数据。

Ø autoTrim 字符串、表头等数据自动 trim

Ø password 读的时候是否需要使用密码



**ReadWorkbook（理解成 excel 对象）参数**

Ø excelType 当前 excel 的类型 默认会自动判断

Ø inputStream 与 file 二选一。读取文件的流，如果接收到的是流就只用，不用流建议使用 file 参数。因为使用了 inputStream easyexcel 会帮忙创建临时文件，最终还是 file

Ø file 与 inputStream 二选一。读取文件的文件。

Ø autoCloseStream 自动关闭流。

Ø readCache 默认小于 5M 用 内存，超过 5M 会使用 EhCache,这里不建议使用这个参数。



**ReadSheet（就是 excel 的一个 Sheet）参数**

Ø sheetNo 需要读取 Sheet 的编码，建议使用这个来指定读取哪个 Sheet

Ø sheetName 根据名字去匹配 Sheet,excel 2003 不支持根据名字去匹配



#### 写

##### 注解

Ø ExcelProperty index 指定写到第几列，默认根据成员变量排序。value 指定写入的名称，默认成员变量的名字，多个 value 可以参照快速开始中的复杂头

Ø ExcelIgnore 默认所有字段都会写入 excel，这个注解会忽略这个字段

Ø DateTimeFormat 日期转换，将 Date 写到 excel 会调用这个注解。里面的 value 参照 java.text.SimpleDateFormat

Ø NumberFormat 数字转换，用 Number 写 excel 会调用这个注解。里面的 value 参照 java.text.DecimalFormat

Ø ExcelIgnoreUnannotated 默认不加 ExcelProperty 的注解的都会参与读写，加了不会参与

##### 参数

**通用参数**

Ø WriteWorkbook,WriteSheet ,WriteTable 都会有的参数，如果为空，默认使用上级。

Ø converter 转换器，默认加载了很多转换器。也可以自定义。

Ø writeHandler 写的处理器。可以实现 WorkbookWriteHandler,SheetWriteHandler,RowWriteHandler,CellWriteHandler，在写入 excel 的不同阶段会调用

Ø relativeHeadRowIndex 距离多少行后开始。也就是开头空几行

Ø needHead 是否导出头

Ø head 与 clazz 二选一。写入文件的头列表，建议使用 class。

Ø clazz 与 head 二选一。写入文件的头对应的 class，也可以使用注解。

Ø autoTrim 字符串、表头等数据自动 trim



**WriteWorkbook（理解成 excel 对象）参数**

Ø excelType 当前 excel 的类型 默认 xlsx

Ø outputStream 与 file 二选一。写入文件的流

Ø file 与 outputStream 二选一。写入的文件

Ø templateInputStream 模板的文件流

Ø templateFile 模板文件

Ø autoCloseStream 自动关闭流。

Ø password 写的时候是否需要使用密码

Ø useDefaultStyle 写的时候是否是使用默认头



**WriteSheet（就是 excel 的一个 Sheet）参数**

Ø sheetNo 需要写入的编码。默认 0

Ø sheetName 需要些的 Sheet 名称，默认同 sheetNo



**WriteTable（就把 excel 的一个 Sheet,一块区域看一个 table）参数**

Ø tableNo 需要写入的编码。默认 0