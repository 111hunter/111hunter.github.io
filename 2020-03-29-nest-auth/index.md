# NestJS + Passport 实现鉴权认证

鉴权(authentication)是指验证用户是否拥有访问系统的权利。传统的鉴权是通过密码来验证的。这种方式的前提是，每个获得密码的用户都已经被授权。

## 建立用户表，密码散列

要实现鉴权认证，首先需要一张 user 表。上一次我们用 NestJS 和 Typeorm 做了最基本的 crud 操作, 这次我们用 NestJS 和 node 中最流行的身份验证库 Passport 来完成鉴权认证。为了方便，我们直接沿用上次的代码库。

创建 user module: `$ nest g mo user `

然后在 user 文件夹新建 user.entity.ts, 其中我们做了密码散列:

```ts
import { BeforeInsert, Column, Entity, PrimaryGeneratedColumn } from 'typeorm'
import * as bcrypt from 'bcryptjs';

@Entity('user')
export class UserEntity {
    @PrimaryGeneratedColumn()
    id: number

    @Column({ length: 20 })
    username: string

    @Column({ length: 255 })
    password: string

    @BeforeInsert()
    async hashPassword() {
        this.password = await bcrypt.hash(this.password, 10);
    }
}
```

在 user.module.ts 中注册 user 表：` TypeOrmModule.forFeature([UserEntity]) `

