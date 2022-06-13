# 用 Spring Boot 和 Vue.js 写一个全栈应用


本篇文章将介绍如何用 Spring boot 与 Vue.js 创建一个具有基本 crud 功能的全栈 web 应用，我们将使用 bootstrap 作为项目的 UI 库，适合 web 开发初学者阅读。

## 后端接口

到 https://start.spring.io/ 去生成和下载 spring 应用，Group 填写 com.jpa，Artifact 填写 spring-jpa-demo， 其他默认选择就好，点击 Generate 就会生成并下载名压缩包，将压缩包解压，并在编辑器中打开。
我们使用 jpa 作为我们的 orm, 连接 mysql 数据库，因此在 pom.xml 中加入以下依赖。

```xml
		<!-- jpa driver -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
		<!-- spring web driver -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<!-- MySQL database driver -->
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
		</dependency>
		
		<!-- Skip test plugin -->
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-surefire-plugin</artifactId>
			<version>2.22.2</version>
			<configuration>
				<skipTests>true</skipTests>
			</configuration>
		</plugin>
		
```
### I. 数据库连接

编写数据库连接文件 application.properties，它在 resources 文件夹中。

```
## use create when running the app for the first time
## then change to "update" which just updates the schema when necessary
spring.datasource.url=jdbc:mysql://localhost:3306/notesapi?createDatabaseIfNotExist=true&useSSL=false&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=root
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
## this shows the sql actions in the terminal logs
spring.jpa.database-platform=org.hibernate.dialect.MySQL8Dialect
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
##optional, but just in case another application is listening on your default  port (8080)
server.port = 8034
```

这是我们的 src 文件夹结构，接下来的文件都会在 src 中编写。

![服务端文件结构](/img/spring-dir.jpg "服务端文件结构")

在 entity文件夹中新建文件 Notes.java，这是我们的数据表映射。

```java
package com.jpa.springjpademo.entity;

import javax.persistence.*;

import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.GenericGenerator;
import org.hibernate.annotations.UpdateTimestamp;

import java.util.Date;

@Entity
@Table(name = "notes_table")
@GenericGenerator(name = "jpa-uuid", strategy = "uuid") // 这个是hibernate的注解/生成32位UUID
public class Notes {
    @Id
    @GeneratedValue(generator = "jpa-uuid")
    @Column(name = "notes_id", nullable = false, length = 32)
    private String notes_id;

    // 默认创建时间
    @Column(name = "create_time")
    @Temporal(TemporalType.TIMESTAMP)
    @CreationTimestamp
    private Date time;

    // 默认更新时间
    @Column(name = "update_time")
    @Temporal(TemporalType.TIMESTAMP)
    @UpdateTimestamp
    private Date update_time;

    @Column(name = "title", nullable = true, length = 100)
    private String title;

    @Column(name = "description", nullable = true, length = 255)
    private String description;

    @Column(name = "content", nullable = true)
    @Lob
    @Basic(fetch = FetchType.LAZY)
    private String content;

    @Column(name = "author", nullable = true, length = 50)
    private String author;

    public String getId() {
        return notes_id;
    }

    public void setId(String notes_id) {
        this.notes_id = notes_id;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public String getAuthor() {
        return author;
    }

    public void setAuthor(String author) {
        this.author = author;
    }

    public void setUpdateTime(Date update_time) {
        this.update_time = update_time;
    }

    public Date getUpdateTime() {
        return update_time;
    }
}
```

在 dao 文件夹中新建文件 NotesDao.java, 只需实现 JpaRepository 接口，我们就能够连接到数据库。

```java
package com.jpa.springjpademo.dao;

import com.jpa.springjpademo.entity.Notes;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface NotesDao extends JpaRepository<Notes, String> {
}
```

接下来终端执行 `$ mvn clean package ` 下载依赖，编译代码。然后 `$ mvn spring-boot:run ` 运行项目。项目运行起来后刷新数据库，可以看到新生成名为 notesapi 的数据库，其中有一个名为 notes_table 的数据表。表示我们的数据库连接成功。

