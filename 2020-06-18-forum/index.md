# MyBatis-Plus 实现论坛 API


网络论坛，常简称为论坛，又称讨论区、讨论版等，是种提供在线讨论的程序，或由这些程序建立的以在线讨论为主的网站。本文记录如何用 Spring Boot 结合 MyBatis-Plus 实现论坛中常见的 API 接口。

## 数据分页获取

### 引入依赖，连接数据库

在 spring-boot 项目的根目录 pom.xml 添加依赖，相关依赖有 lombok, mybatis-plus, druid, swagger, mysql 驱动等。

```xml
<dependency>
	<groupId>com.baomidou</groupId>
	<artifactId>mybatis-plus-boot-starter</artifactId>
	<version>3.1.0</version>
</dependency>

<dependency>
	<groupId>org.projectlombok</groupId>
	<artifactId>lombok</artifactId>
</dependency>

<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>druid-spring-boot-starter</artifactId>
	<version>1.1.9</version>
</dependency>

<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
	<scope>runtime</scope>
	<version>8.0.12</version>
</dependency>

<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.8.0</version>
</dependency>

<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.8.0</version>
</dependency>
```

然后在 src/main/resources 文件夹下添加 application.properties 或 application.yml 文件配置数据库连接。

### 创建数据表

本地连接数据库，然后导入脚本，创建数据表。

### 创建 entity 及 mapper

创建文章实体(entity)

```java
@Data
@EqualsAndHashCode(callSuper = false)
@Accessors(chain = true)
@ApiModel(value = "Article对象", description = "")
public class Article implements Serializable {

    private static final long serialVersionUID = 1L;

    @TableId(value = "id", type = IdType.AUTO)
    private Integer id;

    @ApiModelProperty(value = "标题")
    private String title;

    @ApiModelProperty(value = "内容")
    private String content;

    @ApiModelProperty(value = "用户主键")
    private Integer uId;

    @ApiModelProperty(value = "热度")
    private Integer score;

    @ApiModelProperty(value = "版块主键/为0时表示未设置")
    private Integer bId;

    @JsonFormat(pattern = "yyyy-MM-dd", timezone = "GMT+8")
    private Date createDate;

    @JsonFormat(pattern = "yyyy-MM-dd", timezone = "GMT+8")
    private Date updateDate;

    @TableField(exist = false)
    private User user;

    @TableField(exist = false)
    private Block block;

}
```

lombok 的注解：

- @Data 相当于 @Getter @Setter @RequiredArgsConstructor @ToString @EqualsAndHashCode 这 5 个注解的合集。
- 存在继承,编译时有警提示，加上@EqualsAndHashCode(callSuper=false), 只比较当前的类字段。
- @Accessors(chain = true)链式调用为真, setter 方法返回当前对象的序列化对象，便于存储，便于传输。

mybatis-plus 的注解：

- @TableId 主键的映射,主键的生成策略,自动生成。
- @TableField(exist = false) 不映射数据表字段。

swagger 的注解：

- @Api 开头的是 swagger 的注解。

对应的 Mapper 只需继承 BaseMapper，基本的 crud 方法 mybatis-plus 已经封装好了。

```java
@Mapper
@Component
public interface ArticleMapper extends BaseMapper<Article> {

}
```

@Component 注解代表需要被 Spring IoC 容器管理，才能实现依赖注入。

然后在启动类中加入 @MapperScan("") 就可以完成 spring-boot 到数据表的映射。

## 创建 service

创建 service，我们想要分页获取数据，先配置 mybatis-plus 分页插件：

```java
@Configuration
@MapperScan("com.forum.buge_server.mapper*")
public class MybatisPlusConfig {
    /**
     * 加载分页插件
     *
     * @return
     */
    @Bean
    public PaginationInterceptor paginationInterceptor() {
        return new PaginationInterceptor();
    }
}
```

#### 定义 service 接口

```java
public interface ArticleService extends IService<Article> {

    IPage<Article> getArticlePageing(IPage<Article> iPage, LambdaQueryWrapper<Article> wrapper);

}
```

#### service 实现类

在实现类里注入 mapper, 完成分页获取数据。selectPage 方法接收两个参数，第一个是 IPage 对象，第二个参数是条件构造器 QueryWrapper。我们需要在 controller 中传递这两个参数。

