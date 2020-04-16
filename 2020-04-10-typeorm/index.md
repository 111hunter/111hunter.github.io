# TypeORM 的基本使用



本文主要讲述如何用 typeorm 建表，建立一对一，一对多，多对多的关系，建立表的外连接。
<br>以及在 typeorm 做查询操作的两种常用方式：Find 选项 和 QueryBuilder。

## 建表

typeorm 建表时，将 @Entity() 装饰的 class 映射为数据表，entity 中 @PrimaryColumn() 装饰的属性作为表的主键, @PrimaryGeneratedColumn() 表示自动生成主键, @Column() 装饰属性作为表的属性。

```ts
@Entity()
export class Photo {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({length: 100})
  name: string;

  @Column("text")
  description: string;

  @Column()
  views: number;

  @Column()
  isPublished: boolean;
}
```
数据库中的列类型是根据你使用的属性类型推断的，例如: number 将被转换为 integer，string 将转换为 varchar，boolean 转换为 bool 等。下面我们从实际的例子出发探索如何用 typeorm 建一对一、一对多、多对多的关系。

## 一对一

用户 user 和用户档案 profile 是一对一关系，一个用户只有一份档案。

```ts
@Entity('users')
export class UserEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  username: string;

  @OneToOne(type => ProfileEntity, profile => profile.user)
  @JoinColumn()
  profile: ProfileEntity;
}
```

> 注：profile 是 ProfileEntity 类型的，在数据库中存储的类型却是 profile.id 的类型。

@OneToOne 中需要指明对方 entity 的类型，指明对方 entity 的外键。@JoinColumn 必须在且只在关系的一侧的外键上。

```ts
@Entity('profiles')
export class ProfileEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  gender: string;

  @Column()
  photo: string;
  @OneToOne(type => UserEntity, user => user.profile)
  user: UserEntity;
}
```

这将生成以下数据表：

```bash
+-------------+--------------+----------------------------+
|                          users                          |
+-------------+--------------+----------------------------+
| id          | int(11)      | PRIMARY KEY AUTO_INCREMENT |
| username    | varchar(255) |                            |
| profileId   | int(11)      | FOREIGN KEY                |
+-------------+--------------+----------------------------+

+-------------+--------------+----------------------------+
|                        profiles                         |
+-------------+--------------+----------------------------+
| id          | int(11)      | PRIMARY KEY AUTO_INCREMENT |
| gender      | varchar(255) |                            |
| photo       | varchar(255) |                            |
+-------------+--------------+----------------------------+
```
## 一对多

用户 user 与用户发布的文章 article 是一对多关系，一个用户可发布多篇文章。

```ts
@Entity('users')
export class UserEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  username: string;

  @OneToMany(type => ArticleEntity, article => article.author)
  articles: ArticleEntity[];
}
```

@OneToMany，@ManyToOne 中需要指明对方的 entity 类型，指明对方 entity 的外键。

```ts
@Entity('articles')
export class ArticleEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  title: string;

  @ManyToOne(type => UserEntity, user => user.articles)
  author: UserEntity;
}
```

typeorm 在处理 “一对多”关系时将“一”的主键作为“多”的外键 (即 @ManyToOne 装饰的属性)，建表时有最少的数据表操作代价，避免数据冗余，提高效率。这会生成以下表：

```bash
+-------------+--------------+----------------------------+
|                        articles                         |
+-------------+--------------+----------------------------+
| id          | int(11)      | PRIMARY KEY AUTO_INCREMENT |
| title       | varchar(255) |                            |
| authorId    | int(11)      |                            |
+-------------+--------------+----------------------------+

+-------------+--------------+----------------------------+
|                          users                          |
+-------------+--------------+----------------------------+
| id          | int(11)      | PRIMARY KEY AUTO_INCREMENT |
| username    | varchar(255) |                            |
+-------------+--------------+----------------------------+
```
## 多对多

用户 user 对文章 article 的喜欢 favorite 是多对多关系。一个用户可对多篇文章标记喜欢，一篇文章可被多个用户标记喜欢。

```ts
@Entity('users')
export class UserEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  username: string;

  @ManyToMany(type => ArticleEntity, article => article.favoritedBy)
  favorites: ArticleEntity[];
}
```

