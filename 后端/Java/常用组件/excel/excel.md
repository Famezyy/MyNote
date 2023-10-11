# EasyExcel

## 1.EasyExcel解决了什么

主要来说，有以下几点：

- 传统 Excel 框架，如 Apache poi、jxl 都存在内存溢出的问题
- 传统 Excel 开源框架使用复杂、繁琐
- EasyExcel 底层还是使用了 poi，但是做了很多优化，如修复了并发情况下的一些 bug，具体修复细节，可阅读<a href="https://github.com/alibaba/easyexcel">官方文档</a>

## 2.快速上手

### 2.1 添加依赖

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>easyexcel</artifactId>
    <version>1.1.2-beta5</version>
</dependency>
```

### 2.2 七行代码搞定

```java
@Test
void testExcel() throws IOException {
// 文件输出位置
FileOutputStream fos = new FileOutputStream("C:\\Users\\Administrator\\Desktop\\test.xlsx");
ExcelWriter writer = EasyExcelFactory.getWriter(fos);

// 写仅有一个 sheet 的 Excel
Sheet sheet = new Sheet(1, 0, WriteModel.class);
// 第一个 sheet 的名称
sheet.setSheetName("first sheet");
    
// 写数据到 writer 上下文中
// 入参1：创建要写入的数据模型
// 入参2：要写入的目标 sheet
writer.write(createModelList(), sheet);

// 将上下文中的最终 outputStream 写入到指定文件中
writer.finish();

fos.close();
}
```

上面这段示例代码中，有两个点很重要，小哈已经重点标注标：

- WriteModel 这个对象就是要写入 Excel 的数据模型对象
- 创建需要写入的数据集，正常业务中，这块都是从数据库中查询出来的

> PS：如果说写入的数据量很大，需要做分片查询再写入的处理，否则可能会 OOM

回过头来，我们来看看 `WriteModel` 这个对象内部到底有什么幺蛾子！

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
class WriteModel extends BaseRowModel {
    @ExcelProperty(value = "姓名", index = 0)
    private String name;
    @ExcelProperty(value = "密码", index = 1)
    private String password;
    @ExcelProperty(value = "年龄", index = 2)
    private Integer age;
}
```

ExayExcel 提供注解的方式, 来方便的定义 Excel 需要的数据模型：

- 定义的写入模型必须要继承自 `BaseRowModel.java`
- 通过 `@ExcelProperty` 注解来指定每个字段的**列名称**，以及**下标位置**

同时，上面定义的 `createModelList()` 方法也很简单，通过循环，创建一个写入模型的 List 集合：

```java
private List<WriteModel> createModelList() {
    List<WriteModel> writeModels = new ArrayList<>();
    for (int i = 0; i < 10; i++) {
        WriteModel writeModel = WriteModel.builder()
            .name("name" + i).password("123456").age(i).build();
        writeModels.add(writeModel);
    }
    return writeModels;
}
```

看下实际效果：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220430173940062-e90dbf7870a1e68f5151e50f1d6a17ff-a8d631.png" alt="image-20220430173940062" style="zoom:80%;" />

## 3.特殊场景支持

在实际的业务中，我们还会有一些特需的需求，比如说下面这些。

### 3.1 动态生成 Excel 内容

上面的例子是基于注解的，也就是说表头 head，以及内容都是写死的，换句话说，我定义好了一个数据模型，那么，生成的 Excel 文件也就是只能遵循这种模型来了，但是，实际业务中可能会存在动态变化的需求，要怎么做呢？

```java
void testExcel() throws IOException {
    FileOutputStream fos = new FileOutputStream("C:\\Users\\Administrator\\Desktop\\test.xlsx");
    ExcelWriter writer = EasyExcelFactory.getWriter(fos);

    // 动态添加表头
    Sheet sheet = new Sheet(1, 0);
    sheet.setSheetName("first sheet");

    // 创建一个表格，用于 sheet 中使用
    Table table = new Table(1);
    // 无注解模式，动态添加表头
    table.setHead(createTestListStringHead());
    // 写数据
    writer.write1(createDynamicModelList(), sheet, table);

    writer.finish();
    fos.close();
}
```

