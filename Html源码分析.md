Html源码分析
==

> 代码版本：**Android API 23**



## 1.简介

Html能够通过Html标签来为文字设置样式，让TextView显示富文本信息，其只支持部分标签不是全部，具体支持哪些标签将分析中揭晓。

## 2.使用方法
申明一段带有Html标签的字符串，然后调用`Html.fromHtml`方法就能够根据标签设置对应的样式。

```java
	String htmlString =
        "<font color='#ff0000'>颜色</font><br/>" +
        "<a href='http://www.baidu.com'>链接</a><>br/>" +
        "<big>大字体</big><br/>"+
        "<small>小字体</small><br/>"+
        "<b>加粗</b><br/>"+
        "<i>斜体</i><br/>" +
        "<h1>标题一</h1>" +
        "<h2>标题二</h2>" +
        "<h3>标题三</h3>" +
        "<h4>标题四</h4>" +
        "<img src='ic_launcher'/>" +
        "<blockquote>引用</blockquote>" +
        "<div>块</div>" +
        "<u>下划线</u><br/>" +
        "<sup>上标</sup>正常字体<sub>下标</sub><br/>" +
        "<u><b><font color='@holo_blue_light'><sup><sup>组</sup>合</sup><big>样式</big><sub>字<sub>体</sub></sub></font></b></u>";
    tv.setText(Html.fromHtml(htmlString));
```
运行效果:

