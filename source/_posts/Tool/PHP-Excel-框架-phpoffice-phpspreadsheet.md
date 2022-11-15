---
title: PHP Excel 框架 phpoffice/phpspreadsheet
date: 2022-11-15 13:16:22
tags:
- Excel
---

# phpoffice/phpspreadsheet

[https://github.com/PHPOffice/PhpSpreadsheet](https://github.com/PHPOffice/PhpSpreadsheet)

# 1. composer 依赖

通过 composer 导入相关依赖
```bash
composer require phpoffice/phpspreadsheet
```

```json
"require": {
  "phpoffice/phpspreadsheet": "^1.6"
}
```


# 2. 读取 Excel 步骤
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

# 3. 相关类
## Spreadsheet
|函数|描述|应用|
|:---|:---|:---|
|getActiveSheet()|获得活跃的工作簿|
|createSheet()|创建新工作簿|

## Worksheet

### 方法
|函数|描述|应用|
|:---|:---|:---|
|setTitle()|设置工作簿的标题|
|getHighestRow()|获得最大行数|
|setCellValue()|以坐标方式设置单元格的值。|
|setCellValueByColumnAndRow()|以列号、行号方式设置单元格值|
|getCell()|以坐标方式获取单元格的值|
|getCellByColumnAndRow()|以列号、行号方式获取单元格的值|
|getRowDimension()|以行号方式获得行维度|
|getColumnDimension()|以列名方式获得列维度。如：'A'|
|getStyle()|以坐标方式获得样式|[查看](#getStyle)|
|getDataValidation()|以坐标方式获得数据校验|[查看](#getDataValidation)|
|mergeCells()|合并单元格，参数示例：`A1:E1`|
|mergeCellsByColumnAndRow|合并单元格，需要传入 4 个参数表示两个单元格|

### 应用

```php
// 如果要获取整列单元格的样式，则仅仅传入列名
// 注意：对于列名是大写字母，而不是索引数字
$style = $worksheet->getStyle('C');
// 如果要配置整列的单元格，则这样传入参数
$worksheet->getDataValidation('$A:$A');
```


## RowDimension
### 方法
|函数|描述|应用|
|:---|:---|:---|
|setRowHeight()|设置行高|

> 注意，设置行高并不是绝对准确的，excel 会适当缩放

### 应用
```php
// 设置行高
// 行高这里传入索引数字，从 1 开始
$worksheet->getRowDimension($row)->setRowHeight(13.2);
```

## ColumnDimension 

|函数|描述|应用|
|:---|:---|:---|
|setWidth()|设置宽度|

```php
// 设置列宽
$worksheet->getColumnDimensionByColumn(1)->setWidth(8);
$worksheet->getColumnDimension('A')->setWidth(8);
```

> 注意，设置列宽并不是绝对准确的，excel 会适当缩放

## Style
|函数|描述|应用|
|:---|:---|:---|
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
#### Alignment
|函数|描述|应用|
|:---|:---|:---|
|setWrapText()|设置对齐方式||
|setHorizontal()|设置自动换行|

```php
// 设置 A 列垂直居中，自动换行
$worksheet->getStyle('A')->getAlignment()->setVertical(Alignment::VERTICAL_CENTER)->setWrapText(true);
// 设置 A 列水平居中，自动换行
$worksheet->getStyle('A')->getAlignment()->setHorizontal(Alignment::HORIZONTAL_CENTER)->setWrapText(true);
```


### Font
|函数|描述|应用|
|:---|:---|:---|
|setName()|设置字体||
|setSize()|设置字体大小|

#### 应用
```php
// 级联设置字体以及大小
$worksheet->getStyleByColumnAndRow(1, $row)->getFont()->setName('Arial')->setSize(10);
```

### Fill
|函数|描述|
|:---|:---|
|setFillType()|设置填充类型|
|getStartColor()|获得起始颜色|

#### 应用
```php
// 填充背景色
$worksheet->getStyleByColumnAndRow($col, $row)->getFill()->setFillType(Fill::FILL_SOLID)->getStartColor()->setRGB('D6E9AD');
```

### Color


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