> `write0()`和`write1()`和区别：
>
> ```java
> /**
>   *
>   * Write data to a sheet
>   * @param data Data to be written
>   * @param sheet Write to this sheet
>   * @return this
>   */
> public ExcelWriter write1(List<List<Object>> data, Sheet sheet) {
>     excelBuilder.addContent(data, sheet);
>     return this;
> }
> 
> /**
>   * Write data to a sheet
>   * @param data  Data to be written
>   * @param sheet Write to this sheet
>   * @return this
>   */
> public ExcelWriter write0(List<List<String>> data, Sheet sheet) {
>     excelBuilder.addContent(data, sheet);
>     return this;
> }
> ```

无注解模式，动态添加表头，也可自由组合复杂表头，代码如下：

```java
private List<List<String>> createTestListStringHead() {
    // 模型上没有注解，表头数据动态传入
    List<List<String>> head = new ArrayList<>();
    List<String> headCol1 = new ArrayList<>();
    List<String> headCol2 = new ArrayList<>();
    List<String> headCol3 = new ArrayList<>();
    List<String> headCol4 = new ArrayList<>();
    List<String> headCol5 = new ArrayList<>();
    
    headCol1.add("firstRow");headCol1.add("firstRow");headCol1.add("firstRow");
    headCol2.add("firstRow");headCol2.add("firstRow");headCol2.add("firstRow");
    
    headCol3.add("secondRow");headCol3.add("secondRow");headCol3.add("secondRow");
    headCol4.add("thirdRow");headCol3.add("thirdRow2");headCol3.add("thirdRow2");
    headCol5.add("firstRow");headCol5.add("thirdRow");headCol5.add("forthRow");
    
    head.add(headCol1);
    head.add(headCol2);
    head.add(headCol3);
    head.add(headCol4);
    head.add(headCol5);
    
    return head;
}
```

创建动态数据，注意这里的数据类型是 `Object`：

```java
private List<List<Object>> createDynamicModelList() {
    List<List<Object>> rows = new ArrayList<>();
    for (int i = 0; i < 10; i++) {
        List<Object> row = new ArrayList<>();
        row.add(i);
        row.add(new Random().nextInt(1000));
        row.add(new Random().nextInt(100));
        row.add("test");
        row.add("ttt");
        rows.add(row);
    }
    return rows;
}
```

看下效果：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220430182441022-34983aecf64e9d529f35114a87823941-43f31b.png" alt="image-20220430182441022" style="zoom:80%;" />

### 3.2 自定义表头以及内容样式

我想自定义表头，内容样式，咋办？

```java
void testExcel() throws IOException {
    FileOutputStream fos = new FileOutputStream("C:\\Users\\Administrator\\Desktop\\test.xlsx");
    ExcelWriter writer = EasyExcelFactory.getWriter(fos);
    Sheet sheet = new Sheet(1, 0);
    sheet.setSheetName("first sheet");
    Table table = new Table(1);

    // 自定义表格样式
    table.setTableStyle(createTableStyle());

    table.setHead(createTestListStringHead());
    writer.write1(createDynamicModelList(), sheet, table);
    writer.finish();
    fos.close();
}
```

我们复用了上面的示例代码，并额外添加了设置自定义表格样式的代码，`createTableStytle()`具体内容如下：

```java
TableStyle createTableStyle() {
    TableStyle tableStyle = new TableStyle();
    // 设置表头样式
    Font headFont = new Font();
    // 字体是否加粗
    headFont.setBold(true);
    // 字体大小
    headFont.setFontHeightInPoints((short)12);
    // 字体
    headFont.setFontName("楷体");
    tableStyle.setTableHeadFont(headFont);
    // 背景色
    tableStyle.setTableHeadBackGroundColor(IndexedColors.BLUE);
    
    // 设置表格主题样式
    Font contentFont = new Font();
    contentFont.setBold(false);
    contentFont.setFontHeightInPoints((short)10);
    contentFont.setFontName("黑体");
    tableStyle.setTableContentFont(contentFont);
    tableStyle.setTableContentBackGroundColor(IndexedColors.YELLOW);
    
    return tableStyle;
}
```