```java
@Service
public class ArticleServiceImpl extends ServiceImpl<ArticleMapper, Article> implements ArticleService {

    @Autowired
    ArticleMapper articleMapper;

    @Override
    public IPage<Article> getArticlePageing(IPage<Article> iPage, LambdaQueryWrapper<Article> wrapper) {
        IPage<Article> articleIPage = articleMapper.selectPage(iPage, wrapper);
        return articleIPage;
    }
}
```

## 创建 controller

我们需要传递 IPage 对象，QueryWrapper 对象给 service，然后从返回的 IPage 对象中取得查询到的数据。

```java
@RestController
@RequestMapping("/open")
public class OpenController {

	@Autowired
    ArticleService articleService;

    @ApiOperation("分页查询文章，时间排序")
    @PostMapping("/article/new")
    public Object getArticleNew(Integer current, Integer size) {
        // 分页条件
        IPage<Article> page = new Page<>(current, size);
        // 查询条件
        LambdaQueryWrapper<Article> wrapper = new LambdaQueryWrapper<>();
        // 根据时间顺序查询
        wrapper.orderByDesc(Article::getCreateDate);
        IPage<Article> articlePageing = articleService.getArticlePageing(page, wrapper);

        // 当前页数
        long atCurrent = articlePageing.getCurrent();
        // 当前页的数据
        List<Article> records = articlePageing.getRecords();
        for (Article article : records) {
            article.setContent("");
            article.setUser(userService.getById(article.getUId()));
            article.setBlock(blockService.getById(article.getBId()));
        }
        // 组装Dto
        PageResult pageResult = new PageResult();
        pageResult.setCurrent(atCurrent);
        pageResult.setList(records);

        return new JsonResult(200, "查询成功", pageResult);
    }
}
```

这样就实现了数据时间排序的分页获取。

## JWT 登录认证

添加依赖

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.7.0</version>
</dependency>
```

写一个 jwt 工具类

```java
@Component
@Data
public class JwtConfig {
    private String secret = "abcdefg1234567";
    private long expire = 3600 * 60 * 60;
    private String header = "token";

    /**
     * 生成token
     *
     * @param subject
     * @return
     */
    public String createToken(String subject) {
        Date nowDate = new Date();
        Date expireDate = new Date(nowDate.getTime() + expire * 1000);// 过期时间

        return Jwts.builder().setHeaderParam("typ", "JWT").setSubject(subject).setIssuedAt(nowDate)
                .setExpiration(expireDate).signWith(SignatureAlgorithm.HS512, secret).compact();
    }

    /**
     * 获取token中注册信息
     *
     * @param token
     * @return
     */
    public Claims getTokenClaim(String token) {
        try {
            return Jwts.parser().setSigningKey(secret).parseClaimsJws(token).getBody();
        } catch (Exception e) {
            // e.printStackTrace();
            return null;
        }
    }

    /**
     * 验证token是否过期失效
     *
     * @param expirationTime
     * @return
     */
    public boolean isTokenExpired(Date expirationTime) {
        return expirationTime.before(new Date());
    }
}
```

里面写了 token 的生成和解密的方法。

添加 jwt 拦截器, 除了注册，登录和 get 请求的 uri 地址，都需要验证 token。

```java
@Component
public class TokenInterceptor extends HandlerInterceptorAdapter {