### II. 增删改查

五步实现完整的数据增删改查和接口测试：

- 1.实现 Conctroller 路由处理
- 2.实现 Service 数据库操作
- 3.实现 Exception 异常捕获
- 4.实现 Cors 跨域配置
- 5.实现 Swagger 文档配置

下面是详细步骤和代码示例：

- 实现 Conctroller 路由处理：

```java
package com.jpa.springjpademo.controller;

import com.jpa.springjpademo.entity.Notes;
import com.jpa.springjpademo.service.NotesService;
import org.springframework.http.ResponseEntity;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/notes")
public class NotesController {
    @Autowired
    NotesService notesService;

    @RequestMapping(value = "/all", method = RequestMethod.GET)
    public List<Notes> getAllNotes() {
        return notesService.getAllNotes();
    }

    @RequestMapping(value = "/create", method = RequestMethod.POST, consumes = MediaType.APPLICATION_JSON_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
    public Notes addNotes(@RequestBody Notes notes) {
        return notesService.addNotes(notes);
    }

    @RequestMapping(value = "/update", method = RequestMethod.PUT, consumes = MediaType.APPLICATION_JSON_VALUE, produces = MediaType.APPLICATION_JSON_VALUE)
    public Notes updateNotes(@RequestParam("notes_id") String id, @RequestBody Notes notes) {
        return notesService.updateNotes(id, notes);
    }

    @RequestMapping(value = "/{id}", method = RequestMethod.GET)
    public Notes getNotes(@PathVariable("id") String id) {
        return notesService.getNotesById(id);
    }

    @RequestMapping(value = "/delete/all", method = RequestMethod.DELETE)
    public void deleteAllNotes() {
        notesService.deleteAllNotes();
    }

    @RequestMapping(value = "/delete", method = RequestMethod.DELETE)
    public ResponseEntity<?> deleteNotes(@RequestParam("notes_id") String id) {
        return notesService.deleteNotesById(id);
    }
}
```

- 实现 Service 数据库操作：

```java
package com.jpa.springjpademo.service;

import com.jpa.springjpademo.dao.NotesDao;
import com.jpa.springjpademo.entity.Notes;
import com.jpa.springjpademo.exception.ResourceNotFoundException;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class NotesService {
    @Autowired
    NotesDao notesDao;

    public List<Notes> getAllNotes() {
        return notesDao.findAll();
    }

    public Notes addNotes(Notes notes) {
        return notesDao.save(notes);
    }

    public Notes getNotesById(String id) {
        return notesDao.findById(id).orElseThrow(() -> new ResourceNotFoundException("Notes", "id", id));
    }

    public Notes updateNotes(String id, Notes notes) {

        Notes tnotes = notesDao.findById(id).orElseThrow(() -> new ResourceNotFoundException("Notes", "id", id));
        tnotes.setTitle(notes.getTitle());
        tnotes.setDescription(notes.getDescription());
        tnotes.setContent(notes.getContent());
        tnotes.setAuthor(notes.getAuthor());
        return notesDao.save(tnotes);

    }

    public ResponseEntity<?> deleteNotesById(String id) {
        Notes notes = notesDao.findById(id).orElseThrow(() -> new ResourceNotFoundException("Notes", "id", id));
        notesDao.delete(notes);
        return ResponseEntity.ok().build();
    }

    public void deleteAllNotes() {
        notesDao.deleteAll();
    }
}
```

- 实现 Exception 异常捕获：

```java
package com.jpa.springjpademo.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(value = HttpStatus.NOT_FOUND)
public class ResourceNotFoundException extends RuntimeException {

    private static final long serialVersionUID = 1L;
    private String resourceName;
    private String fieldName;
    private Object fieldValue;

    public ResourceNotFoundException(String resourceName, String fieldName, Object fieldValue) {
        super(String.format("%s not found with %s : '%s'", resourceName, fieldName, fieldValue));
        this.resourceName = resourceName;
        this.fieldName = fieldName;
        this.fieldValue = fieldValue;
    }

    public String getResourceName() {
        return resourceName;
    }

    public String getFieldName() {
        return fieldName;
    }

    public Object getFieldValue() {
        return fieldValue;
    }
}
```

