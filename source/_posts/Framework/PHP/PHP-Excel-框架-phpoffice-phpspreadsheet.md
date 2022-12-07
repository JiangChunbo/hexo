---
title: PHP Excel 框架 phpoffice/phpspreadsheet
date: 2022-11-15 13:16:22
tags:
- Excel
---

# phpoffice/phpspreadsheet

[https://github.com/PHPOffice/PhpSpreadsheet](https://github.com/PHPOffice/PhpSpreadsheet)

# composer 依赖

通过 composer 导入相关依赖
```bash
composer require phpoffice/phpspreadsheet
```

```json
"require": {
  "phpoffice/phpspreadsheet": "^1.6"
}
```


# 读取 Excel
```php
$filename='';
// 创建 Reader
$reader= \PhpOffice\PhpSpreadsheet\IOFactory::createReader('Xlsx'); 
//也可以使用文件名，自动识别后缀，创建 Reader
$objReader = \PhpOffice\PhpSpreadsheet\IOFactory::createReaderForFile($filename);
// 如果只读，提高效率
$reader->setReadDataOnly(true);
// $filename 相对路径或绝对路径
$spreadsheet = $reader->load($filename);
```

# 常用 API 记录
## [Spreadsheet](https://phpoffice.github.io/PhpSpreadsheet/classes/PhpOffice-PhpSpreadsheet-Spreadsheet.html)
|函数|描述|
|:---|:---|
|[createSheet()](#method_Spreadsheet_createSheet)|创建新工作簿|
|[getActiveSheet()](#method_Spreadsheet_getActiveSheet)|获得当前活动的 Worksheet|

### <span id="method_Spreadsheet_createSheet">createSheet()</span>

<table>
<thead><tr align="left"><th>
createSheet
</th></tr></thead>
<tbody>
<tr><td>
创建 Worksheet 并将其添加到该 workbook

```php
public createSheet([null|int $sheetIndex = null ]) : Worksheet
```

</td></tr>
</tbody>
</table>


### <span id="method_Spreadsheet_getActiveSheet">getActiveSheet()</span>

<table>
<thead><tr align="left"><th>
getActiveSheet
</th></tr></thead>
<tbody>
<tr><td>
获得当前活动的 Worksheet

```php
public getActiveSheet() : Worksheet
```

</td></tr>
</tbody>
</table>





## Worksheet

|函数|描述|
|:---|:---|
|[getCell()](#method_getCell)|以指定坐标获得单元格|
|[getCellByColumnAndRow()](#method_getCellByColumnAndRow)|通过数字形式的单元格坐标以指定坐标获得单元格|
|[getColumnDimension()](#method_getColumnDimension)|以指定列获得列维度|
|[getHighestRow()](#method_getHighestRow)|获得最大的工作簿行数|
|getRowDimension()|以指定行获得行维度|
|[getStyle()](#method_getStyle)|获得单元格的样式|
|[getTitle()](#method_getTitle)|获得标题|
|[getDataValidation()](#method_getDataValidation)|获得数据验证|
|[mergeCells()](#method_mergeCells)|在单元格范围内设置合并|
|[mergeCellsByColumnAndRow()](#method_mergeCellsByColumnAndRow)|通过数字形式的单元格坐标在单元格范围内设置合并|
|[setTitle()](#method_setTitle)|设置工作簿的标题|
|setCellValue()|设置单元格的值|
|setCellValueByColumnAndRow()|通过数字形式的单元格坐标设置单元格值|



### <span id="method_getCell">getCell()</span>

<table>
<thead><tr align="left"><th>
getCell
</th></tr></thead>
<tbody>
<tr><td>
以指定坐标获得单元格

```php
public getCell(array<string|int, int>|CellAddress|string $coordinate) : Cell
```


</td></tr>
</tbody>
</table>


### <span id="method_getCellByColumnAndRow">getCellByColumnAndRow()</span>

<table>
<thead><tr align="left"><th>
getCellByColumnAndRow
</th></tr></thead>
<tbody>
<tr><td>
通过数字形式的单元格坐标以指定坐标获得单元格

```php
public getCellByColumnAndRow(int $columnIndex, int $row) : Cell
```

</td></tr>
</tbody>
</table>


### <span id="method_getColumnDimension">getColumnDimension()</span>

<table>
<thead><tr align="left"><th>
getColumnDimension
</th></tr></thead>
<tbody>
<tr><td>
以指定列获得列维度

```php
public getColumnDimension(string $column) : ColumnDimension
```

</td></tr>
</tbody>
</table>


### <span id="method_getHighestRow">getHighestRow()</span>

<table>
<thead><tr align="left"><th>
getHighestRow
</th></tr></thead>
<tbody>
<tr><td>
获得最大的工作簿行数

```php
public getHighestRow([null|string $column = null ]) : int
```

</td></tr>
</tbody>
</table>

### <span id="method_getRowDimension">getRowDimension()</span>

<table>
<thead><tr align="left"><th>
getRowDimension
</th></tr></thead>
<tbody>
<tr><td>
以指定行获得行维度

```php
public getRowDimension(int $row) : RowDimension
```

</td></tr>
</tbody>
</table>


### <span id="method_getStyle">getStyle()</span>

<table>
<thead><tr align="left"><th>
getStyle
</th></tr></thead>
<tbody>
<tr><td>
以指定行获得行维度

```php
public getStyle(AddressRange|array<string|int, int>|CellAddress|int|string $cellCoordinate) : Style
```

如果要获取整列单元格的样式，则仅仅传入列名
注意：对于列名是大写字母，而不是索引数字
```php
$style = $worksheet->getStyle('C');
```

</td></tr>
</tbody>
</table>



### <span id="method_getTitle">getTitle()</span>

<table>
<thead><tr align="left"><th>
getTitle
</th></tr></thead>
<tbody>
<tr><td>
获得标题

```php
public getTitle() : string
```

</td></tr>
</tbody>
</table>


### <span id="method_getDataValidation">getDataValidation()</span>

<table>
<thead><tr align="left"><th>
getDataValidation
</th></tr></thead>
<tbody>
<tr><td>
获得数据验证

```php
public getDataValidation(string $cellCoordinate) : DataValidation
```

如果要获取整列的单元格，则这样传入参数:

```php
$worksheet->getDataValidation('$A:$A');
```

</td></tr>
</tbody>
</table>



### <span id="method_mergeCells">mergeCells()</span>

<table>
<thead><tr align="left"><th>
mergeCells
</th></tr></thead>
<tbody>
<tr><td>
在单元格范围内设置合并

```php
public mergeCells(AddressRange|array<string|int, int>|string $range[, string $behaviour = self::MERGE_CELL_CONTENT_EMPTY ]) : $this
```

支持合并多行多列，例如:

```php
$worksheet->mergeCells('A1:E3');
```

</td></tr>
</tbody>
</table>





### <span id="method_mergeCellsByColumnAndRow">mergeCellsByColumnAndRow()</span>

<table>
<thead><tr align="left"><th>
mergeCellsByColumnAndRow
</th></tr></thead>
<tbody>
<tr><td>
通过数字形式的单元格坐标设置单元格值

```php
public mergeCellsByColumnAndRow(int $columnIndex1, int $row1, int $columnIndex2, int $row2[, string $behaviour = self::MERGE_CELL_CONTENT_EMPTY ]) : $this
```


支持合并多行多列，例如:

```php
$worksheet->mergeCellsByColumnAndRow(1, 1, 3, 3);
```

</td></tr>
</tbody>
</table>





### <span id="method_setTitle">setTitle()</span>

<table>
<thead><tr align="left"><th>
setTitle
</th></tr></thead>
<tbody>
<tr><td>
设置标题

```php
public setTitle(string $title[, bool $updateFormulaCellReferences = true ][, bool $validate = true ]) : $this
```

</td></tr>
</tbody>
</table>

### <span id="method_setCellValue">setCellValue()</span>

<table>
<thead><tr align="left"><th>
setCellValue
</th></tr></thead>
<tbody>
<tr><td>
设置单元格的值

```php
public setCellValue(array<string|int, int>|CellAddress|string $coordinate, mixed $value) : $this
```

</td></tr>
</tbody>
</table>

### <span id="method_setCellValueByColumnAndRow">setCellValueByColumnAndRow()</span>

<table>
<thead><tr align="left"><th>
setCellValueByColumnAndRow
</th></tr></thead>
<tbody>
<tr><td>
以数字形式的单元格坐标设置单元格值

```php
public setCellValueByColumnAndRow(int $columnIndex, int $row, mixed $value) : $this
```

</td></tr>
</tbody>
</table>


## RowDimension
|函数|描述|应用|
|:---|:---|:---|
|setRowHeight()|设置行高|



### <span id="method_rowdimension_setRowHeight">setRowHeight()</span>

<table>
<thead><tr align="left"><th>
setRowHeight
</th></tr></thead>
<tbody>
<tr><td>
设置行高

```php
public setRowHeight(float $height[, string|null $unitOfMeasure = null ]) : $this
```

设置行高示例:

```php
// 设置行高
// 行高这里传入索引数字，从 1 开始
$worksheet->getRowDimension($row)->setRowHeight(13.2);

> 注意，设置行高并不是绝对准确的，excel 会适当缩放
```

</td></tr>
</tbody>
</table>

## ColumnDimension 

|函数|描述|
|:---|:---|
|setWidth()|设置宽度|

### <span id="method_columndimension_setWidth">setWidth()</span>

<table>
<thead><tr align="left"><th>
setWidth
</th></tr></thead>
<tbody>
<tr><td>
设置宽度

```php
public setWidth(float $width[, string|null $unitOfMeasure = null ]) : $this
```

示例，设置列宽:

```php
$worksheet->getColumnDimensionByColumn(1)->setWidth(8);
$worksheet->getColumnDimension('A')->setWidth(8);
```

> 注意，设置列宽并不是绝对准确的，excel 会适当缩放
</td></tr>
</tbody>
</table>


## Style
|函数|描述|
|:---|:---|
|getAlignment()|获得对齐方式||
|getFont()|获得字体|
|getBorders()|获得边框|

```php
// 设置边框
$worksheet->getStyleByColumnAndRow($col, $row)->getBorders()->getAllBorders()->setBorderStyle(Border::BORDER_THIN);
```

### Borders
设置全部边框
```php
$worksheet->getStyle(
	'A1:' .
	$worksheet->getHighestColumn() .
	$worksheet->getHighestRow()
)->getBorders()->getAllBorders()->setBorderStyle(Border::BORDER_THIN);
```
## Alignment
|函数|描述|
|:---|:---|
|[setHorizontal()](#method_alignment_setHorizontal)|设置水平方式|
|[setHorizontal()](#method_alignment_setHorizontal)|设置水平方式|
|setWrapText()|设置是否自动换行|


### <span id="method_alignment_setHorizontal">setHorizontal()</span>

<table>
<thead><tr align="left"><th>
setHorizontal
</th></tr></thead>
<tbody>
<tr><td>
设置水平方式

```php
public setHorizontal(string $horizontalAlignment) : $this
```

**参数**

```php
$horizontalAlignment : string
	see self::HORIZONTAL_*
```

</td></tr>
</tbody>
</table>



### <span id="method_alignment_setVertical">setVertical()</span>

<table>
<thead><tr align="left"><th>
setVertical
</th></tr></thead>
<tbody>
<tr><td>
设置垂直对齐方式

```php
public setVertical(string $verticalAlignment) : $this
```

**参数**

```php
$verticalAlignment : string
	see self::VERTICAL_*
```

</td></tr>
</tbody>
</table>



### <span id="method_alignment_setWrapText">setWrapText()</span>

<table>
<thead><tr align="left"><th>
setWrapText
</th></tr></thead>
<tbody>
<tr><td>
设置宽度

```php
public setWrapText(bool $wrapped) : $this
```

</td></tr>
</tbody>
</table>


```php
// 设置 A 列垂直居中，自动换行
$worksheet->getStyle('A')->getAlignment()->setVertical(Alignment::VERTICAL_CENTER)->setWrapText(true);
// 设置 A 列水平居中，自动换行
$worksheet->getStyle('A')->getAlignment()->setHorizontal(Alignment::HORIZONTAL_CENTER)->setWrapText(true);
```


## Font
|函数|描述|
|:---|:---|
|[setBold()](#method_font_setBold)|设置是否加粗|
|[setColor()](#method_font_setColor)|设置字体颜色|
|[setItalic()](#method_font_setItalic)|设置是否斜体|
|[setName()](#method_font_setName)|设置字体|
|[setSize()](#method_font_setSize)|设置字体大小|


### <span id="method_font_setBold">setBold()</span>

<table>
<thead><tr align="left"><th>
setBold
</th></tr></thead>
<tbody>
<tr><td>
设置是否粗体

```php
public setBold(bool $bold) : $this
```

</td></tr>
</tbody>
</table>

### <span id="method_font_setColor">setColor()</span>

<table>
<thead><tr align="left"><th>
setColor
</th></tr></thead>
<tbody>
<tr><td>
设置颜色

```php
public setColor(Color $color) : $this
```

这种方式设置颜色可能还是有些繁琐，可以使用下面这种方式替换:

```php
$sheet->getStyle('A10')->getFont()->getColor()->setRGB('0000FF');
```

</td></tr>
</tbody>
</table>

### <span id="method_font_setItalic">setItalic()</span>

<table>
<thead><tr align="left"><th>
setItalic
</th></tr></thead>
<tbody>
<tr><td>
设置斜体

```php
public setItalic(bool $italic) : $this
```

</td></tr>
</tbody>
</table>


### <span id="method_font_setName">setName()</span>

<table>
<thead><tr align="left"><th>
setName
</th></tr></thead>
<tbody>
<tr><td>
设置名称

```php
public setName(string $fontname) : $this
```

</td></tr>
</tbody>
</table>

### <span id="method_font_setSize">setSize()</span>

<table>
<thead><tr align="left"><th>
setSize
</th></tr></thead>
<tbody>
<tr><td>
设置大小

```php
public setSize(mixed $sizeInPoints[, bool $nullOk = false ]) : $this
```

</td></tr>
</tbody>
</table>

### 应用
```php
// 级联设置字体以及大小
$worksheet->getStyleByColumnAndRow(1, $row)->getFont()->setName('Arial')->setSize(10);
```

## Fill
|函数|描述|
|:---|:---|
|setFillType()|设置填充类型|
|getStartColor()|获得起始颜色|

#### 应用
```php
// 填充背景色
$worksheet->getStyleByColumnAndRow($col, $row)->getFill()->setFillType(Fill::FILL_SOLID)->getStartColor()->setRGB('D6E9AD');
```

## DataValidation
|函数|描述|应用|
|:---|:---|:---|
|setType()|设置数据格式||
|setShowErrorMessage()|设置是否显示错误信息||
|setErrorTitle()|设置错误提示框标题|
|setError()|设置错误信息|
|setErrorStyle()|设置错误样式|
|setPromptTitle()|设置提示标题。需要与 setPrompt 搭配使用，否则无效果。|
|setPrompt()|设置提示文字|
|setAllowBlank()|设置是否允许为空|
|setShowInputMessage()|设置是否展示输入信息|
|setShowDropDown()|设置是否展示下拉|
|setFormula1()|设置公式|[查看](#setFormula1)|


**应用**

设置下拉菜单

```php
$worksheet->getDataValidation('E3')
	->setType(DataValidation::TYPE_LIST)
	->setShowDropDown(true)
	->setFormula1('"小学,初中,高中"');
```

> 如果不设置 type 或者 showDropDown 则不会生效

`setFormula1` 一般有这样几种用法：

1. 序列设置。这种适用于选项比较少的情况，因为 Excel 的序列长度是有限的。
```php
$data_validation->setFormula1('"选项1,选项2"');
```
2. 单元格引用。使用广泛，可以支持更多选项。
```php
$data_validation->setFormula1('教师列表!$A$1:$A' . sizeof($teacher_title_list)); // 教师列表是 sheet 的名字
```

## 下载 Excel
|后缀名|MIME|
|:---|:---|
|xlsx|application/vnd.openxmlformats-officedocument.spreadsheetml.sheet|
|xls|application/vnd.ms-excel|

```php
$writer = new Xlsx($spreadsheet);
$filename = rawurlencode("文件名称");
header('Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet');
header("Content-Disposition: attachment;filename=\"{$filename}\";filename*=UTF-8''{$filename}");
header('Cache-Control: max-age=0');
$writer->save('php://output');
```