上一次我们直接在 module 中写了数据库连接配置，其实更常见的做法是写一个数据库配置文件。可以用环境变量设置数据库连接，这是 typeorm 数据库连接配置的[参考地址](https://typeorm.io/#/using-ormconfig/using-environment-variables)。在文件夹建立一个 .env 文件：

```ts
# App
JWT_SECRET = 'ThisIsASecretKey'

# Database
TYPEORM_CONNECTION = mysql
TYPEORM_HOST = localhost
TYPEORM_USERNAME = root
TYPEORM_PASSWORD = 123456
TYPEORM_DATABASE = test
TYPEORM_PORT = 3306
TYPEORM_SYNCHRONIZE = true
TYPEORM_LOGGING = true
TYPEORM_ENTITIES = dist/**/*.entity.js
```

其中写了数据库配置和自定义的 jwt 密匙，关于如何生成 jwt 格式的字符串, 可以看[这篇文章](https://www.footmark.info/programming-language/nodejs/nodejs-restful-webapi-oauth2-jwt/#fm-chapter-2-1), 本文只讲如何使用它来做用户登录验证。

在 app.module.ts 的imports数组中修改数据库为注册：` TypeOrmModule.forRoot() `，然后写入 UserModule。测试一下我们的数据库连接情况：` $ npm run start `, 控制台打印了 sql 语句，说明我们的连接配置是对的。查看数据库会发现新增加了 user 表。

在 user 文件夹新建 user.dto.ts：

```ts
import { IsString } from 'class-validator'
import { ApiProperty } from '@nestjs/swagger';

export class UserDto {

    @ApiProperty()
    @IsString()
    readonly username: string;

    @ApiProperty()
    @IsString()
    readonly password: string;
}
```

然后创建 user service：` $ nest g s user `，注意在 createUser 方法中一定要先 **实例化** user, 再返回创建的对象。否则 user.entity.ts 中的 @BeforeInsert() 装饰的方法不会执行，密码就不会取散列后的值。

```ts
import { Injectable } from '@nestjs/common'
import { InjectRepository } from '@nestjs/typeorm'
import { UserEntity } from './user.entity'
import { Repository } from 'typeorm'
import { UserDto } from './user.dto'

@Injectable()
export class UserService {
    constructor(
        @InjectRepository(UserEntity)
        private readonly userRepository: Repository<UserEntity>,
    ) { }

    async createUser(userDto: UserDto) {
        // const user = Object.assign(new UserEntity(), userDto)
        const user = this.userRepository.create(userDto);
        return await this.userRepository.save(user);
    }

    async findUsername(username: string) {
        return this.userRepository.findOne({ where: { username } })
    }

    async findAll(): Promise<UserEntity[]> {
        return await this.userRepository.find();
    }
}
```

## 实现本地认证策略

实现本地认证策略需要先安装以下依赖：

```bash
yarn add @nestjs/passport passport passport-local
yarn add -D @types/passport-local
```

说明一下，这一步不是必须的。其实本地认证就是做用户名和密码的核对，我们自己去实现也不算麻烦。但是为了和 NestJS 官网教程保持一致，我们也这样做。
创建 auth module: ` $ nest g mo auth `，在 auth 目录下创建一个 local.strategy.ts 文件：

```ts
import { Injectable, UnauthorizedException } from '@nestjs/common'
import { PassportStrategy } from '@nestjs/passport'
import { Strategy } from 'passport-local'
import { AuthService } from './auth.service'


@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
    constructor(private readonly authService: AuthService) {
        super()
    }

    async validate(username: string, password: string) {
        const user = await this.authService.validateUser(username, password)
        if (!user) {
            throw new UnauthorizedException()
        }
        return user
    }
}

```

使用 @nestjs/passport ，你需要继承 PassportStrategy 类来配置 passport 策略。通过调用子类中的 super() 方法传递策略选项，通过在子类中实现 validate() 方法，可以提供verify回调。Passport 定义的 **所有策略** 都是将validate() 方法执行的结果作为 user 属性存储在当前  **HTTP Request 对象** 上,你也可以自定义此属性的名称。上面文件中的 validateUser 方法需要在 auth.service.ts 自己实现，因为框架不清楚你定义的密码散列方式。
```ts
//auth.service.ts

...

    async validateUser(username: string, pass: string): Promise<any> {
        const user = await this.userService.findUsername(username);
        console.log('-----------Login-----------')
        if (user && bcrypt.compareSync(pass, user.password)) {
            return user;
        }
        return null;
    }
```
## 实现注册登录功能

创建 auth controller: ` $ nest g co auth `，路由功能：

```ts
import { UserDto } from './../user/user.dto';
import { Body, Controller, Get, Post, UseGuards, Res, Request } from '@nestjs/common'
import { AuthGuard } from '@nestjs/passport'
import { ApiTags, ApiBearerAuth } from '@nestjs/swagger'
import { AuthService } from 'src/auth/auth.service'

@ApiBearerAuth()
@ApiTags('Auth')
@Controller('auth')
export class AuthController {
    constructor(
        private readonly authService: AuthService,
    ) { }

    @UseGuards(AuthGuard('jwt'))
    @Get('users')
    async findAll(@Request() req): Promise<any[]> {
        console.log('--------------Auth--Success---------------')
        console.log(req.user);
        return await this.authService.findAll();
    }

    @Post('signUp')
    async register(@Body() req: UserDto, @Res() res) {
        const result = await this.authService.register(req);
        res.status(result.statusCode).send(result);
    }

    @UseGuards(AuthGuard('local'))
    @Post('signIn')
    async login(@Body() @Request() req: UserDto, @Res() res) {
        console.log('----------Login--Success-----------')
        console.log(req);
        const result = await this.authService.login(req);
        res.status(result.statusCode).send(result);
    }
}
```

注意其中的 @UseGuards(AuthGuard('local')) 装饰器，因为我们写了 local.strategy.ts文件，其中继承了 PassportStrategy 类，并实现了 validate 方法。@nestjs/passport 就会为我们实现一个 AuthGuard，我们直接在需要验证的路由前使用就好。@UseGuards(AuthGuard('jwt')) 是我们接下来要讲的 JWT 认证策略。

再补充完整 auth.service.ts 文件：

```ts
// auth.service.ts

import { BadRequestException, Injectable, Body, Request } from '@nestjs/common'
import { UserService } from '../user/user.service'
import { JwtService } from '@nestjs/jwt'
import * as bcrypt from 'bcryptjs';

@Injectable()
export class AuthService {

    constructor(
        private readonly userService: UserService,
        private readonly jwtService: JwtService,
    ) { }

    async findAll(): Promise<any[]> {
        return await this.userService.findAll();
    }

    async validateUser(username: string, pass: string): Promise<any> {
        const user = await this.userService.findUsername(username);
        console.log('-----------Login-----------')
        if (user && bcrypt.compareSync(pass, user.password)) {
            return user;
        }
        return null;
    }

    async register(user: any) {
        let userData: any;
        userData = await this.userService.findUsername(user.username);
        if (userData) {
            return { statusCode: 400, message: 'This username aleady exists' };
        }
        await this.userService.createUser(user);
        userData = await this.userService.findUsername(user.username);
        return {
            username: userData.username,
            statusCode: 201
        };
    }

    async login(user: any) {
        return this.userService.findUsername(user.username).then((userData) => {
            const Token = this.createToken(userData);
            return {
                username: userData.username,
                access_token: Token,
                statusCode: 200
            }
        });
    }

    createToken(user: any) {
        const payload = { username: user.username, sub: user.id };
        return this.jwtService.sign(payload);
    }
}

```
## 实现 JWT 认证策略

实现了用户注册登录后，我们需要保护 API，限制有的路由地址需要用户登录后才能访问，有的路由地址需要管理员登录后才能访问。我们这里只实现需要普通用户登录后才能访问的路由。

什么是Token？

- 前后端分离模式下，Token 是我们验证用户登录的常用方式。Token是服务端生成的一串字符串，以作客户端进行请求的一个令牌，当第一次登录后，服务器会生成一个Token并将此Token，返回给客户端，以后客户端只需带上这个Token前来请求数据即可，无需再次带上用户名和密码。

为什么要使用Token？

- 在很多项目案例中，需要实现账户的功能，客户端所有的功能都基于用户已登陆的前提下才可以使用。这就要求每次客户端向服务器请求数据时都要验证账户是否正确，如果正确则按正常方式返回数据，如果错误则进行拦截并返回错误信息。但是当客户端频繁向服务器请求数据的话，每次服务器都要频繁地查询数据库。而Token正是为了 ***减轻服务器的压力，减少频繁的查询数据库，使服务器更加健壮***。并取代传统使用session的方法来进行验证。

在 Nest.js 中使用 jwt(json web token), 我们需要安装以下依赖：

```bash
yarn add @nestjs/jwt passport-jwt
yarn add -D @types/passport-jwt
```

我们在 auth.service.ts 中已经实现了生成 jwt 字符串的方法，在用户登录路由中就会调用，并返回 jwt 字符串：

```ts
    createToken(user: any) {
        const payload = { username: user.username, sub: user.id };
        return this.jwtService.sign(payload);
    }
```
> 注意：上面 sign 的参数 payload 是可逆加密的，拿到 token 后是可以解密成明文内容的，所以这部分不要放敏感信息。

我们已经创建了 jwt 字符串作为请求令牌，那么服务端如何根据 jwt 字符串的内容，找到用户信息？
我们就需要实现 jwt 认证策略，在 auth 文件夹下新建 jwt.strategy.ts 文件：

```ts
import { Injectable } from '@nestjs/common'
import { PassportStrategy } from '@nestjs/passport'
import { ExtractJwt, Strategy } from 'passport-jwt'
import { JWT_SECRET } from 'config'

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
    constructor() {
        super({
            jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
            ignoreExpiration: false,
            secretOrKey: JWT_SECRET,
        })
    }

    async validate(payload: any) {
        return { userId: payload.sub, username: payload.username };
    }
}
```

解释一下，对于 JWT 策略，Passport 首先验证 JWT 的签名并解码为 JSON 格式内容。仅在 @nestjs/passport 模块验证令牌有效后，才调用 validate() 方法。该方法将解码后的 JSON 作为其单个参数继续传递。否则。将阻止请求，抛出 401 Unauthorized 的异常。

现在来看我们的auth.controller.ts，可以将 validate() 返回值输出到控制台：

```ts
// auth.controller.ts

    @UseGuards(AuthGuard('jwt'))
    @Get('users')
    async findAll(@Request() req): Promise<any[]> {
        console.log('--------------Auth--Success---------------')
        console.log(req.user);
        return await this.authService.findAll();
    }
```

最后这是我们的 auth.module.ts，其中注册了 jwt 字符串过期时间，我们在 auth.service.ts 中注入了 UserService，记得导入 UserModule。

```ts
// auth.module.ts
 
@Module({
    imports: [
        PassportModule,
        JwtModule.register({
            secret: JWT_SECRET,
            signOptions: { expiresIn: '3600s' },
        }),
        UserModule,
    ],
    controllers: [AuthController],
    providers: [AuthService, LocalStrategy, JwtStrategy],
    exports: [AuthService],
})
export class AuthModule { }

```

启动项目：` $ npm run start:dev `，打开 http://localhost:3000/docs 在 swagger 文档模型中测试我们的 api。先 signUp, 然后 signIn, 登录成功返回 access_token，点击那个锁符号，将 access_token 的值粘贴过去，就能通过认证了。

附：[源码地址](https://github.com/111hunter/nest-crud)

**参考资料**

- [Nest.JS 官方文档](https://docs.nestjs.com/techniques/authentication)
- [Typeorm 官方文档](https://typeorm.io/#/using-ormconfig/using-environment-variables)
- [Node.js JWT 范例](https://www.footmark.info/programming-language/nodejs/nodejs-restful-webapi-oauth2-jwt/#fm-chapter-2-1)





