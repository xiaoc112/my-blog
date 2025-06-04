# NestJS 框架完整应用实践指南

基于 nest-admin 项目的 NestJS 最佳实践

## 📚 目录

1. [项目架构设计](#项目架构设计)
2. [模块系统应用](#模块系统应用)
3. [依赖注入最佳实践](#依赖注入最佳实践)
4. [中间件和拦截器](#中间件和拦截器)
5. [守卫和权限控制](#守卫和权限控制)
6. [异常处理](#异常处理)
7. [配置管理](#配置管理)
8. [生命周期管理](#生命周期管理)

## 🏗️ 项目架构设计

### 1. 整体架构图

```
src/
├── app.module.ts          # 根模块
├── main.ts               # 应用入口
├── modules/              # 业务模块
│   ├── auth/            # 认证模块
│   ├── user/            # 用户模块
│   ├── system/          # 系统管理
│   └── ...
├── shared/              # 共享模块
├── common/              # 公共组件
├── config/              # 配置文件
└── utils/               # 工具函数
```

### 2. 核心模块配置

```typescript
// app.module.ts - 根模块配置
@Module({
  imports: [
    // 全局配置模块
    ConfigModule.forRoot({
      isGlobal: true,
      expandVariables: true,
      envFilePath: ['.env.local', `.env.${process.env.NODE_ENV}`, '.env'],
      load: [...Object.values(config)],
    }),

    // CLS 上下文管理 (用于请求追踪)
    ClsModule.forRoot({
      global: true,
      interceptor: {
        mount: true,
        setup: (cls, context) => {
          const req = context.switchToHttp().getRequest()
          if (req.params?.id && req.body) {
            cls.set('operateId', Number.parseInt(req.params.id))
          }
        },
      },
    }),

    // 共享模块
    SharedModule,
    DatabaseModule,

    // 业务模块
    AuthModule,
    SystemModule,
    TasksModule.forRoot(),
    ToolsModule,
    // ... 其他模块
  ],
  providers: [
    // 全局过滤器
    { provide: APP_FILTER, useClass: AllExceptionsFilter },

    // 全局拦截器
    { provide: APP_INTERCEPTOR, useClass: ClassSerializerInterceptor },
    { provide: APP_INTERCEPTOR, useClass: TransformInterceptor },
    { provide: APP_INTERCEPTOR, useFactory: () => new TimeoutInterceptor(15 * 1000) },
    { provide: APP_INTERCEPTOR, useClass: IdempotenceInterceptor },

    // 全局守卫
    { provide: APP_GUARD, useClass: JwtAuthGuard },
    { provide: APP_GUARD, useClass: RbacGuard },
    { provide: APP_GUARD, useClass: ThrottlerGuard },
  ],
})
export class AppModule {}
```

## 🧩 模块系统应用

### 1. 业务模块设计

```typescript
// user.module.ts - 用户模块示例
@Module({
  imports: [
    TypeOrmModule.forFeature([UserEntity]),
    forwardRef(() => AuthModule),
  ],
  controllers: [UserController],
  providers: [UserService],
  exports: [UserService],
})
export class UserModule {}
```

### 2. 共享模块设计

```typescript
// shared.module.ts - 全局共享模块
@Global()
@Module({
  imports: [
    LoggerModule.forRoot(),
    HttpModule,
    ScheduleModule.forRoot(),
    ThrottlerModule.forRoot([{
      limit: 20,
      ttl: 60000,
    }]),
    EventEmitterModule.forRoot({
      wildcard: true,
      delimiter: '.',
      maxListeners: 20,
      ignoreErrors: false,
    }),
    RedisModule,
    MailerModule,
    HelperModule,
  ],
  exports: [HttpModule, MailerModule, RedisModule, HelperModule],
})
export class SharedModule {}
```

### 3. 动态模块

```typescript
// tasks.module.ts - 动态模块示例
@Module({})
export class TasksModule {
  static forRoot(): DynamicModule {
    return {
      module: TasksModule,
      imports: [
        BullModule.forRootAsync({
          imports: [ConfigModule],
          useFactory: (configService: ConfigService) => ({
            redis: {
              host: configService.get('redis.host'),
              port: configService.get('redis.port'),
            },
          }),
          inject: [ConfigService],
        }),
      ],
      providers: [TasksService],
      exports: [TasksService],
    }
  }
}
```

## 💉 依赖注入最佳实践

### 1. 服务注入

```typescript
// user.service.ts
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(UserEntity)
    private readonly userRepository: Repository<UserEntity>,
    private readonly configService: ConfigService,
    private readonly loggerService: LoggerService,
  ) {}

  async createUser(userData: CreateUserDto): Promise<UserEntity> {
    // 使用注入的依赖
    const config = this.configService.get('app')
    this.loggerService.log('Creating user', 'UserService')

    const user = this.userRepository.create(userData)
    return await this.userRepository.save(user)
  }
}
```

### 2. 自定义提供者

```typescript
// 工厂提供者
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (configService: ConfigService) => {
    const config = configService.get('database')
    return createConnection(config)
  },
  inject: [ConfigService],
}

// 值提供者
const configProvider = {
  provide: 'CONFIG',
  useValue: { timeout: 5000 },
}

// 类提供者
const loggerProvider = {
  provide: 'LOGGER',
  useClass: CustomLogger,
}
```

## 🛡️ 中间件和拦截器

### 1. 全局拦截器

```typescript
// transform.interceptor.ts - 响应数据转换
@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map(data => ({
        code: 200,
        message: 'success',
        data,
        timestamp: Date.now(),
      })),
    )
  }
}

// timeout.interceptor.ts - 请求超时处理
@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  constructor(private readonly timeout: number) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(this.timeout),
      catchError((err) => {
        if (err instanceof TimeoutError) {
          throw new RequestTimeoutException('请求超时')
        }
        throw err
      }),
    )
  }
}
```

### 2. 幂等性拦截器

```typescript
// idempotence.interceptor.ts - 防重复提交
@Injectable()
export class IdempotenceInterceptor implements NestInterceptor {
  async intercept(context: ExecutionContext, next: CallHandler): Promise<Observable<any>> {
    const request = context.switchToHttp().getRequest()
    const key = this.generateKey(request)

    // 检查是否已存在相同请求
    const exists = await this.cacheService.get(key)
    if (exists) {
      throw new BadRequestException('请勿重复提交')
    }

    // 设置缓存
    await this.cacheService.set(key, '1', 5) // 5秒内防重复

    return next.handle()
  }

  private generateKey(request: any): string {
    const { method, url, body, headers } = request
    return `idempotence:${method}:${url}:${JSON.stringify(body)}:${headers['user-id']}`
  }
}
```

## 🔐 守卫和权限控制

### 1. JWT 认证守卫

```typescript
// jwt-auth.guard.ts
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super()
  }

  canActivate(context: ExecutionContext): boolean | Promise<boolean> | Observable<boolean> {
    // 检查是否为公开接口
    const isPublic = this.reflector.getAllAndOverride<boolean>('isPublic', [
      context.getHandler(),
      context.getClass(),
    ])

    if (isPublic) {
      return true
    }

    return super.canActivate(context)
  }
}
```

### 2. RBAC 权限守卫

```typescript
// rbac.guard.ts - 基于角色的访问控制
@Injectable()
export class RbacGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private authService: AuthService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const requiredPermissions = this.reflector.getAllAndOverride<string[]>('permissions', [
      context.getHandler(),
      context.getClass(),
    ])

    if (!requiredPermissions) {
      return true
    }

    const request = context.switchToHttp().getRequest()
    const user = request.user

    // 检查用户权限
    const hasPermission = await this.authService.checkPermissions(user.id, requiredPermissions)

    if (!hasPermission) {
      throw new ForbiddenException('权限不足')
    }

    return true
  }
}
```

### 3. 权限装饰器

```typescript
// permissions.decorator.ts
export function RequirePermissions(...permissions: string[]) {
  return SetMetadata('permissions', permissions)
}

// 使用示例
@Controller('users')
export class UserController {
  @Get()
  @RequirePermissions('user:read')
  async getUsers() {
    return this.userService.findAll()
  }

  @Post()
  @RequirePermissions('user:create')
  async createUser(@Body() userData: CreateUserDto) {
    return this.userService.create(userData)
  }
}
```

## ⚠️ 异常处理

### 1. 全局异常过滤器

```typescript
// any-exception.filter.ts
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  constructor(private readonly logger: LoggerService) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp()
    const response = ctx.getResponse()
    const request = ctx.getRequest()

    let status = HttpStatus.INTERNAL_SERVER_ERROR
    let message = '服务器内部错误'

    if (exception instanceof HttpException) {
      status = exception.getStatus()
      const errorResponse = exception.getResponse()
      message = typeof errorResponse === 'string' ? errorResponse : errorResponse.message
    }

    // 记录错误日志
    this.logger.error(
      `${request.method} ${request.url}`,
      exception instanceof Error ? exception.stack : String(exception),
      'ExceptionFilter',
    )

    // 返回错误响应
    response.status(status).json({
      code: status,
      message,
      timestamp: new Date().toISOString(),
      path: request.url,
    })
  }
}
```

### 2. 自定义业务异常

```typescript
// 业务异常基类
export class BusinessException extends HttpException {
  constructor(message: string, code?: number) {
    super(message, code || HttpStatus.BAD_REQUEST)
  }
}

// 具体业务异常
export class UserNotFoundException extends BusinessException {
  constructor(userId: number) {
    super(`用户 ${userId} 不存在`)
  }
}

export class DuplicateUsernameException extends BusinessException {
  constructor(username: string) {
    super(`用户名 ${username} 已存在`)
  }
}
```

## ⚙️ 配置管理

### 1. 环境配置

```typescript
// config/app.config.ts
export const AppConfig = registerAs('app', () => ({
  name: env('APP_NAME', 'nest-admin'),
  port: envNumber('APP_PORT', 7001),
  globalPrefix: env('GLOBAL_PREFIX', 'api'),
  locale: env('APP_LOCALE', 'zh-CN'),

  // 安全配置
  cors: {
    enabled: envBoolean('CORS_ENABLED', true),
    origin: env('CORS_ORIGIN', '*'),
  },

  // JWT 配置
  jwt: {
    secret: env('JWT_SECRET'),
    expiresIn: env('JWT_EXPIRES_IN', '7d'),
  },
}))

// 使用配置
@Injectable()
export class AuthService {
  constructor(
    @Inject(AppConfig.KEY)
    private readonly appConfig: ConfigType<typeof AppConfig>,
  ) {}

  generateToken(payload: any): string {
    return this.jwtService.sign(payload, {
      secret: this.appConfig.jwt.secret,
      expiresIn: this.appConfig.jwt.expiresIn,
    })
  }
}
```

### 2. 配置验证

```typescript
// config/validation.schema.ts
import * as Joi from 'joi'

export const validationSchema = Joi.object({
  NODE_ENV: Joi.string().valid('development', 'production', 'test').default('development'),
  APP_PORT: Joi.number().default(7001),
  DB_HOST: Joi.string().required(),
  DB_PORT: Joi.number().default(3306),
  DB_USERNAME: Joi.string().required(),
  DB_PASSWORD: Joi.string().required(),
  DB_DATABASE: Joi.string().required(),
  REDIS_HOST: Joi.string().required(),
  REDIS_PORT: Joi.number().default(6379),
  JWT_SECRET: Joi.string().required(),
})

// 在 AppModule 中使用
ConfigModule.forRoot({
  validationSchema,
  validationOptions: {
    allowUnknown: true,
    abortEarly: true,
  },
})
```

## 🔄 生命周期管理

### 1. 应用生命周期

```typescript
// main.ts - 应用启动配置
async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    fastifyApp,
    {
      bufferLogs: true,
      snapshot: true,
    },
  )

  // 应用配置
  const configService = app.get(ConfigService)
  const { port, globalPrefix } = configService.get('app')

  // 启用跨域
  app.enableCors({
    origin: '*',
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization', 'Accept'],
  })

  // 全局前缀
  app.setGlobalPrefix(globalPrefix)

  // 静态资源
  app.useStaticAssets({ root: path.join(__dirname, '..', 'public') })

  // 优雅关闭
  !isDev && app.enableShutdownHooks()

  // 全局管道
  app.useGlobalPipes(
    new ValidationPipe({
      transform: true,
      whitelist: true,
      transformOptions: { enableImplicitConversion: true },
      errorHttpStatusCode: HttpStatus.UNPROCESSABLE_ENTITY,
      stopAtFirstError: true,
    }),
  )

  // WebSocket 适配器
  app.useWebSocketAdapter(new RedisIoAdapter(app))

  // 启动应用
  await app.listen(port, '0.0.0.0')
}
```

### 2. 模块生命周期钩子

```typescript
@Injectable()
export class UserService implements OnModuleInit, OnModuleDestroy {
  private intervalId: NodeJS.Timeout

  async onModuleInit() {
    // 模块初始化时执行
    console.log('UserService 初始化')

    // 启动定时任务
    this.intervalId = setInterval(() => {
      this.cleanupExpiredSessions()
    }, 60000)
  }

  async onModuleDestroy() {
    // 模块销毁时执行
    console.log('UserService 销毁')

    // 清理资源
    if (this.intervalId) {
      clearInterval(this.intervalId)
    }
  }

  private cleanupExpiredSessions() {
    // 清理过期会话
  }
}
```

## 🚀 最佳实践总结

### 1. 项目结构规范
- 按功能模块组织代码
- 使用统一的命名约定
- 分离业务逻辑和配置

### 2. 依赖注入原则
- 优先使用构造函数注入
- 避免循环依赖
- 合理使用作用域

### 3. 错误处理策略
- 统一异常处理机制
- 详细的错误日志记录
- 用户友好的错误信息

### 4. 性能优化
- 合理使用缓存
- 实现请求超时控制
- 避免内存泄漏

### 5. 安全考虑
- 实现完整的认证授权
- 输入验证和过滤
- 防止常见安全漏洞

这份指南涵盖了在 nest-admin 项目中 NestJS 框架的核心应用实践，帮助你更好地理解和使用 NestJS 构建企业级应用。