@OneToMany 中需要指明对方的 entity 类型，指明对方 entity 的外键。@JoinTable 必须在且只在关系的一侧的外键上。

```ts
@Entity('articles')
export class ArticleEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  title: string;

  @ManyToMany(type => UserEntity, user => user.favorites)
  @JoinTable()
  favoritedBy: UserEntity[];
}
```

typeorm 的处理方式是将多对多关系转化为两个一对多关系:

  - 用户 user 与 喜欢 favorites 一对多。
  - 文章 article 与被喜欢 favoritedBy 一对多。

多对多关系需要采用中间表的方式处理，这是为了避免笛卡尔积的出现。这会生成以下表：

```bash
+-------------+--------------+----------------------------+
|                          users                          |
+-------------+--------------+----------------------------+
| id          | int(11)      | PRIMARY KEY AUTO_INCREMENT |
| username    | varchar(255) |                            |
+-------------+--------------+----------------------------+

+-------------+--------------+----------------------------+
|                        articles                         |
+-------------+--------------+----------------------------+
| id          | int(11)      | PRIMARY KEY AUTO_INCREMENT |
| title       | varchar(255) |                            |
+-------------+--------------+----------------------------+

+-------------+--------------+----------------------------+
|              articles_favorited_by_users                |
+-------------+--------------+----------------------------+
| articlesId  | int(11)      | PRIMARY KEY FOREIGN KEY    |
| usersId     | int(11)      | PRIMARY KEY FOREIGN KEY    |
+-------------+--------------+----------------------------+
```

## 增删改查

创建好一对一，一对多，多对多的实体 entity 后，我们如何做增删改查呢？单个实体的 crud 可参考我的这一篇[文章](https://111hunter.github.io/post/nest-crud/)。而关联后的实体对象会作为该实体对象的一个属性, 直接对属性进行操作即可。如下是文章被用户喜欢的实现:

```ts
async favoriteArticle(slug: string, user: UserEntity) {
    const article = await this.articleRepo.findOne({ where: { slug }});
    article.favoritedBy.push(user);
    await article.save();
    return article;
}
```

crud 操作中查询操作是我们最常遇到的，下面讲如何查询，typeorm 支持两种查询方式：Find 选项 和 QueryBuilder。

### Find 选项

在 Nest.JS 中，对具体实体的管理（insert, update, delete, load 等)我们使用的是 Repository。对应的查找方法是：Repository.find(FindOptions)。
使用 find 查询只能获得一种类型的结果：entities。

find 选项的完整例子如下：

```ts
userRepository.find({
    select: ["username"],
    relations: ["profile", "article"，"article.favoritedBy"],
    where: {
        username: "Timber",
    },
    order: {
        id: "DESC"
    },
    skip: 5,
    take: 10,
    cache: true
});
```

直接使用 find 是不会查出关联的对象的，要查询的关联对象需要添加到 relations 数组中。
<br>除了 relations 以外，其他选项等同于原生 sql 操作, order 等同于 order by, skip 等同于 offset, take 等同于 limit, cache 是查询缓存。细节请参考 [Find 选项](https://typeorm.io/#/find-options)。

这种查询有个局限就是只能查询到关联对象的整个实体或主键。而不能 select 关联实体的其他属性。因此更复杂的查询我们需要使用 QueryBuilder。

### QueryBuilder

使用QueryBuilder 查询可以获得两种类型的结果：entities 或原始数据。
要获取entities，请使用getOne和getMany。
要获取原始数据，请使用getRawOne和getRawMany。
它能够很方便的帮我们构造出 sql 语句，addSelect() 可以获取关联对象上的其他属性。

```ts
if (query.author) {
    const article = await getRepository(ArticleEntity)
        .createQueryBuilder('article')
        .select("article.id", 'id')
        .addSelect('favoritedBy.username', 'name')
        .leftJoin('article.favoritedBy', 'favoritedBy')
        .where("favoritedBy.username = :name", { name: query.author })
        .getRawMany();
}
```

获取生成的 sql 语句可以在 getRawMany() 前获取 getSql() 或打印 printSql() 生成的sql语句。细节请参考 [Query Builder](https://typeorm.io/#/select-query-builder)。

**参考资料**

- [Typeorm 官方文档](https://typeorm.io/#/)

