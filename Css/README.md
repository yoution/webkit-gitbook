# Css
css从解析到生成renderStyle

## css 结构
例子:
```html
html#html-id  body[body-attr=body-attr-value].body-class > div:first-child + div::after, html.html-class body{
  color: #000;
  border:1px solid #000;
}
```

`styleSheet`由一条条的`cssRule`构成；  
`cssRule`由`selectorList`和`properties`构成；  
`selectorList`由","分隔的`selector`构成，例子中为","分隔的2个selector，每个selector又是由许多selector构成，比如html.html-class body，就是由3个selector构成的selecotor，多个selector又可构成compoundSelector，比如body[body-attr=body-attr-value].body-class就是由3个selector构成的compoundSelector，分别为body，[body-attr=body-attr-value]，.body-class;   
`properties`为selector后"{","}"内的内容，每个property由";"分隔，property又是由propertyId和propertyValue构成，propertyId和propertyValue分别为最简单的值，比如padding，会被分解成4个perpertyId，分别为paddingTop，paddingRight，paddingBottom，paddingLeft;

### selector基本类型
selector按照前缀分为5种基本类型:   
* "#": id    
* ".": class   
* "[": attribute   
* ":": pseudo (PseudoClass | PseudoElement)   
* 默认为tag

### selecotor之间的关系
* " ": DescendantSpace
* "+": DirectAdjacent
* ">": Child
* "~": IndirectAdjacent
* compoundSelector内部的selector: Subselector

## css 解析过程
css的解析过程按照文本顺序遍历token，每条cssRule由`prelude`和`blcok`构成，先截取prelude和block，再解析prelude和block;   
* 截取:   
prelude:取从开始到"{"前;   
block: 取"{"开始到对应的"}"，"{"，"}"需要成对匹配，如果没有匹配的"}"，则一直截取到styleSheet最后，相当于后面的cssRule全部废弃;   
* 解析:   
prelude: 截取后解析selectorList，如果selectorList里有一个selector解析错误，则整条cssRule丢弃，解析下一条cssRule，比如语法错误，出现"}"；  
生成selector过程，compoundSelector内部的selector从左向右，selector之间是从右向左,对于pseudoClass包含"()"的复杂类，会对"()"内的selector生成selectorList,对于例子中的第二个selector,结果为:
```json
{
  selector,   //body
  tagHitory: {
    selector,  //html
    tagHistory: {
      selector,   //.html-class
      tagHistory: null
    }
  }
}
```
block: 截取后解析properties，生成对应的propertyId和propertyValue，如果properties为空，也不会丢弃整条cssRule,
```json
properties:{
  propertyId: propertyValue;
  propertyId: propertyValue;
  ......
}
```
解析block过程会有解析失败问题，   
如propertyId和propertyValue为无效值，则丢弃该property，解析下一条property;   
如block内多了"{","}"对，移除"{","}"对内的内容;

## 生成ruleSet
ruleSet为内置的数据容器，按照cssRule最右边的selector的类型放入对应的队列中,
ruleSet:
* idRules
* classRules
* linkPseudoClassRules
* focusPseudoClassRules
* tagRules
* universalRules   

如例子中的cssRule会按照Tag生成对应的结构
```
{
  body:[...,cssRule],
  div:[...,cssRule],
}  
```

## 查找与匹配cssRule
查找css按照dom树的结构遍历，从分别从匹配表的ruleSet里获取cssRules，进行对比匹配。
textNode,display:none的node不需要匹配cssRule，已经匹配过且不受前面node影响的node不需要匹配cssRule；   
匹配表顺序
* UARules   
  CSSDefaultStyleSheets::defaultStyle   
  CSSDefaultStyleSheets::defaultQuirksStyle
* UserRules
* AuthorRules

3个表遍历结束后按照specificity排序后，解析inlineStyle，插到AuthorRules里的最后。

比如对于body元素，在依次从每个表的ruleSet里，依次到idRules，classRules，linkPseudoClassRules，focusPseudoClassRules, tagRules, universalRules里获取cssRules，比如获取tag时，会获取全部body的cssRules，如"#html-id body{}, xx html body{}"，后期会匹配，排除不合适的cssRule。

匹配cssRule时，如果selectorList里selector大于1， 按照selectorList上倒数4个(最多4个)selector上id, class, tag值的内容*117, *19, *13得到hash，与当前dom对比；
在计算specificity时会进行最终对比，compoundSelector会生成对应的selectorFragment。

### specificity权重计算
* id : 0x10000
* class|pseudoClass|attr: 0x100
* tag|pseudoElement: 0x1

特俗处理pseudoClass:   
* :match :0，本身的:match不计算，但是会计算"()"，内的selectorList的权重
* :not :0, 本身的:not不计算，但是取的是"()"内selectorList中每个compoundSelector的最大值

下面的3个pseudoClass在上面的计算后，内部的selectorList的还会再计算一次
* :nth-child()和:nth-last-child() 包含表达式时(1 of .foo)时会计算，普通的用法(2n+1),(even)不会
* :matches 

例子:
* `html body:any(#body-id)`: 0x1(body) + 0x100(:any) + 0x1(html) = 0x102 = 258
* `html body:match(#body-id)`: 0x1(body) + 0x0(:match) + 0x10000(#body-id) + 0x1(html) = 0x10002 = 65538
* `html body:nth-child(1 of .body-class)`: 0x1(body) + 0x100(:nth-child) + 0x100(.body-class) + 0x1(html) = 0x202 = 514
* `html body:matches(#body-id)`: 0x1(body) + 0x0(:match) + 0x10000(#body-id) + 0x1(html) = 0x10002 = 65538
* `html body:not(.class, #id)`: 0x1(body) + 0x0(:not) + max(0x100(.class),0x10000(.id)) + 0x1(html) = 0x10002 = 65538

## 设置cascade
cascade内置440个propertity，按照匹配表顺序从每个匹配表的匹配的cssRule里取出propertyValue，设置到cascade的property上。
* UARule 非important
* UserRule 非important
* AuthorRule 非important
* AuthorRule important
* UserRule important
* UARule important

##把cascade上的样式设置到RenderStyle
440个样式的设置顺序 
* 1
* 418
* 2-28
* 29-440