- 实现 Cors 跨域配置：

```java
package com.jpa.springjpademo.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**").allowedOrigins("*").allowedMethods("GET", "HEAD", "POST", "PUT", "DELETE", "OPTIONS")
                .allowCredentials(true).maxAge(3600).allowedHeaders("*");
    }
}
```

- 实现 Swagger 文档配置：

```java
package com.jpa.springjpademo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.service.Contact;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

@Configuration
@EnableSwagger2
public class SwaggerConfig {

    @Bean
    public Docket productApi() {
        return new Docket(DocumentationType.SWAGGER_2).select()
                .apis(RequestHandlerSelectors.basePackage("com.jpa.springjpademo")).apis(RequestHandlerSelectors.any())
                .paths(PathSelectors.any()).build().apiInfo(apiInfo());
    }

    private ApiInfo apiInfo() {
        Contact contact = new Contact("111hunter", "https://111hunter.github.io", "");
        return new ApiInfoBuilder().title("Spring Boot Restful Api Demo").description("Spring Boot Restful Api Demo")
                .version("0.0.1").license("Apache 2.0").licenseUrl("http://www.apache.org/licenses/LICENSE-2.0")
                .contact(contact).build();
    }

}
```

编写完后端代码，启动服务： ` $ mvn spring-boot:run `

浏览器打开 http://localhost:8034/swagger-ui.html

## 前端界面

### I. 初始化配置

先安装好 Vue/cli 工具: `$ npm install -g @vue/cli`

安装完成后生成 Vue 项目: `$ vue create blog-frontend`

输入此命令后，我们将看到一个简短的提示。选择 manually select features 选项（手动选择特性）。然后按空格表示选择，我们选择Babel、Router 和 Linter/Formatter。后面选项一路回车就好。

我们使用 bootstrap 库定义基本的 css 样式。在 public 文件夹的 index.html 中加入以下代码：

```html
  <link rel="stylesheet" href="https://cdn.staticfile.org/twitter-bootstrap/4.3.1/css/bootstrap.min.css">
  <script src="https://cdn.staticfile.org/jquery/3.2.1/jquery.min.js"></script>
  <script src="https://cdn.staticfile.org/popper.js/1.15.0/umd/popper.min.js"></script>
  <script src="https://cdn.staticfile.org/twitter-bootstrap/4.3.1/js/bootstrap.min.js"></script>
```

安装 axios 处理 http 请求，这是一种基于 Promise 的浏览器 HTTP 客户端：

`$ yarn add axios `

在 src 目录中新建文件夹 utils, 新建文件 helper.js, 对接后端接口:

```js
export const server = {

    baseURL: 'http://localhost:8034/api/notes'

}
```

### II. 创建页面组件

在 component 中新建文件夹 post, 新建三个文件：Create.vue, Edit.vue, Post.vue

- 新增帖子组件 Create.vue

```html
<template>
  <div>
    <div class="col-md-12 form-wrapper">
      <h2>Create Post</h2>
      <form id="create-post-form" @submit.prevent="createPost">
        <div class="form-group col-md-12">
          <label for="title">Title</label>
          <input type="text" id="title" v-model="title" name="title" class="form-control"
            placeholder="Enter title"/>
        </div>
        <div class="form-group col-md-12">
          <label for="description">Description</label>
          <input type="text" id="description" v-model="description" name="description" 
          class="form-control" placeholder="Enter Description"/>
        </div>
        <div class="form-group col-md-12">
          <label for="content">Write Content</label>
          <textarea id="content" cols="30" rows="5" v-model="content" class="form-control"></textarea>
        </div>
        <div class="form-group col-md-12">
          <label for="author">Author</label>
          <input type="text" id="author" v-model="author" name="author" class="form-control" />
        </div>
        <div class="form-group col-md-12">
          <button class="btn btn-success" type="submit">Create Post</button>
        </div>
      </form>
    </div>
  </div>
</template>
```

