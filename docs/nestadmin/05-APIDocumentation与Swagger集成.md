# API 文档与 Swagger 集成指南

基于 nest-admin 项目的 API 文档最佳实践

## 📚 目录

1. [Swagger 概述](#swagger-概述)
2. [配置 Swagger](#配置-swagger)
3. [集成 Swagger](#集成-swagger)
4. [API 文档的使用](#api-文档的使用)
5. [最佳实践](#最佳实践)

## 💡 Swagger 概述

Swagger（现在称为 OpenAPI Specification）是一套用于描述 RESTful API 的工具和规范。它提供了一种标准化的方式来定义 API 的结构、端点、请求参数、响应模型和认证方式。

**为什么使用 Swagger？**

*   **自动生成文档**：根据代码中的定义自动生成交互式 API 文档，省去了手动编写文档的繁琐工作。
*   **易于维护**：随着代码的更新，API 文档也能保持同步，减少了文档过时的问题。
*   **前后端协作**：为前端开发人员提供清晰的 API 接口定义，方便他们进行接口调用和联调。
*   **接口测试**：文档界面提供了在线测试功能，可以直接发送请求并查看响应。
*   **标准化**：遵循 OpenAPI 规范，使得 API 描述具有通用性和可读性。

在 `nest-admin` 项目中，我们利用 Swagger 来自动生成和管理后台管理系统的 API 文档，极大地提高了开发效率和接口管理能力。

## ⚙️ 配置 Swagger

Swagger 的基本配置信息位于 `src/config/swagger.config.ts` 文件中。通过这些配置项，你可以控制 Swagger 的启用状态、访问路径和服务器 URL。

```typescript
// src/config/swagger.config.ts
import { ConfigType, registerAs } from '@nestjs/config'

import { env, envBoolean } from '~/global/env'

export const swaggerRegToken = 'swagger'

export const SwaggerConfig = registerAs(swaggerRegToken, () => ({
  enable: envBoolean('SWAGGER_ENABLE'), // 是否启用 Swagger 文档，通过环境变量 SWAGGER_ENABLE 控制
  path: env('SWAGGER_PATH'), // Swagger 文档的访问路径，通过环境变量 SWAGGER_PATH 控制
  serverUrl: env('SWAGGER_SERVER_URL', env('APP_BASE_URL')), // Swagger 服务器的 URL，默认为应用基础 URL
}))

export type ISwaggerConfig = ConfigType<typeof SwaggerConfig>
```

*   `enable`：一个布尔值，用于控制 Swagger 文档是否启用。在生产环境中通常会禁用，以避免暴露 API 细节。
*   `path`：定义了 Swagger UI 的访问路径。例如，如果设置为 `docs`，那么你就可以通过 `http://localhost:port/docs` 来访问文档。
*   `serverUrl`：指定 Swagger 服务器的 URL。在开发环境中通常是 `http://localhost:port`，在生产环境中则是部署的域名。

这些配置项都可以通过 `.env` 文件中的环境变量进行设置，方便不同环境下的灵活配置。

## 🔌 集成 Swagger

`nest-admin` 项目在 `src/setup-swagger.ts` 文件中集成了 Swagger。这个文件负责配置 Swagger 文档的内容和行为。

```typescript
// src/setup-swagger.ts
import { INestApplication, Logger } from '@nestjs/common'
import { ConfigService } from '@nestjs/config'
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger'

import { API_SECURITY_AUTH } from './common/decorators/swagger.decorator'
import { CommonEntity } from './common/entity/common.entity'
import { ResOp, TreeResult } from './common/model/response.model'
import { ConfigKeyPaths, IAppConfig, ISwaggerConfig } from './config'
import { Pagination } from './helper/paginate/pagination'

export function setupSwagger(
  app: INestApplication,
  configService: ConfigService<ConfigKeyPaths>,
) {
  const { name, globalPrefix } = configService.get<IAppConfig>('app')!
  const { enable, path, serverUrl } = configService.get<ISwaggerConfig>('swagger')!

  if (!enable) // 如果 Swagger 未启用，则直接返回
    return

  const swaggerPath = `${serverUrl}/${path}` // 构建 Swagger UI 的完整访问路径

  const documentBuilder = new DocumentBuilder()
    .setTitle(name) // 设置 API 文档的标题，取自应用名称
    .setDescription(`
🔷 **Base URL**: \`${serverUrl}/${globalPrefix}\` <br>
🧾 **Swagger JSON**: [查看文档 JSON](${swaggerPath}/json)

📌 [nest-admin](https://github.com/buqiyuan/nest-admin) 后台管理系统 API 文档. 在线 demo [vue3-antdv-admin.pages.dev](https://vue3-antdv-admin.pages.dev/)
    `) // 设置 API 文档的描述，包含基础 URL 和 JSON 文档链接
    .setVersion('1.0') // 设置 API 版本
    .addServer(`${serverUrl}/${globalPrefix}`, 'Base URL') // 添加 API 服务器的 URL

  // 添加 JWT 认证安全定义
  documentBuilder.addSecurity(API_SECURITY_AUTH, {
    description: '输入令牌（Enter the token）', // 安全描述
    type: 'http', // 安全类型为 HTTP
    scheme: 'bearer', // 认证方案为 Bearer
    bearerFormat: 'JWT', // Bearer 格式为 JWT
  })

  // 创建 Swagger 文档
  const document = SwaggerModule.createDocument(app, documentBuilder.build(), {
    ignoreGlobalPrefix: true, // 忽略全局前缀，使 Swagger 文档路径不受影响
    extraModels: [CommonEntity, ResOp, Pagination, TreeResult], // 额外添加的实体和模型，用于在文档中显示它们的结构
  })

  // 设置 Swagger UI
  SwaggerModule.setup(path, app, document, {
    swaggerOptions: {
      persistAuthorization: true, // 保持登录状态，避免刷新后重新输入 token
    },
    jsonDocumentUrl: `/${path}/json`, // Swagger JSON 文档的访问路径
  })

  return () => {
  // 启动日志
    const logger = new Logger('SwaggerModule')
    logger.log(`Swagger UI: ${swaggerPath}`)
    logger.log(`Swagger JSON: ${swaggerPath}/json`)
  }
}
```

**核心步骤解析：**

1.  **`setupSwagger` 函数**：这个函数接收 NestJS 应用实例 (`app`) 和配置服务 (`configService`) 作为参数，负责 Swagger 的初始化和配置。
2.  **`DocumentBuilder`**：这是 Swagger 文档的构建器。它允许你设置文档的标题、描述、版本、服务器 URL 等元数据。例如，`setTitle(name)` 会将应用名称设置为文档标题。
3.  **JWT 认证配置**：通过 `documentBuilder.addSecurity(API_SECURITY_AUTH, {...})`，我们为 Swagger 文档添加了 JWT 认证的支持。这意味着你可以在 Swagger UI 中输入 JWT 令牌，以便测试需要认证的 API 接口。`API_SECURITY_AUTH` 是一个常量，定义在 `common/decorators/swagger.decorator.ts` 中。
    ```typescript
    // common/decorators/swagger.decorator.ts
    import { applyDecorators } from '@nestjs/common'
    import { ApiBearerAuth, ApiOperation, ApiTags } from '@nestjs/swagger'

    export const API_SECURITY_AUTH = 'Authorization'

    export function ApiSecurityAuth(summary: string) {
      return applyDecorators(
        ApiBearerAuth(API_SECURITY_AUTH),
        ApiOperation({ summary }),
      )
    }
    ```
    这个装饰器使得在 API 接口上添加 `@ApiSecurityAuth('获取用户列表')` 即可在 Swagger 文档中显示认证图标，并且要求输入 JWT Token。
4.  **`SwaggerModule.createDocument`**：这个方法根据 `DocumentBuilder` 的配置和 NestJS 应用的路由结构，生成 Swagger 文档对象。`extraModels` 参数用于包含那些在 API 响应中可能作为复杂类型出现的实体或模型，确保它们在文档中得到正确描述。
5.  **`SwaggerModule.setup`**：这个方法将生成的 Swagger 文档与 NestJS 应用集成，并在指定的 `path`（例如 `/docs`）上暴露 Swagger UI 界面。
    *   `persistAuthorization: true`：这是一个非常实用的选项，它会在浏览器中保存你输入的认证令牌，这样在刷新页面或切换接口时，就不需要重新输入令牌了。
6.  **日志输出**：在 `setupSwagger` 函数的最后，会打印出 Swagger UI 和 JSON 文档的访问地址，方便开发者直接访问。

## 📋 API 文档的使用

当项目启动并启用了 Swagger 后，你可以通过浏览器访问配置的 `serverUrl` 和 `path` 来查看 API 文档。

例如，如果你的应用运行在 `http://localhost:7001`，并且 Swagger `path` 配置为 `docs`，那么你就可以通过 `http://localhost:7001/docs` 访问 Swagger UI。

**如何在 Swagger UI 中测试 API 接口：**

1.  **认证**：对于需要认证的接口（例如带有锁图标的接口），你需要点击页面顶部的"Authorize"按钮，输入你的 JWT Token。通常，你的 JWT Token 会以 `Bearer your-token-string` 的形式存在。只需输入 `your-token-string` 即可。
2.  **选择接口**：在文档页面中，你可以看到所有分组的 API 接口。点击一个接口可以展开其详细信息。
3.  **尝试请求**：展开接口后，点击"Try it out"按钮。你可以在这里修改请求参数（如果接口有的话）。
4.  **执行请求**：点击"Execute"按钮发送请求。
5.  **查看响应**：在请求发送后，你将看到请求的 URL、响应状态码、响应头和响应体。这有助于你调试接口。

## 🚀 最佳实践

为了更好地利用 Swagger 并生成高质量的 API 文档，可以遵循以下最佳实践：

### 1. 使用 Swagger 装饰器

NestJS 提供了丰富的 `@nestjs/swagger` 装饰器，可以用来描述 API 接口的各个方面，例如：

*   `@ApiTags('用户管理')`：为 Controller 定义标签，将相关的接口分组显示在文档中。
*   `@ApiOperation({ summary: '获取用户列表' })`：为每个 API 方法添加操作摘要。
*   `@ApiResponse({ status: 200, description: '成功获取用户列表' })`：描述 API 的响应。
*   `@ApiQuery({ name: 'username', description: '用户名' })`：描述查询参数。
*   `@ApiParam({ name: 'id', description: '用户ID' })`：描述路径参数。
*   `@ApiBody({ type: CreateUserDto })`：描述请求体。
*   `@ApiBearerAuth()`：表示该接口需要 JWT 认证。
*   `@ApiExtraModels()`：如果你的响应包含嵌套模型，可以使用此装饰器帮助 Swagger 正确解析。

例如：

```typescript
// modules/user/user.controller.ts
import { Body, Controller, Get, Post, Query } from '@nestjs/common'
import { ApiBearerAuth, ApiOperation, ApiQuery, ApiTags } from '@nestjs/swagger'

import { ApiSecurityAuth } from '~/common/decorators/swagger.decorator'
import { PaginationDto } from '~/common/dto/pagination.dto'
import { CreateUserDto, QueryUserDto } from './user.dto'
import { UserEntity } from './user.entity'
import { UserService } from './user.service'

@ApiTags('用户管理') // 为该控制器添加标签
@ApiBearerAuth('Authorization') // 表示该控制器下的所有接口都需要 JWT 认证
@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get('list')
  @ApiSecurityAuth('获取用户列表（分页）') // 为单个接口添加操作摘要和安全认证
  @ApiQuery({ name: 'username', required: false, description: '用户名' })
  @ApiQuery({ name: 'status', required: false, description: '用户状态' })
  async getUsers(@Query() query: QueryUserDto & PaginationDto) {
    return this.userService.findUsersWithPagination(query.page, query.limit)
  }

  @Post()
  @ApiSecurityAuth('创建用户')
  @ApiOperation({ summary: '创建新用户' }) // 可以与 ApiSecurityAuth 结合使用
  async createUser(@Body() userData: CreateUserDto): Promise<UserEntity> {
    return this.userService.createUser(userData)
  }
}
```

### 2. 保持文档与代码同步

由于 Swagger 是根据代码自动生成的，因此保持代码的清晰和规范是生成优质文档的关键。每次修改 API 接口时，务必更新相应的 Swagger 装饰器，确保文档的准确性。

### 3. 使用 `extraModels`

如果你的 API 响应中包含一些复杂的数据结构（例如，自定义的响应模型、分页结果等），但是这些模型没有直接被 `@ApiProperty()` 装饰器引用，那么它们可能不会在 Swagger 文档中显示。此时，你需要在 `SwaggerModule.createDocument` 的 `extraModels` 中手动添加这些模型，例如：

```typescript
// src/setup-swagger.ts
// ... existing code ...
    extraModels: [CommonEntity, ResOp, Pagination, TreeResult],
// ... existing code ...
```

通过遵循这些实践，你可以确保 `nest-admin` 项目的 API 文档始终是最新的、准确的，并且对其他开发人员非常友好。