    JwtConfig jwtConfig = new JwtConfig();

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {

        if (request.getMethod().equals("OPTIONS")) {
            return true;
        }

        /** 地址过滤 */
        String uri = request.getRequestURI();
        if (uri.contains("/api/login") || uri.contains("/api/reg") || uri.contains("/sys/login")
                || uri.contains("/api/get")) {
            return true;
        }

        /** Token 验证是否存在 */
        String token = request.getHeader(jwtConfig.getHeader());

        if (StringUtils.isEmpty(token)) {
            token = request.getParameter(jwtConfig.getHeader());
        }
        if (StringUtils.isEmpty(token)) {
            throw new SignatureException(jwtConfig.getHeader() + "不能为空");
        }

        Claims claims = null;
        // 判断是否有效
        try {
            claims = jwtConfig.getTokenClaim(token);
            if (claims == null || jwtConfig.isTokenExpired(claims.getExpiration())) {
                throw new SignatureException(jwtConfig.getHeader() + "失效，请重新登录。");
            }
        } catch (Exception e) {
            throw new SignatureException(jwtConfig.getHeader() + "失效，请重新登录。");
        }

        /** 设置 identityId 用户身份ID */
        request.setAttribute("identityId", claims.getSubject());
        return true;

    }
}
```

## 点赞关注收藏评论

点赞，关注，收藏，评论功能都可以用单表实现。点赞，收藏，评论功能就是在分别在 likes, collection, comment 表中记录 uid(用户),aid(文章) 实现关联，而关注则是在 follow 表中记录两个 uid 实现关联，下面给出点赞功能的逻辑实现，关注，收藏，评论功能的逻辑同点赞功能。

```java
    @PostMapping("/add/like")
    public Object setLike(Integer aid, HttpServletRequest request) {
        Claims token = jwtConfig.getTokenClaim(request.getHeader("token"));
        String subject = token.getSubject();
        LambdaQueryWrapper<Likes> queryWrapper = new LambdaQueryWrapper<>();
        queryWrapper.eq(Likes::getUid, Integer.valueOf(subject));
        queryWrapper.eq(Likes::getAId, aid);
        Likes one = likeService.getOne(queryWrapper);
        if (one == null) {
            boolean save = likeService.save(new Likes().setAId(aid).setUid(Integer.valueOf(subject)));
            return new JsonResult(200, "点赞成功!");
        } else {
            boolean b = likeService.removeById(one.getId());
            return new JsonResult(200, "已取消点赞!");
        }
    }
```

## 搜索和浏览记录

搜索使用 mysql 的模糊查询实现。

```java
    @PostMapping("/search/article")
    public Object searchArticle(String title) {
        LambdaQueryWrapper<Article> articleWrapper = new LambdaQueryWrapper<>();
        articleWrapper.like(Article::getTitle, title);
        List<Article> list = articleService.list(articleWrapper);
        for (Article article : list) {
            article.setContent("");
            article.setUser(userService.getById(article.getUId()));
            article.setBlock(blockService.getById(article.getBId()));
        }
        return new JsonResult(200, "", list);
    }
```

浏览记录，进入文章详情页面后，如果用户是登录状态，就加入 History 表。

History 表有三个字段，uid(用户), aid(文章), time(访问时间)。如果已经在 History 表中存在记录，则更新时间。

```java
    @GetMapping("/article/{id}")
    public Object getArticleById(@PathVariable("id") Integer id, HttpServletRequest request) {
        Claims token = jwtConfig.getTokenClaim(request.getHeader("token"));
        // 在登陆状态下保存到历史表中
        if (token != null) {
            String subject = token.getSubject();
            LambdaQueryWrapper<History> queryWrapper = new LambdaQueryWrapper<>();
            queryWrapper.eq(History::getAid, id);
            queryWrapper.eq(History::getUid, Integer.valueOf(subject));
            History one = historyService.getOne(queryWrapper);
            // 如果已存在记录，则更新时间
            History history = new History().setAid(id).setTime(new Date()).setUid(Integer.valueOf(subject));
            if (one == null) {
                historyService.save(history);
            } else {
                one.setTime(new Date());
                historyService.updateById(one);
            }
        }

        Article byId = articleService.getById(id);
        byId.setUser(userService.getById(byId.getUId()));
        byId.setBlock(blockService.getById(byId.getBId()));

        // 每次访问热度加1
        articleService.updateById(new Article().setId(id).setScore(byId.getScore() + 1));

        return new JsonResult(200, "查询成功", byId);
    }
```

每次访问文章详情页，文章热度发生变化，首页热度排序的内容将重新排序。

项目启动主类

```java
@SpringBootApplication
public class BugeServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(BugeServerApplication.class, args);
    }
}
```

spring boot 提供了一个统一的注解@SpringBootApplication。

代表了@Configuration, @EnableAutoConfiguration, @ComponentScan。

@Configuration 和 @Bean。使用这两个注解就可以创建一个简单的 spring 配置类，可以用来替代相应的 xml 配置文件。@Configuration 的注解类标识这个类可以使用 Spring IoC 容器作为 bean 定义的来源。@Bean 注解告诉 Spring，一个带有@Bean 的注解方法将返回一个对象，该对象应该被注册为在 Spring 应用程序上下文中的 bean。

@EnableAutoConfiguration：能够自动配置 spring 的上下文，试图猜测和配置你想要的 bean 类，通常会自动根据你的类路径和你的 bean 定义自动配置。

@ComponentScan：会自动扫描指定包下的全部标有@Component 的类，并注册成 bean，当然包括@Component 下的子注解@Service, @Repository, @Controller。

