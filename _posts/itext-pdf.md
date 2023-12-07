---
title: iText处理pdf书签和标注代码乱记 
date: 2016-12-23 22:59:09 
categories: Java
tags: [工具, 代码, Java]
---

将文档1中的书签转存到文档2﻿​：

<!--more-->

```java
PdfReader reader1 = new PdfReader("in1.pdf");
List<HashMap<String, Object>> bookmarks = SimpleBookmark.getBookmark(reader1);
 
PdfReader reader2 = new PdfReader("in2.pdf");
PdfStamper stamper = new PdfStamper(reader2, new FileOutputStream("out.pdf"));
stamper.getWriter().setOutlines(bookmarks);
stamper.close();
```

导出书签文件﻿​：

```java
PdfReader reader = new PdfReader("in.pdf");
List<HashMap<String, Object>> bookmarks = SimpleBookmark.getBookmark(reader);
SimpleBookmark.exportToXML(bookmarks, new FileWriter("out.xml"), "utf-8", false);

导入书签到文件﻿​：
PdfReader reader = new PdfReader("in.pdf");
List<HashMap<String, Object>> bookmarks = SimpleBookmark.importFromXML(new FileReader("out.xml"));
PdfStamper stamper = new PdfStamper(reader, new FileOutputStream("out.pdf"));
stamper.getWriter().setOutlines(bookmarks);
stamper.close();
```

将文档1中的标注（包括超链接）转存到文档2﻿​：

```java
PdfReader reader1 = new PdfReader("in1.pdf");
PdfReader reader2 = new PdfReader("in2.pdf");
Document document = new Document(reader2.getPageSize(3));
PdfWriter writer = PdfWriter.getInstance(document, new FileOutputStream("out.pdf"));
document.open();
 
List<HashMap<String, Object>> bookmarks = SimpleBookmark.getBookmark(reader1);
writer.setOutlines(bookmarks);
 
int num = reader1.getNumberOfPages();
for (int i = 1; i <= num; i++) {
    PdfImportedPage page = writer.getImportedPage(reader2, i);
    PdfContentByte content = writer.getDirectContent();
    content.addTemplate(page, 0, 0);
 
    PdfDictionary pageDict = reader1.getPageN(i);
    PdfArray annotArray = pageDict.getAsArray(PdfName.ANNOTS);
    for (int j = 0; annotArray != null && j < annotArray.size(); ++j) {
        PdfDictionary curAnnot = annotArray.getAsDict(j);
        PdfAnnotation pdfAnnot = new PdfAnnotation(writer, null);
        pdfAnnot.putAll(curAnnot);
        writer.addAnnotation(pdfAnnot);
    }
    document.newPage();
}
 
document.close();
reader1.close();
reader2.close();
```

如果仅仅读取全部链接，只需将上述代码18行开始的for循环修改为：

```java
for (int j = 0; annotArray != null && j < annotArray.size(); ++j) {
    PdfDictionary curAnnot = annotArray.getAsDict(j);
    if (PdfName.LINK.equals(curAnnot.getAsName(PdfName.SUBTYPE))) {
        PdfAnnotation pdfAnnot = new PdfAnnotation(writer, null);
        pdfAnnot.putAll(curAnnot);
        writer.addAnnotation(pdfAnnot);
    }
}
```
 
**参考资料：**

1. iText 5 examples - iText 5 actions and annotations <http://developers.itextpdf.com/content/itext-5-examples>
2. iText in Action, 2nd 2010, 7. Making documents interactive <https://www.manning.com/books/itext-in-action-second-edition>

