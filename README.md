##HtmlExtractor是一个Java实现的基于模板的网页结构化信息精准抽取组件，本身并不包含爬虫功能，但可被爬虫或其他程序调用以便更精准地对网页结构化信息进行抽取。

##HtmlExtractor是为大规模分布式环境设计的，采用主从架构，主节点负责维护抽取规则，从节点向主节点请求抽取规则，当抽取规则发生变化，主节点主动通知从节点，从而能实现抽取规则变化之后的实时动态生效。

##[捐赠致谢](https://github.com/ysc/QuestionAnsweringSystem/wiki/donation)

##如何使用？

    HtmlExtractor由2个子项目构成，html-extractor和html-extractor-web。
    html-extractor实现了数据抽取逻辑，是从节点，html-extractor-web提供web界面来维护抽取规则，是主节点。
    html-extractor是一个jar包，可通过maven引用：
    <dependency>
        <groupId>org.apdplat</groupId>
        <artifactId>html-extractor</artifactId>
        <version>1.1</version>
    </dependency>
    html-extractor-web是一个war包，需要部署到Servlet/Jsp容器上。
    在html-extractor-web目录下运行mvn jetty:run就可以启动Servlet/Jsp容器jetty，之后打开浏览器访问：
    http://localhost:8080/html-extractor-web/api/ 查看自己定义的规则。
    
    注意：页面模板中定义的所有CSS路径和抽取表达式全部抽取成功，才算抽取成功，
         只要有一个CSS路径或抽取表达式失败，就是抽取失败。
         
