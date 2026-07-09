# 1. HTML 上下标实体无法正确转化  
  
### 问题  
OA 数据库中的 HTML 包含 `&sup1;`/`&sup2;`/`&sup3;` 等 HTML5 命名实体。老版 Jsoup (1.5.x) 不认识这些实体，解码失败产生乱码（如 `&sup3;` → `⊃3;`）。  
  
关键点：只要 Jsoup 将 `&sup3;` 正确解码为 Unicode 上标字符 `³`，Word 就会将其当作普通字符直接显示，视觉效果就是正常的上标，**不需要额外转换**。问题纯粹是老版 Jsoup 解码失败。  
  
### 解决方案  
- 使用 **Maven Shade Plugin** 将 Jsoup 1.20.1 **重定位包名**：`org.jsoup` → `torch.shaded.jsoup`  
- 新旧 jar 共存：OA 系统内置功能继续使用 `web/WEB-INF/lib/jsoup.jar`（旧版），不受任何影响；业务代码使用 `jsoup-shaded-1.20.1.jar`，包名不同，互不干扰  
- 修改 `HtmlTableConverter` 的 import 为 `torch.shaded.jsoup.*`  
  
### 涉及文件  
| 文件 | 变更 |  
|------|------|  
| `web/WEB-INF/lib/jsoup-shaded-1.20.1.jar` | 新增 shade jar |  
| `HtmlTableConverter.java` | import 改为 `torch.shaded.jsoup.*` |  
  
---  
  
## 2. 表格列宽无法在 Word 中均匀分配  
  
### 问题  
MARK 单元格原本跨整行（gridSpan=15），需按 HTML 列数拆分为 N 列。直接操作外层表网格会破坏其他行的格式。  
  
### 尝试过的方案（均失败）  
| 方案                       | 问题                                                 |
| ------------------------ | -------------------------------------------------- |
| 均分 gridSpan + 均分 tcW     | `getTcW().setW()` 在克隆的 CTTcPr 上不生效（POI 低层 API bug） |
| LCM 公倍数重建网格              | 改变了旧行的绝对宽度，破坏了样式                                   |
| `cell.setWidth()` 高层 API | 不改网格列宽，`tcW` 无法覆盖不等宽的网格列                           |
  
### 最终方案：MARK 格内嵌独立表格  
在 MARK 单元格内用 POI 创建一个**全新的独立 Word 表格**，与外层表完全隔离：  
  
- **每列等宽**：`cell.setWidth(totalW / N)` + `setWidthType(TableWidthType.DXA)`  
- **视觉一致**：内部边框设为 1 磅（`setInsideHBorder` / `setInsideVBorder`），四周边框设为无（`setTopBorder(NONE)` 等），使其与模板原有表格在视觉上无缝融合  
- **垂直居中**：`cell.setVerticalAlignment(XWPFTableCell.XWPFVertAlign.CENTER)`  
  
```java  
// MARK 格内嵌独立等宽表格  
XWPFTable inner = new XWPFTable(
	markCell.getCTTc().addNewTbl(), markCell, M, N);
inner.setWidth(String.valueOf(totalW));  
inner.setWidthType(TableWidthType.DXA);  
// 内部 1 磅，四周无边框  
inner.setInsideHBorder(XWPFTable.XWPFBorderType.SINGLE, 8, 0, "000000");  
inner.setInsideVBorder(XWPFTable.XWPFBorderType.SINGLE, 8, 0, "000000");  
inner.setTopBorder(XWPFTable.XWPFBorderType.NONE, 0, 0, "000000");  
inner.setBottomBorder(XWPFTable.XWPFBorderType.NONE, 0, 0, "000000");  
inner.setLeftBorder(XWPFTable.XWPFBorderType.NONE, 0, 0, "000000");  
inner.setRightBorder(XWPFTable.XWPFBorderType.NONE, 0, 0, "000000");  
  
long cellW = totalW / N;  
for (int r = 0; r < M; r++) {
	for (int c = 0; c < N; c++) {
		XWPFTableCell cell = inner.getRow(r).getCell(c);
		cell.setWidth(String.valueOf(cellW));
		cell.setWidthType(TableWidthType.DXA);
		cell.setVerticalAlignment(XWPFTableCell.XWPFVertAlign.CENTER);
		setCellText(cell, ...);
	}
}  
```  
  
### 涉及文件  
| 文件                        | 变更                           |
| ------------------------- | ---------------------------- |
| `MarkTableProcessor.java` | 由"操作外层表格行列"改为"MARK 格内创建嵌套表格" |
  
---  
  
## 3. HTML 表格渲染行数错乱  
  
### 问题  
部分复杂 HTML 表格解析后行列数不对，数据错位。  
  
### 根因及修复  
  
| 问题                           | 根因                          | 修复                                                             |
| ---------------------------- | --------------------------- | -------------------------------------------------------------- |
| 嵌套表格的行被选为外层行                 | `table.select("tr")` 穿透嵌套表  | 改为 `table.select("> tbody > tr, > tr")`                        |
| 嵌套表格的格被当成列                   | `tr.select("td, th")` 穿透嵌套表 | 改为 `tr.select("> td, > th")`                                   |
| colspan/rowspan 导致列数不一致      | 未处理合并单元格                    | 两遍扫描：colspan 展开 + rowspan 追踪填充                                 |
| `&nbsp;` 单元格变成脏数据            | Java `trim()` 不处理 ` `       | `cellText` 中先 `trim()` 再 `replace(' ', ' ')`                   |
| `<p>` 多段落变成一行                | `td.text()` 拼成一行            | `collectText` 递归，`<p>` 前后插 `\n`，`<br>` 无条件插 `\n`               |
| 块级元素间空白产生多余换行                | HTML 源码的空白文本节点被收集           | `t.trim().isEmpty()` 跳过纯空白文本节点                                 |
| `colspan` + `rowspan` 同表时被截断 | 第一遍只算 colspan，忽略 rowspan 位移 | 第二遍动态扩展 `maxCols`                                              |
| 表格边框和垂直居中                    | 格子默认无样式                     | 表级 `setInsideHBorder(1pt)` + 格级 `setVerticalAlignment(CENTER)` |
  
### 涉及文件  
| 文件                        | 变更                          |
| ------------------------- | --------------------------- |
| `HtmlTableConverter.java` | 选择器限定直选、两遍扫描处理合并、递归提取文本保留换行 |
  
---  
  
## 最终文件清单  
  
| 文件                                        | 作用                                   |
| ----------------------------------------- | ------------------------------------ |
| `web/WEB-INF/lib/jsoup-shaded-1.20.1.jar` | Shade 后 Jsoup 1.20.1，支持 HTML5 实体     |
| `HtmlTableConverter.java`                 | HTML→二维数组，处理 colspan/rowspan/嵌套表/多段落 |
| `MarkTableProcessor.java`                 | MARK 格内嵌独立等宽 Word 表格                 |
## 经验和教训

[[../daily/2026-07-09|2026-07-09]]