我们可以通过 `TableStyle` 这个类来设置表头、表格主题的样式，效果见下图：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220430180427300-3e338d5e2f722af1e3da8a928ebbb9ae-a2637a.png" alt="image-20220430180427300" style="zoom:80%;" />

### 3.3 合并单元格

我们可以通过 `merge()` 方法来合并单元格：

```java
writer.write1(createDynamicModelList(), sheet, table);

// 合并单元格
// firstRow, lastRow, firstCol, lastCol
writer.merge(5, 6, 0, 4);

writer.finish();
```

注意下标是从 0 开始的，也就是说合并了第六行到第七行，其中的第一列到第五列，跑下代码，看下效果：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220430182927792-453aa03ebc276ed1071b99d47cc5271d-f46bd9.png" alt="image-20220430182927792" style="zoom:80%;" />

### 3.4 自定义处理

对于更复杂的处理，EasyExcel 预留了 `WriterHandler` 接口来，允许你自定义处理代码：

```java
public interface WriteHandler {
    /**
     * Custom operation after creating each sheet
     *
     * @param sheetNo
     * @param sheet
     */
    void sheet(int sheetNo, Sheet sheet);

    /**
     * Custom operation after creating each row
     *
     * @param rowNum
     * @param row
     */
    void row(int rowNum, Row row);

    /**
     * Custom operation after creating each cell
     * @param cellNum
     * @param cell
     */
    void cell(int cellNum, Cell cell);
}
```

接口中定义了三个方法:

- `sheet()`：在创建每个 sheet 后自定义业务逻辑处理
- `row()`：在创建每个 row 后自定义业务逻辑处理
- `cell()`：在创建每个 cell 后自定义业务逻辑处理

我们实现了该接口后，编写自定义逻辑处理代码，然后调用 `getWriterWithTempAndHandler()`静态方法获取 `ExcelWriter` 对象时，传入 `WriterHandler` 的实现类即可。

```java
/**
  * Get ExcelWriter with a template file
  *
  * @param temp         Append data after a POI file, Can be null（the template POI filesystem that contains the Workbook stream)
  * @param outputStream the java OutputStream you wish to write the data to
  * @param typeEnum     03 or 07
  * @param needHead
  * @param handler      User-defined callback
  * @return new  ExcelWriter
  */
public static ExcelWriter getWriterWithTempAndHandler(InputStream temp,
                                                      OutputStream outputStream,
                                                      ExcelTypeEnum typeEnum,
                                                      boolean needHead,
                                                      WriteHandler handler) {
    return new ExcelWriter(temp, outputStream, typeEnum, needHead, handler);
}
```

比如下面的示例代码：

```java
ExcelWriter excelWriter = EasyExcelFactory.getWriterWithTempAndHandler(null, fos, ExcelTypeEnum.XLSX, true, new MyWriterHandler());
```

## 4.需要注意的点

### 4.1 写入大数据时，需分片

比如说，我们需要从数据库中查询出数据量较大时，我们需要在业务层做分片处理，也就是，我们需要分多次查询，再写入，防止内存溢出 OOM。

### 4.2 Excel 最大行数问题

Excel 03, 07 版本均有行数、列数的限制：

| 版本       | 最大行  | 最大列 |
| :--------- | :------ | :----- |
| Excel 2003 | 65536   | 256    |
| Excel 2007 | 1048576 | 16384  |

> csv 由于是文本文件，实际上没有最大行数的限制，但是用 Excel 客户端打开还是多了不显示。

也就是说，如果你想写入更多的行数是不行的，强行这么做，程序会报类似如下异常：

```bash
invalid row number (1048576) outside allowable range (0..1048575)
```

**如何解决呢？**

- 分多个 Excel 文件写入
- 同一个 Excel 文件，分多个 Sheet 写入
