# SpringBoot集成Swagger

## 引入Maven依赖

``` xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.2.2</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.2.2</version>
</dependency>
```

## 创建Swagger配置类

``` java
@Configuration
@EnableSwagger2
public class Swagger2Configuration {
    @Bean
    public Docket buildDocket() {
        return new Docket(DocumentationType.SWAGGER_2)
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.xxx.web.controller")) //要扫描的API(Controller)基础包
                .paths(PathSelectors.any()) // and by paths
                .build()
                .apiInfo(buildApiInf());
    }
    private ApiInfo buildApiInf() {
        return new ApiInfoBuilder()
                .title("Spring Boot中使用Swagger2 UI构建API文档")
                .contact("test")
                .version("1.0.0")
                .build();
    }
}
```

## 创建Controller并使用swagger构建API

* @Api注解用来表述该服务的信息，如果不使用则显示类名称.
* @ApiOperation注解用于表述接口信息
* @ApiParam注解用于描述接口的参数
  
``` java
@RestController
@RequestMapping("article")
@Api(value = "文章服务",description="提供 增删改查 API")
public class ArticleController {

    @Autowired
    private ArticleService articleServiceImpl;

    @PostMapping
    @ApiOperation("添加")
    public WebResult save(@RequestBody Article article) {
        return WebResult.ok(articleServiceImpl.save(article));
    }

    @PutMapping
    @ApiOperation("修改")
    public WebResult update(@RequestBody Article article) {
        return WebResult.ok(articleServiceImpl.update(article));
    }

    @DeleteMapping("{id}")
    @ApiOperation("删除")
    public WebResult delete(@ApiParam("文章id") @PathVariable String id) {
        return WebResult.ok(articleServiceImpl.remove(id));
    }

    @GetMapping("{id}")
    @ApiOperation("查找")
    public WebResult get(@ApiParam("文章id") @PathVariable String id) {
        return WebResult.ok(articleServiceImpl.get(id));
    }

}
```

## 为Model(JSON)添加注释(非必须)

``` java
@Entity
@ApiModel("Article(文章实体)")
public class Article extends BaseEntity {

    /**
     * 标题
     */
    @ApiModelProperty("标题")
    private String title;

    /**
     * 正文
     */
    @Type(type = "text")
    @ApiModelProperty("正文")
    private String content;

    /**
     * 发表时间
     */
    @JsonFormat(pattern = "yyyy/MM/dd")
    @ApiModelProperty("发表时间")
    private LocalDateTime publishedTime;

    /**
     * 作者
     */
    @ApiModelProperty("作者")
    private String author ;

    public Article() {
        this.publishedTime = LocalDateTime.now();
    }

    public Article(String title , String content) {
        this.title = title ;
        this.content = content ;
        this.publishedTime = LocalDateTime.now();
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public LocalDateTime getPublishedTime() {
        return publishedTime;
    }

    public void setPublishedTime(LocalDateTime publishedTime) {
        this.publishedTime = publishedTime;
    }

    public String getAuthor() {
        return author;
    }

    public void setAuthor(String author) {
        this.author = author;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }
}
```

## 验证

重新启动下Application，访问 http://localhost:8080/swagger-ui.html   (“项目访问链接”+“/swagger-ui.html”)。即可看到 swagger 生成的 API文档。

## 相关文档

* [Annotations-Reference documentation](https://github.com/swagger-api/swagger-core/wiki/Annotations-1.5.X)
* [swagger-api](https://github.com/swagger-api)
* [swagger-ui](https://github.com/swagger-api/swagger-ui)
* [swagger-core](https://github.com/swagger-api/swagger-core)
* [Reference documentation](http://springfox.github.io/springfox/docs/current/#springfox-configuration-and-demo-applications)