这是 Create.vue 组件 script 标签中的内容

```js
import axios from "axios";
import { server } from "../../utils/helper";
import router from "../../router";
export default {
  data() {
    return {
      title: "",
      description: "",
      content: "",
      author: "111hunter"
    };
  },
  methods: {
    createPost() {
      let postData = {
        title: this.title,
        description: this.description,
        content: this.content,
        author: this.author
      };
      this.__submitToServer(postData);
    },
    __submitToServer(data) {
      axios.post(`${server.baseURL}/create`, data).then(data => {
        router.push({ name: "home" });
      });
    }
  }
};
```

- 修改帖子组件 Edit.vue

```html
<template>
  <div>
    <h4 class="text-center mt-20">
      <small>
        <button class="btn btn-info" v-on:click="navigate()">View All Posts</button>
      </small>
    </h4>
    <div class="col-md-12 form-wrapper">
      <h2>Edit Post</h2>
      <form id="edit-post-form" @submit.prevent="editPost">
        <div class="form-group col-md-12">
          <label for="title">Title</label>
          <input type="text" id="title" v-model="post.title" name="title" 
          class="form-control" placeholder="Enter title"/>
        </div>
        <div class="form-group col-md-12">
          <label for="description">Description</label>
          <input type="text" id="description" v-model="post.description" name="description" class="form-control"placeholder="Enter Description"/>
        </div>
        <div class="form-group col-md-12">
          <label for="content">Write Content</label>
          <textarea id="content" cols="30" rows="5" v-model="post.content" class="form-control"></textarea>
        </div>
        <div class="form-group col-md-12">
          <label for="author">Author</label>
          <input type="text" id="author" v-model="post.author" name="author" class="form-control" />
        </div>

        <div class="form-group col-md-12">
          <button class="btn btn-warning" type="submit">Edit Post</button>
        </div>
      </form>
    </div>
  </div>
</template>
```

这是 Edit.vue 组件 script 标签中的内容

```js
import { server } from "../../utils/helper";
import axios from "axios";
import router from "../../router";
export default {
  data() {
    return {
      id: 0,
      post: {}
    };
  },
  created() {
    this.id = this.$route.params.id;
    this.getPost();
  },
  methods: {
    editPost() {
      let postData = {
        title: this.post.title,
        description: this.post.description,
        content: this.post.content,
        author: this.post.author
      };

      axios
        .put(`${server.baseURL}/update/?notes_id=${this.id}`, postData)
        .then(data => {
          router.push({ name: "home" });
        });
    },
    getPost() {
      axios
        .get(`${server.baseURL}/${this.id}`)
        .then(data => (this.post = data.data));
    },
    navigate() {
      router.go(-1);
    }
  }
};
```

- 帖子详情组件 Post.vue

```html
<template>
  <div class="text-center">
    <div class="col-sm-12">
      <h4 style="margin-top: 30px;">
        <small>
          <button class="btn btn-info" v-on:click="navigate()">View All Posts</button>
        </small>
      </h4>
      <hr />
      <h2>{{ post.title }}</h2>
      <h5>
        <span class="glyphicon glyphicon-time"></span>
        Post by {{post.author}}, {{new Date(post.updateTime).toLocaleDateString()}}.
      </h5>
      <p>{{ post.content }}</p>
    </div>
  </div>
</template>
```

这是 Post.vue 组件 script 标签中的内容

```js
import { server } from "../../utils/helper";
import axios from "axios";
import router from "../../router";
export default {
  data() {
    return {
      id: 0,
      post: {}
    };
  },
  created() {
    this.id = this.$route.params.id;
    this.getPost();
  },
  methods: {
    getPost() {
      axios
        .get(`${server.baseURL}/${this.id}`)
        .then(data => (this.post = data.data));
    },
    navigate() {
      router.go(-1);
    }
  }
};
```

- 展示帖子组件 Home.vue

修改 src/views 文件中的 Home.vue 组件为以下代码：

