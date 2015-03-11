title: 简单谈谈dom解析xml和html
date: 2014-09-21 09:49:50
tags: [Dom,JavaScript,Java]
description: 记录学习dom规范及java和javascript中的实现

-----------
## 前言 ##

[文件对象模型（Document Object Model，简称DOM）](http://www.w3.org/TR/2004/REC-DOM-Level-3-Core-20040407/)，是W3C组织推荐的处理可扩展标志语言的标准编程接口。html，xml都是基于这个模型构造的。这也是一个W3C推出的标准。java，python，javascript等语言都提供了一套基于dom的编程接口。

## java使用dom解析xml ##
一段xml文档, note.xml：

	<?xml version="1.0" encoding="UTF-8"?>
	<note>
	    <to id="1">George</to>
	    <from>John</from>
	    <heading>Reminder</heading>
	    <body>Don't forget the meeting!</body>
	</note>

我们先使用w3c dom解析该xml：

    @Test
    public void test() {
        NodeList nodeList = doc.getChildNodes().item(0).getChildNodes();
        System.out.println("xml size: " + nodeList.getLength());
        for(int i = 0; i < nodeList.getLength(); i ++) {
            Node node = nodeList.item(i);
            System.out.println(node.getNodeType());
            System.out.println(node.getNodeName());
        }
    }

输出：

    xml size: 9
    3
    #text
    1
    to
    3
    #text
    1
    from
    3
    #text
    1
    heading
    3
    #text
    1
    body
    3
    #text

我们看到代码输出note节点的字节点的时候，有9个节点，但是xml文档中note节点实际上只有to、from、heading、body4个节点。 那为什么是9个呢，原因是这样的。
选取几个w3c规范中关于节点类型的描述：

| 节点类型 | 描述 | nodeName返回值 | nodeValue返回值 | 子元素 | 类型常量值 |
| ------------- |:-------------:| -----:| -----:| -----:| -----:|
| Document      | 表示整个文档（DOM 树的根节点） | #document |null|Element(max. one)，Comment，DocumentType |9
| Element      | 表示 element（元素）元素      |   element name |null|Text，Comment，CDATASection |1
| Attr | 表示属性      |    属性名称 |属性值|Text |2
| Text | 表示元素或属性中的文本内容。      |    #text |节点内容|None |3
| CDATASection | 表示文档中的 CDATA 区段（文本不会被解析器解析）      |    #cdata-section |节点内容|None |4
| Comment | 表示注释      |    #comment |注释文本|None |8

更多细节请查看[w3c DOM节点类型](http://www.w3school.com.cn/xmldom/dom_nodetype.asp)

下面解释一下文档节点的字节点的处理过程：
![](http://format-blog-image.qiniudn.com/dom_parse_xml1.png)

其中红色部分为Text节点，紫色部分是Element节点(只画了部分)。&lt;/body&gt;后面的也是一个Element节点，所有4个Element节点，5个Text节点。

所以输出的内容中3 #text表示该节点是个Text节点，1 节点name是个Element节点，这与表格中表述的是一样的。

测试代码：

    @Test
    public void test1() {
        NodeList nodeList = doc.getChildNodes().item(0).getChildNodes();
        System.out.println("xml size: " + nodeList.getLength());
        for(int i = 0; i < nodeList.getLength(); i ++) {
            Node node = nodeList.item(i);
            if(node.getNodeType() == Node.TEXT_NODE) {
                System.out.println(node.getNodeValue().replace("\n","hr").replace(' ', '-'));
            }
        }
    }

![](http://format-blog-image.qiniudn.com/dom_parse_xml2.png)
很明显，我们把空格和回车键替换打印后发现我们的结论是正确的。

测试代码：

    @Test
    public void test2() {
        System.out.println("doc type: " + doc.getNodeType());
        NodeList nodeList = doc.getChildNodes().item(0).getChildNodes();
        Node secondNode = nodeList.item(1);
        System.out.println("element [to] node type: " + secondNode.getNodeType());
        System.out.println("element [to] node name: " + secondNode.getNodeName());
        System.out.println("element [to] node value: " + secondNode.getNodeValue());
        System.out.println("element [to] children len: " + secondNode.getChildNodes().getLength());
        System.out.println("element [to] children node type: " + secondNode.getChildNodes().item(0).getNodeType());
        System.out.println("element [to] children node value: " + secondNode.getChildNodes().item(0).getNodeValue());
        System.out.println("element [to] children node name: " + secondNode.getChildNodes().item(0).getNodeName());
        Node attNode = secondNode.getAttributes().item(0);
        System.out.println("attr type: " + attNode.getNodeType());
    }

![](http://format-blog-image.qiniudn.com/dom_parse_xml3.png)
输出结果跟表格中是一样的。

大家有兴趣的话其他类型的节点比如CDATA节点大家可以自行测试～

## javascript使用dom解析html ##

html代码：

    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="utf-8">
      <title>JS Bin</title>
    </head>
    <body>
      <div>
        <p>gogogo</p>
      </div>
    </body>
    </html>

js代码：

    console.log(document.nodeType);
    var div = document.getElementsByTagName("div")[0]; //9
    console.log(div.nodeType); //1
    for(var i = 0;i < div.childNodes.length; i ++) {
      console.log(div.childNodes[i].nodeType);
    }

分别输出9，1，3，1，3
跟我们在表格中对应～

## 总结 ##
本次博客主要讲解了dom解析xml和html。 以前使用java解析xml的时候总是使用一些第三方库，比如jdom。 但是dom却是w3c的规范，不止java，包括javascript，python这些主流语言也都主持，有了规范，语言只是实现了这些规范而已。