[如何使用HtmlExtractor实现基于模板的网页结构化信息精准抽取?](http://my.oschina.net/apdplat/blog/402149)

##单机集中式使用方法：

    //1、构造抽取规则
    List<UrlPattern> urlPatterns = new ArrayList<>();
    //1.1、构造URL模式
    UrlPattern urlPattern = new UrlPattern();
    urlPattern.setUrlPattern("http://money.163.com/\\d{2}/\\d{4}/\\d{2}/[0-9A-Z]{16}.html");
    //1.2、构造HTML模板
    HtmlTemplate htmlTemplate = new HtmlTemplate();
    htmlTemplate.setTemplateName("网易财经频道");
    htmlTemplate.setTableName("finance");
    //1.3、将URL模式和HTML模板建立关联
    urlPattern.addHtmlTemplate(htmlTemplate);
    //1.4、构造CSS路径
    CssPath cssPath = new CssPath();
    cssPath.setCssPath("h1");
    cssPath.setFieldName("title");
    cssPath.setFieldDescription("标题");
    //1.5、将CSS路径和模板建立关联
    htmlTemplate.addCssPath(cssPath);
    //1.6、构造CSS路径
    cssPath = new CssPath();
    cssPath.setCssPath("div#endText");
    cssPath.setFieldName("content");
    cssPath.setFieldDescription("正文");
    //1.7、将CSS路径和模板建立关联
    htmlTemplate.addCssPath(cssPath);
    //可象上面那样构造多个URLURL模式
    urlPatterns.add(urlPattern);
    
    //2、获取抽取规则对象
    ExtractRegular extractRegular = ExtractRegular.getInstance(urlPatterns);
    //注意：可通过如下3个方法动态地改变抽取规则
    //extractRegular.addUrlPatterns(urlPatterns);
    //extractRegular.addUrlPattern(urlPattern);
    //extractRegular.removeUrlPattern(urlPattern.getUrlPattern());
    
    //3、获取HTML抽取工具
    HtmlExtractor htmlExtractor = new DefaultHtmlExtractor(extractRegular);
    
    //4、抽取网页
    String url = "http://money.163.com/08/2025-10-30THR2TMP002533QK.html";
    HtmlFetcher htmlFetcher = new JSoupHtmlFetcher();
    String html = htmlFetcher.fetch(url);
    List<ExtractResult> extractResults = htmlExtractor.extract(url, html);
    
    //5、输出结果
    int i = 1;
    for (ExtractResult extractResult : extractResults) {
        System.out.println((i++) + "、网页 " + extractResult.getUrl() + " 的抽取结果");
        if(!extractResult.isSuccess()){
            System.out.println("抽取失败：");
            for(ExtractFailLog extractFailLog : extractResult.getExtractFailLogs()){
                System.out.println("\turl:"+extractFailLog.getUrl());
                System.out.println("\turlPattern:"+extractFailLog.getUrlPattern());
                System.out.println("\ttemplateName:"+extractFailLog.getTemplateName());
                System.out.println("\tfieldName:"+extractFailLog.getFieldName());
                System.out.println("\tfieldDescription:"+extractFailLog.getFieldDescription());
                System.out.println("\tcssPath:"+extractFailLog.getCssPath());
                if(extractFailLog.getExtractExpression()!=null) {
                    System.out.println("\textractExpression:" + extractFailLog.getExtractExpression());
                }
            }
            continue;
        }
        Map<String, List<ExtractResultItem>> extractResultItems = extractResult.getExtractResultItems();
        for(String field : extractResultItems.keySet()){
            List<ExtractResultItem> values = extractResultItems.get(field);
            if(values.size() > 1){
                int j=1;
                System.out.println("\t多值字段:"+field);
                for(ExtractResultItem item : values){
                    System.out.println("\t\t"+(j++)+"、"+field+" = "+item.getValue());   
                }
            }else{
                System.out.println("\t"+field+" = "+values.get(0).getValue());     
            }
        }
        System.out.println("\tdescription = "+extractResult.getDescription());
        System.out.println("\tkeywords = "+extractResult.getKeywords());
    }

##多机分布式使用方法：

    1、运行主节点，负责维护抽取规则：
    方法一：在html-extractor-web目录下运行mvn jetty:run 。
    方法二：在html-extractor-web目录下运行mvn install ，
          然后将target/html-extractor-web-1.0.war部署到Tomcat。

    2、获取一个HtmlExtractor的实例（从节点），示例代码如下：
    String allExtractRegularUrl = "http://localhost:8080/HtmlExtractorServer/api/all_extract_regular.jsp";
    String redisHost = "localhost";
    int redisPort = 6379;
    ExtractRegular extractRegular = ExtractRegular.getInstance(allExtractRegularUrl, redisHost, redisPort);
    HtmlExtractor htmlExtractor = new DefaultHtmlExtractor(extractRegular);

    3、抽取信息，示例代码如下：
    String url = "http://money.163.com/08/2025-10-30THR2TMP002533QK.html";
    HtmlFetcher htmlFetcher = new JSoupHtmlFetcher();
    String html = htmlFetcher.fetch(url);
    List<ExtractResult> extractResults = htmlExtractor.extract(url, html);

    int i = 1;
    for (ExtractResult extractResult : extractResults) {
        System.out.println((i++) + "、网页 " + extractResult.getUrl() + " 的抽取结果");
        if(!extractResult.isSuccess()){
            System.out.println("抽取失败：");
            for(ExtractFailLog extractFailLog : extractResult.getExtractFailLogs()){
                System.out.println("\turl:"+extractFailLog.getUrl());
                System.out.println("\turlPattern:"+extractFailLog.getUrlPattern());
                System.out.println("\ttemplateName:"+extractFailLog.getTemplateName());
                System.out.println("\tfieldName:"+extractFailLog.getFieldName());
                System.out.println("\tfieldDescription:"+extractFailLog.getFieldDescription());
                System.out.println("\tcssPath:"+extractFailLog.getCssPath());
                if(extractFailLog.getExtractExpression()!=null) {
                    System.out.println("\textractExpression:" + extractFailLog.getExtractExpression());
                }
            }
            continue;
        }
        for(ExtractResultItem extractResultItem : extractResult.getExtractResultItems()){
            System.out.print("\t"+extractResultItem.getField()+" = "+extractResultItem.getValue());              
        }
        System.out.println("\tdescription = "+extractResult.getDescription());
        System.out.println("\tkeywords = "+extractResult.getKeywords());
    }