```html
<template>
  <div>
    <div class="text-center">
      <h1>Notes</h1>
      <p>The blog built with Spring Boot and Vue.js</p>
      <div v-if="posts.length === 0">
        <h2 class="text-warning">No post found at the moment</h2>
      </div>
    </div>

    <div class="row">
      <div class="col-md-4" v-for="post in posts" :key="post.notes_id">
        <div class="card mb-4 shadow">
          <div class="card-body">
            <h2 class="card-img-top text-info">{{ post.title }}</h2>
            <p class="card-text text-truncate">{{ post.content }}</p>
            <div class="d-flex justify-content-center align-items-center">
              <div class="btn-group" style="margin-bottom: 1rem;">
                <router-link :to="{name: 'Post', params: {id: post.id}}"
                  class="btn btn-sm btn-outline-info">View Post</router-link>
                <router-link :to="{name: 'Edit', params: {id: post.id}}"
                  class="btn btn-sm btn-outline-warning">Edit Post</router-link>
                <button class="btn btn-sm btn-outline-danger"
                  v-on:click="deletePost(post.id)">Delete Post</button>
              </div>
            </div>

            <div class="card-footer">
              <small class="text-muted">Posted on: {{ new Date(post.updateTime).toLocaleDateString()}}</small>
              <br />
              <small class="text-muted">by: {{ post.author}}</small>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="container">
      <!-- 按钮：用于打开模态框 -->
      <button v-if="this.posts.length" class="btn btn-warning" data-toggle="modal"
        data-target="#myModal">Delete All Post</button>
      <!-- 模态框 -->
      <div class="modal" id="myModal">
        <div class="modal-dialog">
          <div class="modal-content">
            <div class="modal-header" style="border-bottom: 0">
              <button type="button" class="close" data-dismiss="modal">&times;</button>
            </div>
            <div class="modal-body">Are you sure to delete all posts?</div>
            <div class="modal-footer" style="border-top: 0">
              <button class="btn btn-info" data-dismiss="modal">Quit</button>
              <button v-on:click="deleteAllPost()" class="btn btn-danger" data-dismiss="modal">Sure</button>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
```

这是 Home.vue 组件 script 标签中的内容

```js
import { server } from "@/utils/helper";
import axios from "axios";

export default {
  data() {
    return {
      posts: []
    };
  },
  created() {
    this.fetchPosts();
  },
  methods: {
    fetchPosts() {
      axios.get(`${server.baseURL}/all`).then(data => {
        this.posts = data.data;
      });
    },
    deletePost(id) {
      axios.delete(`${server.baseURL}/delete/?notes_id=${id}`).then(data => {
        window.location.reload();
      });
    },
    deleteAllPost() {
      axios.delete(`${server.baseURL}/delete/all`).then(data => {
        window.location.reload();
      });
    }
  }
};
```

### III. 搭建路由

在根组件 App.vue 中修改链接:

```
<router-link to="/about">About</router-link>
```

将 about 改为 create, 链接到 Create 组件:

```
<router-link to="/Create">Create</router-link>
```

最后将 router/index.js 改为以下代码：

```js
import Vue from 'vue'
import Router from 'vue-router'
import HomeComponent from '@/views/Home';
import EditComponent from '@/components/post/Edit';
import CreateComponent from '@/components/post/Create';
import PostComponent from '@/components/post/Post';

Vue.use(Router)

export default new Router({
  mode: 'history',
  base: process.env.BASE_URL,
  routes: [
    { path: '/', redirect: { name: 'home' } },
    { path: '/home', name: 'home', component: HomeComponent },
    { path: '/create', name: 'Create', component: CreateComponent },
    { path: '/edit/:id', name: 'Edit', component: EditComponent },
    { path: '/post/:id', name: 'Post', component: PostComponent }
  ]
});
```

编写完前端代码，启动服务： `$ npm run serve `

浏览器打开 http://localhost:8080

## 成果展示

pc端页面展示：

![pc端页面展示](/img/vue-pc.png "pc端页面展示")

移动端页面展示：

![移动端页面展示](/img/vue-mobile.png "移动端页面展示")

附：[源码地址](https://github.com/111hunter/notes-app)