![Html](https://github.com/DennyCai/AndroidSdkSourceAnalysis/blob/master/img/show.jpg?raw=true)

在Demo中发现使用`<img>`标签都显示小方块,解决这个问题的办法是调用`Html.fromHtml(String,Html.ImageGetter,Html.TagHandler)`的重载方法，并传入自定义的`Html.ImageGetter`对象，重写`getDrawable`方法，代码如下：

```java
	Html.ImageGetter getter = new Html.ImageGetter() {
        @Override
        public Drawable getDrawable(String source) {
            int id = getResources().getIdentifier(source,"mipmap",getPackageName());
            Drawable drawable = getResources().getDrawable(id);
            //必须设置手动设置drawable大小，否则无法显示
            drawable.setBounds(0,0,drawable.getIntrinsicWidth(),drawable.getIntrinsicHeight());
            return drawable;
        }
    };
    textView.setText( Html.fromHtml(htmlString,getter,null));
```
运行效果:

![Html](https://github.com/DennyCai/AndroidSdkSourceAnalysis/blob/master/img/showimg.png?raw=true)

原生支持的Html标签优先，为了方便扩展，我们可以通过自定义`Html.TagHandler`来支持自定义标签显示效果，代码如下：

```java
	Html.TagHandler tagHandler = new Html.TagHandler() {
        int start = -1;
        int end = -1;
        @Override
        public void handleTag(boolean opening, String tag, Editable output, XMLReader xmlReader) {
            if(opening){
                if(tag.equals("custom")) {
                    start = output.length();
                }
            }else{
                if(tag.equals("custom")&&start!=-1){
                    end = output.length();
                    BulletSpan bullet = new BulletSpan(10);
                    output.setSpan(bullet,start,end,Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
                }
            }
        }
    };
    textView.setText( Html.fromHtml("<custom>自定义标签</custom>",null,tagHandler));
```

运行效果：

![TagHandler](https://github.com/DennyCai/AndroidSdkSourceAnalysis/blob/master/img/custmtag.png?raw=true)

使用`Html.toHtml`方法能够将带有样式效果的Spanned文本对象生成对应的Html格式，标签内的字符会被转译成，下面为WebView显示效果，部分效果与上面TextView显示的效果有差异，代码如下：

```java
	webView.loadData(Html.toHtml(Html.fromHtml(htmlString)),"text/html", "utf-8");
```

运行效果：

![toHtml](https://github.com/DennyCai/AndroidSdkSourceAnalysis/blob/master/img/tohtml.png?raw=true)

返回字符串：

```text
<p dir="ltr"><font color ="#ff0000">&#39068;&#33394;</font><br>
<a href="http://www.baidu.com">&#38142;&#25509;</a><br>
&#22823;&#23383;&#20307;<br>
&#23567;&#23383;&#20307;<br>
<b>&#21152;&#31895;</b><br>
<i>&#26012;&#20307;</i></p>
<p dir="ltr"><b>&#26631;&#39064;&#19968;</b></p>
<p dir="ltr"><b>&#26631;&#39064;&#20108;</b></p>
<p dir="ltr"><b>&#26631;&#39064;&#19977;</b></p>
<p dir="ltr"><b>&#26631;&#39064;&#22235;</b></p>
<p dir="ltr"><img src="ic_launcher"></p>
<blockquote><p dir="ltr">&#24341;&#29992;<br>
</p>
</blockquote>
<p dir="ltr"><br>
&#22359;</p>
<p dir="ltr"><u>&#19979;&#21010;&#32447;</u><br>
<sup>&#19978;&#26631;</sup>&#27491;&#24120;&#23383;&#20307;<sub>&#19979;&#26631;</sub><br>
<sup><sup><b><u>&#32452;</u></b></sup></sup><sup><b><u>&#21512;</u></b></sup><b><u>&#26679;&#24335;</u></b><sub><b><u>&#23383;</u></b></sub><sub><sub><b><u>&#20307;</u></b></sub></sub></p>
```

`Html.escapeHtml`方法则是将Html标签进行转译,例如`<p dir="ltr">`变转译成`&lt;p dir="ltr"&gt;`。

## 3、原理分析

### 3.1、初探Html类

```java
/**
 * 该类将HTML处理成带样式的文本，但不支持所有的HTML标签
 */
public class Html {
    
    /**
     * 为<img>标签提供图片检索功能
     */
    public static interface ImageGetter {
        /**
         * 当HTML解析器解析到<img>标签时，source参数为标签中的src的属性值，
         * 返回值必须为Drawable;如果返回null则会使用小方块来显示，如前面所见，
         * 并需要调用Drawable.setBounds()方法来设置大小，否则无法显示图片。
         * @param source:
         */
        public Drawable getDrawable(String source);
    }

    /**
     * HTML标签解析扩展接口
     */
    public static interface TagHandler {
        /**
         * 当解析器解析到本身不支持或用户自定义的标签时，该方法会被调用
         * @param opening:标签是否打开
         * @param tag:标签名
         * @param output:截止到当前标签，解析到的文本内容
         * @param xmlReader:解析器对象
         */
        public void handleTag(boolean opening, String tag,
                                 Editable output, XMLReader xmlReader);
    }

    private Html() { }

    /**
     * 返回样式文本，所有<img>标签都会显示为一个小方块
     * 使用TagSoup库处理HTML
     * @param source:带有html标签字符串
     */
    public static Spanned fromHtml(String source) {
        return fromHtml(source, null, null);
    }

    /**
     * 可传入ImageGetter来获取图片源，TagHandler添加支持其他标签
     */
    public static Spanned fromHtml(String source, ImageGetter imageGetter,
                                   TagHandler tagHandler) {
        .....
    }

    /**
     * 将带样式文本反向解析成带Html的字符串，注意这个方法并不是还原成fromHtml接收的带Html标签文本
     */
    public static String toHtml(Spanned text) {
        StringBuilder out = new StringBuilder();
        withinHtml(out, text);
        return out.toString();
    }

    /**
     * 返回转译标签后的字符串
     */
    public static String escapeHtml(CharSequence text) {
        StringBuilder out = new StringBuilder();
        withinStyle(out, text, 0, text.length());
        return out.toString();
    }

    /**
     * 懒加载HTML解析器的Holder
     * a) zygote对其进行预加载
     * b) 直到需要的时候才加载
     */
    private static class HtmlParser {
        private static final HTMLSchema schema = new HTMLSchema();
    }

```

### 3.2、从fromHtml开始
Html类主要方法就`4`个，功能也简单，生成带样式的`fromHtml`方法最总都是调用重载3个参数的方法。
```java
public static Spanned fromHtml(String source, ImageGetter imageGetter,
                                   TagHandler tagHandler) {
    //初始化解析器
    Parser parser = new Parser();
    try {
        //配置解析Html模式
        parser.setProperty(Parser.schemaProperty, HtmlParser.schema);
    } catch (org.xml.sax.SAXNotRecognizedException e) {
        throw new RuntimeException(e);
    } catch (org.xml.sax.SAXNotSupportedException e) {
        throw new RuntimeException(e);
    }
    //初始化真正的解析器
    HtmlToSpannedConverter converter =
            new HtmlToSpannedConverter(source, imageGetter, tagHandler,parser);
    return converter.convert();
}
```

源代码中并没有包含Parser对象，根据`import org.ccil.cowan.tagsoup.Parser;`和`fromHtml`注释可知，HTML解析器是使用[Tagsoup](http://home.ccil.org/~cowan/XML/tagsoup/)库来解析HTML标签，为什么会选择该库，进行一番搜索得知[Tagsoup](http://home.ccil.org/~cowan/XML/tagsoup/)是兼容`SAX`的解析器，我们知道对XML常见的的解析方式还有`DOM`、Android系统中还使用`PULL`解析与`SAX`同样是基于事件驱动模型，之所有使用tagsoup是因为该库可以将HTML转化为XML，我们都知道HTML有时候并不像XML那样标签都需要闭合，例如`<br>`也是一个有效的标签，但是XML中则是不良格式。详情可见官方网站，但是好像没有开发文档，这里就不详细说明，只关注`SAX`解析过程。

### 3.3、HtmlToSpannedConverter原理

```java
class HtmlToSpannedConverter implements ContentHandler {

    private static final float[] HEADER_SIZES = {
        1.5f, 1.4f, 1.3f, 1.2f, 1.1f, 1f,
    };

    private String mSource;
    private XMLReader mReader;
    private SpannableStringBuilder mSpannableStringBuilder;
    private Html.ImageGetter mImageGetter;
    private Html.TagHandler mTagHandler;

    public HtmlToSpannedConverter(
            String source, Html.ImageGetter imageGetter, Html.TagHandler tagHandler,
            Parser parser) {
        mSource = source;//html文本
        mSpannableStringBuilder = new SpannableStringBuilder();//用于存放标签中的字符串
        mImageGetter = imageGetter;//图片加载器
        mTagHandler = tagHandler;//自定义标签器
        mReader = parser;//解析器
    }

    public Spanned convert() {
        //设置内容处理器
        mReader.setContentHandler(this);
        try {
            //开始解析
            mReader.parse(new InputSource(new StringReader(mSource)));
        } catch (IOException e) {
            // We are reading from a string. There should not be IO problems.
            throw new RuntimeException(e);
        } catch (SAXException e) {
            // TagSoup doesn't throw parse exceptions.
            throw new RuntimeException(e);
        }
        //省略
        ...
        ...
        return mSpannableStringBuilder;
    }
```
通过上面代码可以发现，`SpannableStringBuilder`是用来存放解析html标签中的字符串，类似`StringBuilder`，但它附带有样式的字符串。重点关注`convert`里面的`setContentHandler`方法，该方法接收的是`ContentHandler`接口，使用过`SAX`解析的读者应该不陌生，该接口定义了一系列`SAX`解析事件的方法。
```java
public interface ContentHandler
{
    //设置文档定位器
    public void setDocumentLocator (Locator locator);
    //文档开始解析事件
    public void startDocument ()
    throws SAXException;
    //文档结束解析事件
    public void endDocument()
    throws SAXException;
    //解析到命名空间前缀事件
    public void startPrefixMapping (String prefix, String uri)
    throws SAXException;
    //结束命名空间事件
    public void endPrefixMapping (String prefix)
    throws SAXException;
    //解析到标签事件
    public void startElement (String uri, String localName,
                  String qName, Attributes atts)
    throws SAXException;
    //标签结束事件
    public void endElement (String uri, String localName,
                String qName)
    throws SAXException;
    //标签中内容事件
    public void characters (char ch[], int start, int length)
    throws SAXException;
    //可忽略的空格事件
    public void ignorableWhitespace (char ch[], int start, int length)
    throws SAXException;
    //处理指令事件
    public void processingInstruction (String target, String data)
    throws SAXException;
    //忽略标签事件
    public void skippedEntity (String name)
    throws SAXException;
}
```
对应`HtmlToSpannedConverter`中的实现。
```java
public void setDocumentLocator(Locator locator) {}
public void startDocument() throws SAXException {}
public void endDocument() throws SAXException {}
public void startPrefixMapping(String prefix, String uri) throws SAXException {}
public void endPrefixMapping(String prefix) throws SAXException {}
public void startElement(String uri, String localName, String qName, Attributes attributes)
        throws SAXException {
    handleStartTag(localName, attributes);
}
public void endElement(String uri, String localName, String qName) throws SAXException {
    handleEndTag(localName);
}
public void characters(char ch[], int start, int length) throws SAXException {
    //忽略
    ...
}
public void ignorableWhitespace(char ch[], int start, int length) throws SAXException {}
public void processingInstruction(String target, String data) throws SAXException {}
public void skippedEntity(String name) throws SAXException {}
```

我们发现该类中只实现了`startElement`，`endElement`，`characters`这三个方法，所以只关心标签的类型和标签里的字符。然后调用`mReader.parse`方法，开始对HTML进行解析。解析的事件流如下：
`startElement` -> `characters` -> `endElement`
`startElemnt`里面调用的是`handleStartTag`方法，`endElement`则是调用`handleEndTag`方法。

```java
/**
 * @param tag：标签类型
 * @param attributes：属性值
 * 例如遇到<font color='#FFFFFF'>标签，tag="font",attributes={"color":"#FFFFFF"}
 */
private void handleStartTag(String tag, Attributes attributes) {
    if (tag.equalsIgnoreCase("br")) {
        // 我们不需要关心br标签是否有闭合，因为Tagsoup会帮我们处理
    } else if (tag.equalsIgnoreCase("p")) {
        handleP(mSpannableStringBuilder);
    } else if (tag.equalsIgnoreCase("div")) {
        handleP(mSpannableStringBuilder);
    } else if (tag.equalsIgnoreCase("strong")) {
        start(mSpannableStringBuilder, new Bold());
    } else if (tag.equalsIgnoreCase("b")) {
        start(mSpannableStringBuilder, new Bold());
    } else if (tag.equalsIgnoreCase("em")) {
        start(mSpannableStringBuilder, new Italic());
    } else if (tag.equalsIgnoreCase("cite")) {
        start(mSpannableStringBuilder, new Italic());
    } else if (tag.equalsIgnoreCase("dfn")) {
        start(mSpannableStringBuilder, new Italic());
    } else if (tag.equalsIgnoreCase("i")) {
        start(mSpannableStringBuilder, new Italic());
    } else if (tag.equalsIgnoreCase("big")) {
        start(mSpannableStringBuilder, new Big());
    } else if (tag.equalsIgnoreCase("small")) {
        start(mSpannableStringBuilder, new Small());
    } else if (tag.equalsIgnoreCase("font")) {
        startFont(mSpannableStringBuilder, attributes);
    } else if (tag.equalsIgnoreCase("blockquote")) {
        handleP(mSpannableStringBuilder);
        start(mSpannableStringBuilder, new Blockquote());
    } else if (tag.equalsIgnoreCase("tt")) {
        start(mSpannableStringBuilder, new Monospace());
    } else if (tag.equalsIgnoreCase("a")) {
        startA(mSpannableStringBuilder, attributes);
    } else if (tag.equalsIgnoreCase("u")) {
       start(mSpannableStringBuilder, new Underline());
    } else if (tag.equalsIgnoreCase("sup")) {
        start(mSpannableStringBuilder, new Super());
    } else if (tag.equalsIgnoreCase("sub")) {
        start(mSpannableStringBuilder, new Sub());
    } else if (tag.length() == 2 &&
               Character.toLowerCase(tag.charAt(0)) == 'h' &&
               tag.charAt(1) >= '1' && tag.charAt(1) <= '6') {
        handleP(mSpannableStringBuilder);
         start(mSpannableStringBuilder, new Header(tag.charAt(1) - '1'));
    } else if (tag.equalsIgnoreCase("img")) {
        startImg(mSpannableStringBuilder, attributes, mImageGetter);
    } else if (mTagHandler != null) {
        mTagHandler.handleTag(true, tag, mSpannableStringBuilder, mReader);
    }
}
//标签结束
private void handleEndTag(String tag) {
        if (tag.equalsIgnoreCase("br")) {
            handleBr(mSpannableStringBuilder);
        } else if (tag.equalsIgnoreCase("p")) {
            handleP(mSpannableStringBuilder);
        } else if (tag.equalsIgnoreCase("div")) {
            handleP(mSpannableStringBuilder);
        } else if (tag.equalsIgnoreCase("strong")) {
            end(mSpannableStringBuilder, Bold.class, new StyleSpan(Typeface.BOLD));
        } else if (tag.equalsIgnoreCase("b")) {
            end(mSpannableStringBuilder, Bold.class, new StyleSpan(Typeface.BOLD));
        } else if (tag.equalsIgnoreCase("em")) {
            end(mSpannableStringBuilder, Italic.class, new StyleSpan(Typeface.ITALIC));
        } else if (tag.equalsIgnoreCase("cite")) {
            end(mSpannableStringBuilder, Italic.class, new StyleSpan(Typeface.ITALIC));
        } else if (tag.equalsIgnoreCase("dfn")) {
            end(mSpannableStringBuilder, Italic.class, new StyleSpan(Typeface.ITALIC));
        } else if (tag.equalsIgnoreCase("i")) {
            end(mSpannableStringBuilder, Italic.class, new StyleSpan(Typeface.ITALIC));
        } else if (tag.equalsIgnoreCase("big")) {
            end(mSpannableStringBuilder, Big.class, new RelativeSizeSpan(1.25f));
        } else if (tag.equalsIgnoreCase("small")) {
            end(mSpannableStringBuilder, Small.class, new RelativeSizeSpan(0.8f));
        } else if (tag.equalsIgnoreCase("font")) {
            endFont(mSpannableStringBuilder);
        } else if (tag.equalsIgnoreCase("blockquote")) {
            handleP(mSpannableStringBuilder);
            end(mSpannableStringBuilder, Blockquote.class, new QuoteSpan());
        } else if (tag.equalsIgnoreCase("tt")) {
            end(mSpannableStringBuilder, Monospace.class,
                    new TypefaceSpan("monospace"));
        } else if (tag.equalsIgnoreCase("a")) {
            endA(mSpannableStringBuilder);
        } else if (tag.equalsIgnoreCase("u")) {
            end(mSpannableStringBuilder, Underline.class, new UnderlineSpan());
        } else if (tag.equalsIgnoreCase("sup")) {
            end(mSpannableStringBuilder, Super.class, new SuperscriptSpan());
        } else if (tag.equalsIgnoreCase("sub")) {
            end(mSpannableStringBuilder, Sub.class, new SubscriptSpan());
        } else if (tag.length() == 2 &&
                Character.toLowerCase(tag.charAt(0)) == 'h' &&
                tag.charAt(1) >= '1' && tag.charAt(1) <= '6') {
            handleP(mSpannableStringBuilder);
            endHeader(mSpannableStringBuilder);
        } else if (mTagHandler != null) {
            mTagHandler.handleTag(false, tag, mSpannableStringBuilder, mReader);
        }
    }
//标签内容 ch[]:存放字符数组 start:开始位置，lenght:长度
public void characters(char ch[], int start, int length) throws SAXException {
        StringBuilder sb = new StringBuilder();
        /*
         * 忽略开头的空格或连续两个空格
         * 换行符视为空格符
         */
        for (int i = 0; i < length; i++) {
            char c = ch[i + start];

            if (c == ' ' || c == '\n') {
                char pred;
                int len = sb.length();

                if (len == 0) {
                    len = mSpannableStringBuilder.length();

                    if (len == 0) {//开头为空格符
                        pred = '\n';
                    } else {//获取上一个字符
                        pred = mSpannableStringBuilder.charAt(len - 1);
                    }
                } else {//获取上一个字符
                    pred = sb.charAt(len - 1);
                }

                if (pred != ' ' && pred != '\n') {//判断是否为连续空格
                    sb.append(' ');
                }
            } else {//不是空格或不连续为空格则添加字符
                sb.append(c);
            }
        }

        mSpannableStringBuilder.append(sb);
    }    
```

从上面方法中我们可以总结出支持的HTML标签列表
* br
* p
* div
* strong
* b
* em
* cite
* dfn
* i
* big
* small
* font
* blockquote
* tt
* monospace
* a
* u
* sup
* sub
* h1-h6
* img

### 3、4 标签是如何处理的

#### 1. br标签

这里分析如何处理`<br>`标签，在`handleStartTag`方法中可以发现br标签直接被忽略了，在`handleEndTag`方法中才被真正处理。
```java
private void handleEndTag(String tag) {
    ...
    if (tag.equalsIgnoreCase("br")) {
        handleBr(mSpannableStringBuilder);
    }
    ...
}
//代码很简单，直接加换行符
private static void handleBr(SpannableStringBuilder text) {
    text.append("\n");
}

```

#### 2. p标签 

p标签为段落，其作用是给p标签中的文字前后换行，在`handleStartTag`和`handleEndTag`遇到p标签都是调用`handleP`方法，`characters`则添加p标签之间的字符串。

```java
private void handleStartTag(String tag, Attributes attributes) {
    ...
    else if (tag.equalsIgnoreCase("p")) {
        handleP(mSpannableStringBuilder);
    }
    ...
}

private void handleEndTag(String tag) {
    ...
    else if (tag.equalsIgnoreCase("p")) {
        handleP(mSpannableStringBuilder);
    }   
    ...
}

private static void handleP(SpannableStringBuilder text) {
    int len = text.length();

    if (len >= 1 && text.charAt(len - 1) == '\n') {
        if (len >= 2 && text.charAt(len - 2) == '\n') {
            //如果前面两个字符都为换行符，则忽略
            return;
        }
        //否则添加一个换行符
        text.append("\n");
        return;
    }
    //其他情况添加两个换行符
    if (len != 0) {
        text.append("\n\n");
    }
}
```

#### 3. strong标签

该标签作用是为加粗字体，在`handleStartTag`和`handleEndTag`分别调用`start`和`end`方法。

```java
private void handleStartTag(String tag, Attributes attributes) {
    ...
    else if (tag.equalsIgnoreCase("strong")) {
        start(mSpannableStringBuilder, new Bold());
    }
    ...
}

private static class Bold { }//什么都没有

private void handleEndTag(String tag) {
    ...
    else if (tag.equalsIgnoreCase("strong")) {
        end(mSpannableStringBuilder, Bold.class, new StyleSpan(Typeface.BOLD));
    }   
    ...
}

private static void start(SpannableStringBuilder text, Object mark) {
    int len = text.length();
    //mark作为类型标记并没有实际功能，指明开始的位置，
    //结束位置延迟到`end`方法中处理，
    //Spannable.SPAN_MARK_MARK表示当文本插入偏移时，它们仍然保持在它们的原始偏移量上。从概念上讲，文本是在标记之后添加的。
    text.setSpan(mark, len, len, Spannable.SPAN_MARK_MARK);
}

private static void end(SpannableStringBuilder text, Class kind,Object repl) {
    //当前字符长度
    int len = text.length();
    //根据kind获取最后一个set进去的对象
    Object obj = getLast(text, kind);
    //获取标签起始位置
    int where = text.getSpanStart(obj);
    //去除标记对象
    text.removeSpan(obj);

    if (where != len) {
        //len则为结束的位置，Spannable.SPAN_EXCLUSIVE_EXCLUSIVE是设置样式文字区间为闭区间
        //将真正的样式对象repl设置进去，Bold对应StyleSpan类型，Typeface.BOLD 加粗样式
        text.setSpan(repl, where, len, Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);
    }
}

private static Object getLast(Spanned text, Class kind) {
    /*
     * 获取最后一个类型为king，在setSpan传入的对象
     * 例如kind类型为Bold.class，则会返回在start中set进去的Bold对象
     */
    Object[] objs = text.getSpans(0, text.length(), kind);

    if (objs.length == 0) {
        return null;
    } else {
        //如果有期间有多个，则获取最后一个
        return objs[objs.length - 1];
    }
}

```

经过`start`和`end`方法处理后，`strong`标签中的文本就被加粗，具体的样式类型这里不做详解，后续可以参考[Spannable源码解析](https://github.com/LittleFriendsGroup/AndroidSdkSourceAnalysis)这篇**目前还没人认领**文章，其他为字体设置不同的样式过程一致，在`handleStartTag`根据不同标签类型调用`start`时方法传入不同对象给mark，并在`handleEndTag`中不同标签调用`end`并传入不同样式。


#### 4. font标签

`font`标签可以给字符串指定颜色和字体。


```java
private void handleStartTag(String tag, Attributes attributes) {
    ...
    else if (tag.equalsIgnoreCase("font")) {
        //attributes带有标签中的属性
        //例如<font color="#FFFFFF">,属性将以key-value的形式存在，{"color":"#FFFFFF"}。
        startFont(mSpannableStringBuilder, attributes);
    }
    ...
}

private static void startFont(SpannableStringBuilder text,Attributes attributes) {
    String color = attributes.getValue("", "color");//获取color属性
    String face = attributes.getValue("", "face");//获取face属性

    int len = text.length();
    //Font同样是一个用来标记属性的对象，没有实际功能
    text.setSpan(new Font(color, face), len, len, Spannable.SPAN_MARK_MARK);
}

//保存颜色值和字体类型
private static class Font {
    public String mColor;
    public String mFace;

    public Font(String color, String face) {
        mColor = color;
        mFace = face;
    }
}

private void handleEndTag(String tag) {
    ...
    else if (tag.equalsIgnoreCase("font")) {
        endFont(mSpannableStringBuilder);
    }   
    ...
}

private static void endFont(SpannableStringBuilder text) {
    int len = text.length();
    Object obj = getLast(text, Font.class);
    int where = text.getSpanStart(obj);

    text.removeSpan(obj);

    if (where != len) {
        Font f = (Font) obj;
        //前面与strong标签解析过程相似，多了下面处理颜色和字体的逻辑
        if (!TextUtils.isEmpty(f.mColor)) {
            //如果color属性中以"@"开头，则是获取colorId对应的颜色值
            //注意：只能支持android.R的资源
            if (f.mColor.startsWith("@")) {
                Resources res = Resources.getSystem();
                String name = f.mColor.substring(1);
                int colorRes = res.getIdentifier(name, "color", "android");
                if (colorRes != 0) {
                    //也可以是color selector，则会根据不同状态显示不同颜色
                    ColorStateList colors = res.getColorStateList(colorRes, null);
                    //1、通过TextAppearanceSpan设置颜色
                    text.setSpan(new TextAppearanceSpan(null, 0, 0, colors, null),
                            where, len,
                            Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);
                }
            } else {
                //如果为"#"开头则解析颜色值
                int c = Color.getHtmlColor(f.mColor);
                if (c != -1) {
                    //2、通过ForegroundColorSpan直接设置字体的rgb值
                    text.setSpan(new ForegroundColorSpan(c | 0xFF000000),
                            where, len,
                            Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);
                }
            }
        }
        if (f.mFace != null) {
            //如果有face参数则通过TypefaceSpan设置字体
            text.setSpan(new TypefaceSpan(f.mFace), where, len,
                         Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);
        }
    }
}

```

具体支持哪些字体，在`TypefaceSpan`的`apply`方法中会先去解析对应的字体，然后绘制出来，源码如下。

```java
private static void apply(Paint paint, String family) {
        ...
        //解析字体
        Typeface tf = Typeface.create(family, oldStyle);
        ...
    }

```

`Typeface`源码
```java

/**
     * 根据字体名称获取字体对象，如果familyName为null，则返回默认字体对象
     * 调用getStyle可查看该字体style属性
     *
     * @param 字体名称，可能为null
     * @param style  NORMAL（标准）, BOLD（粗体）, ITALIC（斜体）, BOLD_ITALIC（粗斜）
     * @return 匹配的字体
     */
    public static Typeface create(String familyName, int style) {
        if (sSystemFontMap != null) {
            //字体缓存在sSystemFontMap中
            return create(sSystemFontMap.get(familyName), style);
        }
        return null;
    }

    //init方法中初始化sSystemFontMap
 private static void init() {
        // 获取字体配置文件目录
        //private static File getSystemFontConfigLocation() {
        //return new File("/system/etc/");
        //}
        File systemFontConfigLocation = getSystemFontConfigLocation();
        //获取字体配置文件
        //static final String FONTS_CONFIG = "fonts.xml";
        File configFilename = new File(systemFontConfigLocation, FONTS_CONFIG);
        try {
            //将字体名称更Typeface对象缓存在map中
            //具体解析过程忽略,有兴趣可自行翻阅源码
            ....
            sSystemFontMap = systemFonts;

        } catch (RuntimeException e) {
           ....
        }
    }

```

具体支持的字体因不同系统而不同，这里附带一份某手机[fonts.xml文件]()。

#### 5. img标签

```java
//img标签只有在标签开始时处理
private void handleStartTag(String tag, Attributes attributes) {
    ...
    else if (tag.equalsIgnoreCase("img")) {
            startImg(mSpannableStringBuilder, attributes, mImageGetter);
    }
    ...
}

//与其他标签处理过程多了Attributes标签属性，Html.ImageGetter 自定义图片获取
private static void startImg(SpannableStringBuilder text,
                                 Attributes attributes, Html.ImageGetter img) {
        //获取src属性
        String src = attributes.getValue("", "src");
        Drawable d = null;

        if (img != null) {
            //调用自定义的图片获取方式，并传入src属性值
            d = img.getDrawable(src);
        }

        if (d == null) {
            //如果图片为空，则返回一个小方块
            d = Resources.getSystem().
                    getDrawable(com.android.internal.R.drawable.unknown_image);
            d.setBounds(0, 0, d.getIntrinsicWidth(), d.getIntrinsicHeight());
        }

        int len = text.length();
        //添加图片占位字符
        text.append("\uFFFC");
        //通过使用ImageSpan设置图片效果
        text.setSpan(new ImageSpan(d, src), len, text.length(),
                     Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);
    }


```

#### 6. 自定义标签

```java
private void handleStartTag(String tag, Attributes attributes) {
    ...
    else if (mTagHandler != null) {//通过自定义标签处理器来扩展自定义标签
            mTagHandler.handleTag(true, tag, mSpannableStringBuilder, mReader);
    }
    ...
}

private void handleEndTag(String tag) {
    ...
    else if (mTagHandler != null) {
        //闭合标签
        mTagHandler.handleTag(false, tag, mSpannableStringBuilder, mReader);
    }
    ...
}

```

关于自定义标签有个小问题是，`handleTag`并没有传入`Attributes`标签属性，所以无法直接获取自定义标签的属性值，下面给出两种方案解决这个问题：
1. 通过某一部分标签名作为属性值，例如`<custom>`标签，我们想加入id的参数，则可将标签名变为`<custom-id-123>`，然后在`handleTag`中自行解析。
2. 通过反射`XMLReader`来获取属性值，具体例子可参考[stackoverflow:How to get an attribute from an XMLReader](http://stackoverflow.com/questions/6952243/how-to-get-an-attribute-from-an-xmlreader/36528149)

### 3、5 convert方法剩下部分

不要忽略了`parse`之后还有一部分代码。

```java
    // 修正段落标记范围
    //ParagraphStyle为段落级别样式
    Object[] obj = mSpannableStringBuilder.getSpans(0, mSpannableStringBuilder.length(), ParagraphStyle.class);
    for (int i = 0; i < obj.length; i++) {
        int start = mSpannableStringBuilder.getSpanStart(obj[i]);
        int end = mSpannableStringBuilder.getSpanEnd(obj[i]);

        // 去除末尾两个换行符
        if (end - 2 >= 0) {
            if (mSpannableStringBuilder.charAt(end - 1) == '\n' &&
                mSpannableStringBuilder.charAt(end - 2) == '\n') {
                end--;
            }
        }

        if (end == start) {
            //除去没有显示的样式
            mSpannableStringBuilder.removeSpan(obj[i]);
        } else {
            //Spannable.SPAN_PARAGRAPH以换行符为起始点和终点
            mSpannableStringBuilder.setSpan(obj[i], start, end, Spannable.SPAN_PARAGRAPH);
        }
    }

    return mSpannableStringBuilder;
```

### 3、6 toHtml解析

理解该方法之前先看看`android.text.style`包中部分样式的UML图帮助理解：
![uml](https://github.com/DennyCai/AndroidSdkSourceAnalysis/blob/master/img/uml-style.png?raw=true)

根据uml可知，样式级别主要分为两类：一类是段落级别，另一类是字符级别。
`toHtml`方法是从大范围级别到小范围级别解析，即先解析段落样式再解析字符样式。

```java

public static String toHtml(Spanned text) {
    StringBuilder out = new StringBuilder();
    withinHtml(out, text);
    return out.toString();
}

```
```java
private static void withinHtml(StringBuilder out, Spanned text) {
    int len = text.length();
    //遍历段落级别样式
    int next;
    for (int i = 0; i < text.length(); i = next) {
        //返回样式类型出现的开始位置
        next = text.nextSpanTransition(i, len, ParagraphStyle.class);
        //返回从i位置到下个段落样式位置之前带有段落样式的数组，比较拗口，总之是一段一段取出来
        ParagraphStyle[] style = text.getSpans(i, next, ParagraphStyle.class);
        String elements = " ";
        boolean needDiv = false;
        //然后逐个判断是否为AlignmentSpan类型，是则加上<div align="">标签，align属性代表段落中字符对齐方向
        for(int j = 0; j < style.length; j++) {
            if (style[j] instanceof AlignmentSpan) {
                Layout.Alignment align =
                    ((AlignmentSpan) style[j]).getAlignment();
                needDiv = true;
                if (align == Layout.Alignment.ALIGN_CENTER) {
                   elements = "align=\"center\" " + elements;
                } else if (align == Layout.Alignment.ALIGN_OPPOSITE) {
                    elements = "align=\"right\" " + elements;
                } else {
                   elements = "align=\"left\" " + elements;
                }
            }
        }

        if (needDiv) {//添加div标签
            out.append("<div ").append(elements).append(">");
        }
        //解析段落样式中其他样式
        withinDiv(out, text, i, next);
        //调用层次较深，先忽略后面
        ....
    }
```
```java

//start为段落起点，end为结尾
private static void withinDiv(StringBuilder out, Spanned text,int start, int end) {
    int next;
    for (int i = start; i < end; i = next) {
        //查找引用样式
        next = text.nextSpanTransition(i, end, QuoteSpan.class);
        QuoteSpan[] quotes = text.getSpans(i, next, QuoteSpan.class);

        for (QuoteSpan quote : quotes) {
            out.append("<blockquote>");
        }
        //解析里面的样式
        withinBlockquote(out, text, i, next);
        //忽略
        ....
    }
}
```
```java

private static void withinBlockquote(StringBuilder out, Spanned text,int start, int end) {
    //解析文本显示方向
    out.append(getOpenParaTagWithDirection(text, start, end));
    ...
    int next;
    for (int i = start; i < end; i = next) {
        next = TextUtils.indexOf(text, '\n', i, end);
        if (next < 0) {
           next = end;
        }

        int nl = 0;

        while (next < end && text.charAt(next) == '\n') {
            nl++;
            next++;
        }
        if (withinParagraph(out, text, i, next - nl, nl, next == end)) {
           ...
        }

        ...
}
```
```java

private static String getOpenParaTagWithDirection(Spanned text, int start, int end) {
    final int len = end - start;
    final byte[] levels = ArrayUtils.newUnpaddedByteArray(len);//通过VMRuntime创建byte数组
    final char[] buffer = TextUtils.obtain(len);//底层通过ArrayUtils.newUnpaddedCharArray(len);创建char数组
    TextUtils.getChars(text, start, end, buffer, 0);//将字符缓存到buffer数组中
    //通过bidi来解析文字方向，作用是兼容多国语言的排列方向，例如阿利伯文字排列是自右向左。
    //里面主要调用native方法，具体可参考系统源码。
    int paraDir = AndroidBidi.bidi(Layout.DIR_REQUEST_DEFAULT_LTR, buffer, levels, len,false /* no info */);
    //使用p标签加dir属性修饰文本方向
    switch(paraDir) {
        case Layout.DIR_RIGHT_TO_LEFT:
            return "<p dir=\"rtl\">";
        case Layout.DIR_LEFT_TO_RIGHT:
        default:
            return "<p dir=\"ltr\">";
    }
}
```
```java

private static void withinBlockquote(StringBuilder out, Spanned text,int start, int end) {
    //解析文本显示方向
    out.append(getOpenParaTagWithDirection(text, start, end));
    //回来
    int next;
    for (int i = start; i < end; i = next) {
        next = TextUtils.indexOf(text, '\n', i, end);
        if (next < 0) {
           next = end;
        }

        int nl = 0;

        while (next < end && text.charAt(next) == '\n') {
            nl++;//统计换行符个数
            next++;
        }
        //以一个换行符为一段进行解析
        if (withinParagraph(out, text, i, next - nl, nl, next == end)) {
           ...
        }

        ...
}
```
```java

//代码较多，截取前部分
//start为起始位置，end为除去换行符文本结尾位置，nl换行符个数，last是否为文本末尾
private static boolean withinParagraph(StringBuilder out, Spanned text,
                                        int start, int end, int nl,
                                        boolean last) {
        int next;
        for (int i = start; i < end; i = next) {
            //获取CharacterStyle样式
            next = text.nextSpanTransition(i, end, CharacterStyle.class);
            CharacterStyle[] style = text.getSpans(i, next,
                                                   CharacterStyle.class);

            for (int j = 0; j < style.length; j++) {
                //StyleSpan样式
                if (style[j] instanceof StyleSpan) {
                    int s = ((StyleSpan) style[j]).getStyle();
                    //粗体使用b标签包裹
                    if ((s & Typeface.BOLD) != 0) {
                        out.append("<b>");
                    }
                    //斜体使用i标签包裹
                    if ((s & Typeface.ITALIC) != 0) {
                        out.append("<i>");
                    }
                }
                //TypefaceSpan样式
                if (style[j] instanceof TypefaceSpan) {
                    //获取字体类型
                    String s = ((TypefaceSpan) style[j]).getFamily();
                    //monospace使用tt标签包裹
                    if ("monospace".equals(s)) {
                        out.append("<tt>");
                    }
                }
                //上标样式使用sup标签包裹
                if (style[j] instanceof SuperscriptSpan) {
                    out.append("<sup>");
                }
                //小标使用sub标签包裹
                if (style[j] instanceof SubscriptSpan) {
                    out.append("<sub>");
                }
                //下划线使用u标签变过
                if (style[j] instanceof UnderlineSpan) {
                    out.append("<u>");
                }
                //删除线使用strike标签
                if (style[j] instanceof StrikethroughSpan) {
                    out.append("<strike>");
                }
                //urlspan使用a标签加href属性的标签包裹
                if (style[j] instanceof URLSpan) {
                    out.append("<a href=\"");
                    out.append(((URLSpan) style[j]).getURL());
                    out.append("\">");
                }
                //图片使用img加src属性的标签包裹
                if (style[j] instanceof ImageSpan) {
                    out.append("<img src=\"");
                    out.append(((ImageSpan) style[j]).getSource());
                    out.append("\">");

                    // Don't output the dummy character underlying the image.
                    i = next;
                }
                //绝对大小样式使用font加size属性标签包裹
                if (style[j] instanceof AbsoluteSizeSpan) {
                    out.append("<font size =\"");
                    out.append(((AbsoluteSizeSpan) style[j]).getSize() / 6);
                    out.append("\">");
                }
                //前景色使用font加color属性标签，只支持#
                //不支持@描述的资源颜色，因为@使用TextAppearanceSpan类型
                if (style[j] instanceof ForegroundColorSpan) {
                    out.append("<font color =\"#");
                    String color = Integer.toHexString(((ForegroundColorSpan)
                            style[j]).getForegroundColor() + 0x01000000);
                    while (color.length() < 6) {
                        color = "0" + color;
                    }
                    out.append(color);
                    out.append("\">");
                }
            }

            withinStyle(out, text, i, next);
            ....
        }
}
```
```java

    //将字符进行转码
    private static void withinStyle(StringBuilder out, CharSequence text,
                                    int start, int end) {
        for (int i = start; i < end; i++) {
            char c = text.charAt(i);
            //将<  >  & '空格' 变成转译字符
            if (c == '<') {
                out.append("&lt;");
            } else if (c == '>') {
                out.append("&gt;");
            } else if (c == '&') {
                out.append("&amp;");
            } else if (c >= 0xD800 && c <= 0xDFFF) {//代理区域
                //java使用的是Unicode编码，所以有一个代理区的概念，主要作用是扩展字符集
                //只支持代理区
                if (c < 0xDC00 && i + 1 < end) {
                    char d = text.charAt(i + 1);
                    if (d >= 0xDC00 && d <= 0xDFFF) {
                        i++;
                        int codepoint = 0x010000 | (int) c - 0xD800 << 10 | (int) d - 0xDC00;
                        out.append("&#").append(codepoint).append(";");
                    }
                }
            } else if (c > 0x7E || c < ' ') {
                out.append("&#").append((int) c).append(";");
            } else if (c == ' ') {
                //多个空格转译为&nbsp;
                while (i + 1 < end && text.charAt(i + 1) == ' ') {
                    out.append("&nbsp;");
                    i++;
                }

                out.append(' ');
            } else {
                out.append(c);
            }
        }
    }
```
```java

//跳出withinStyle接着回到withinParagraph方法
private static boolean withinParagraph(StringBuilder out, Spanned text,
                                        int start, int end, int nl,
                                        boolean last) {
    ...
    //接下来很好理解，把上面包裹的标签闭合
    for (int j = style.length - 1; j >= 0; j--) {
                if (style[j] instanceof ForegroundColorSpan) {
                    out.append("</font>");
                }
                if (style[j] instanceof AbsoluteSizeSpan) {
                    out.append("</font>");
                }
                if (style[j] instanceof URLSpan) {
                    out.append("</a>");
                }
                if (style[j] instanceof StrikethroughSpan) {
                    out.append("</strike>");
                }
                if (style[j] instanceof UnderlineSpan) {
                    out.append("</u>");
                }
                if (style[j] instanceof SubscriptSpan) {
                    out.append("</sub>");
                }
                if (style[j] instanceof SuperscriptSpan) {
                    out.append("</sup>");
                }
                if (style[j] instanceof TypefaceSpan) {
                    String s = ((TypefaceSpan) style[j]).getFamily();

                    if (s.equals("monospace")) {
                        out.append("</tt>");
                    }
                }
                if (style[j] instanceof StyleSpan) {
                    int s = ((StyleSpan) style[j]).getStyle();

                    if ((s & Typeface.BOLD) != 0) {
                        out.append("</b>");
                    }
                    if ((s & Typeface.ITALIC) != 0) {
                        out.append("</i>");
                    }
                }
            }
        }

        //添加br标签
        if (nl == 1) {
            out.append("<br>\n");
            return false;
        } else {
            for (int i = 2; i < nl; i++) {
                out.append("<br>");
            }
            return !last;
        }
    }
```
```java

//跳出withinParagraph回到withinBlockquote剩下部分
private static void withinBlockquote(StringBuilder out, Spanned text,int start, int end) {
    ...
    if (withinParagraph(out, text, i, next - nl, nl, next == end)) {
        /* 闭合p标签 */
        out.append("</p>\n");
        out.append(getOpenParaTagWithDirection(text, next, end));//判断字符排列顺序，添加p标签
    }
    //闭合p标签
    out.append("</p>\n");

}
```
```java

private static void withinDiv(StringBuilder out, Spanned text,
            int start, int end) {
        ...
        withinBlockquote(out, text, i, next);
        //逐个闭合
        for (QuoteSpan quote : quotes) {
            out.append("</blockquote>\n");
        }
    }
}
```
```java

private static void withinHtml(StringBuilder out, Spanned text) {
    ...
    withinDiv(out, text, i, next);
    //按需要添加闭合div标签，即有AlignmentSpan类型样式
    if (needDiv) {
        out.append("</div>");
    }
}

```

### 3、7 escapeHtml解析

```java
public static String escapeHtml(CharSequence text) {
        StringBuilder out = new StringBuilder();
        //直接调用withinStyle方法进行转译
        withinStyle(out, text, 0, text.length());
        return out.toString();
    }
```

## 4、总结

`Html.fromHtml`主要是使用SAX解析HTML标签，使用Spanned为TextView提供强大的富文本支持，还可以通过定制TagHandler来扩展自定义标签，ImageGetter加载自定义图片资源，感兴趣的读者可到`android.text.style`包中继续探究其他支持的样式，`toHtml`主要做了反向解析，`escapeHtml`只调用`withinStyle`方法，里面涉及到编码转换，关于编码方面可参考[Java中char和String 的深入理解 - 字符编码](http://blog.csdn.net/u010297957/article/details/48495791)，第一次写源码分析文章，如有差错处，欢迎吐槽。