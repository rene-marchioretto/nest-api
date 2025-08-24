# NestJS Best Practices Guide

This document outlines best practices for developing with NestJS framework and organizing your components effectively.

## Table of Contents

- [Project Structure](#project-structure)
- [Module Organization](#module-organization)
- [Controllers](#controllers)
- [Services](#services)
- [DTOs and Validation](#dtos-and-validation)
- [Error Handling](#error-handling)
- [Database Integration](#database-integration)
- [Testing](#testing)
- [Security](#security)
- [Performance](#performance)
- [Code Style](#code-style)

## Project Structure

### Recommended Folder Structure

```
src/
├── app.module.ts
├── main.ts
├── common/
│   ├── decorators/
│   ├── filters/
│   ├── guards/
│   ├── interceptors/
│   ├── middleware/
│   └── pipes/
├── config/
│   ├── database.config.ts
│   └── app.config.ts
├── modules/
│   ├── auth/
│   │   ├── auth.module.ts
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── dto/
│   │   ├── guards/
│   │   └── strategies/
│   ├── users/
│   │   ├── users.module.ts
│   │   ├── users.controller.ts
│   │   ├── users.service.ts
│   │   ├── dto/
│   │   └── entities/
│   └── shared/
└── utils/
```

### Key Principles

- **Feature-based organization**: Group files by feature/domain, not by file type
- **Barrel exports**: Use `index.ts` files to create clean import paths
- **Separation of concerns**: Keep business logic in services, HTTP handling in controllers

## Module Organization

### 1. Root Module (AppModule)

```typescript
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      validationSchema: configValidationSchema,
    }),
    DatabaseModule,
    AuthModule,
    UsersModule,
  ],
  controllers: [],
  providers: [],
})
export class AppModule {}
```

### 2. Feature Modules

```typescript
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService], // Export services that other modules need
})
export class UsersModule {}
```

### Best Practices:

- **Keep modules focused**: Each module should handle one domain/feature
- **Use forFeature()**: For database entities and other feature-specific configurations
- **Export strategically**: Only export what other modules actually need
- **Lazy loading**: Consider lazy loading for large modules

## Controllers

### 1. Structure and Responsibilities

```typescript
@Controller('users')
@ApiTags('Users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  @ApiOperation({ summary: 'Get all users' })
  @ApiResponse({ status: 200, description: 'Users retrieved successfully' })
  async findAll(@Query() query: GetUsersDto): Promise<User[]> {
    return this.usersService.findAll(query);
  }

  @Post()
  @UsePipes(new ValidationPipe({ transform: true }))
  @ApiOperation({ summary: 'Create a new user' })
  async create(@Body() createUserDto: CreateUserDto): Promise<User> {
    return this.usersService.create(createUserDto);
  }
}
```

### Best Practices:

- **Single responsibility**: Controllers should only handle HTTP requests/responses
- **Use DTOs**: Always validate input with Data Transfer Objects
- **API documentation**: Use Swagger decorators for API documentation
- **HTTP status codes**: Use appropriate HTTP status codes
- **Route versioning**: Use versioning for API evolution

### 2. Route Design

```typescript
// ✅ Good - RESTful design
@Get(':id')
@Put(':id')
@Delete(':id')

// ✅ Good - Nested resources
@Get(':userId/posts')
@Post(':userId/posts')

// ❌ Avoid - Non-RESTful endpoints
@Get('getUserById/:id')
@Post('createNewUser')
```

## Services

### 1. Business Logic Layer

```typescript
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
    private readonly logger: Logger,
  ) {}

  async findAll(query: GetUsersDto): Promise<User[]> {
    try {
      const { page = 1, limit = 10, search } = query;

      const queryBuilder = this.userRepository.createQueryBuilder('user');

      if (search) {
        queryBuilder.where('user.name ILIKE :search', {
          search: `%${search}%`,
        });
      }

      return await queryBuilder
        .skip((page - 1) * limit)
        .take(limit)
        .getMany();
    } catch (error) {
      this.logger.error('Failed to fetch users', error.stack);
      throw new InternalServerErrorException('Failed to fetch users');
    }
  }
}
```

### Best Practices:

- **Single responsibility**: Each service should handle one business domain
- **Dependency injection**: Use constructor injection for dependencies
- **Error handling**: Properly handle and log errors
- **Transaction management**: Use database transactions for data consistency
- **Logging**: Add meaningful logs for debugging and monitoring

### 2. Service Composition

```typescript
@Injectable()
export class OrderService {
  constructor(
    private readonly userService: UserService,
    private readonly productService: ProductService,
    private readonly emailService: EmailService,
  ) {}

  async createOrder(createOrderDto: CreateOrderDto): Promise<Order> {
    // Compose multiple services to handle business logic
    const user = await this.userService.findById(createOrderDto.userId);
    const products = await this.productService.findByIds(
      createOrderDto.productIds,
    );

    // Business logic here...

    await this.emailService.sendOrderConfirmation(order);
    return order;
  }
}
```

## DTOs and Validation

### 1. Input Validation

```typescript
import { IsEmail, IsString, MinLength, IsOptional } from 'class-validator';
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class CreateUserDto {
  @ApiProperty({ description: 'User email address' })
  @IsEmail()
  email: string;

  @ApiProperty({ description: 'User full name' })
  @IsString()
  @MinLength(2)
  name: string;

  @ApiProperty({ description: 'User password', minLength: 8 })
  @IsString()
  @MinLength(8)
  password: string;

  @ApiPropertyOptional({ description: 'User phone number' })
  @IsOptional()
  @IsString()
  phone?: string;
}
```

### 2. Response DTOs

```typescript
export class UserResponseDto {
  @ApiProperty()
  id: number;

  @ApiProperty()
  email: string;

  @ApiProperty()
  name: string;

  @ApiProperty()
  createdAt: Date;

  // Never expose sensitive data like passwords
}
```

### Best Practices:

- **Separate DTOs**: Different DTOs for create, update, and response
- **Validation decorators**: Use class-validator for input validation
- **API documentation**: Document all DTO properties with Swagger
- **Transformation**: Use class-transformer for data transformation
- **Never expose sensitive data**: Passwords, tokens, etc.

## Error Handling

### 1. Custom Exception Filters

```typescript
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    const errorResponse = {
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      method: request.method,
      message: exception.message,
    };

    response.status(status).json(errorResponse);
  }
}
```

### 2. Custom Exceptions

```typescript
export class UserNotFoundException extends NotFoundException {
  constructor(userId: number) {
    super(`User with ID ${userId} not found`);
  }
}

export class EmailAlreadyExistsException extends ConflictException {
  constructor(email: string) {
    super(`User with email ${email} already exists`);
  }
}
```

### Best Practices:

- **Consistent error format**: Use filters for consistent error responses
- **Meaningful messages**: Provide clear, actionable error messages
- **Proper HTTP codes**: Use appropriate HTTP status codes
- **Log errors**: Always log errors for debugging
- **Don't expose internals**: Never expose stack traces to clients

## Database Integration

### 1. Repository Pattern

```typescript
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  async findByEmail(email: string): Promise<User | null> {
    return this.userRepository.findOne({ where: { email } });
  }

  async create(userData: CreateUserDto): Promise<User> {
    const user = this.userRepository.create(userData);
    return this.userRepository.save(user);
  }
}
```

### 2. Transactions

```typescript
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
    private readonly dataSource: DataSource,
  ) {}

  async createUserWithProfile(userData: CreateUserDto): Promise<User> {
    return this.dataSource.transaction(async (manager) => {
      const user = manager.create(User, userData);
      const savedUser = await manager.save(user);

      const profile = manager.create(Profile, { userId: savedUser.id });
      await manager.save(profile);

      return savedUser;
    });
  }
}
```

### Best Practices:

- **Use repositories**: Leverage TypeORM repositories for database operations
- **Transactions**: Use transactions for data consistency
- **Query optimization**: Use query builders for complex queries
- **Connection pooling**: Configure proper connection pooling
- **Migrations**: Use migrations for database schema changes

## Testing

### 1. Unit Tests

```typescript
describe('UsersService', () => {
  let service: UsersService;
  let repository: Repository<User>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: getRepositoryToken(User),
          useValue: {
            find: jest.fn(),
            findOne: jest.fn(),
            save: jest.fn(),
            create: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    repository = module.get<Repository<User>>(getRepositoryToken(User));
  });

  it('should find all users', async () => {
    const users = [{ id: 1, name: 'John' }];
    jest.spyOn(repository, 'find').mockResolvedValue(users);

    const result = await service.findAll();
    expect(result).toEqual(users);
  });
});
```

### 2. Integration Tests

```typescript
describe('UsersController (e2e)', () => {
  let app: INestApplication;

  beforeEach(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  it('/users (GET)', () => {
    return request(app.getHttpServer())
      .get('/users')
      .expect(200)
      .expect((res) => {
        expect(Array.isArray(res.body)).toBe(true);
      });
  });
});
```

### Best Practices:

- **Test pyramid**: More unit tests, fewer integration tests
- **Mock dependencies**: Mock external dependencies in unit tests
- **Test database**: Use a separate test database for integration tests
- **Test coverage**: Aim for high test coverage (80%+)
- **Test naming**: Use descriptive test names

## Security

### 1. Authentication & Authorization

```typescript
@Controller('users')
@UseGuards(JwtAuthGuard)
export class UsersController {
  @Get('profile')
  @UseGuards(RolesGuard)
  @Roles('admin', 'user')
  getProfile(@Req() req) {
    return req.user;
  }
}
```

### 2. Input Sanitization

```typescript
@Post()
@UsePipes(new ValidationPipe({
  transform: true,
  whitelist: true, // Strip unknown properties
  forbidNonWhitelisted: true, // Throw error for unknown properties
}))
async create(@Body() createUserDto: CreateUserDto) {
  return this.usersService.create(createUserDto);
}
```

### Best Practices:

- **Authentication**: Implement proper JWT authentication
- **Authorization**: Use guards and roles for authorization
- **Input validation**: Validate and sanitize all inputs
- **Rate limiting**: Implement rate limiting to prevent abuse
- **HTTPS**: Always use HTTPS in production
- **Environment variables**: Store secrets in environment variables

## Performance

### 1. Caching

```typescript
@Injectable()
export class UsersService {
  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  @CacheKey('users_all')
  @CacheTTL(300) // 5 minutes
  async findAll(): Promise<User[]> {
    return this.userRepository.find();
  }
}
```

### 2. Database Optimization

```typescript
// ✅ Good - Use query builder for complex queries
async findUsersWithPosts(): Promise<User[]> {
  return this.userRepository
    .createQueryBuilder('user')
    .leftJoinAndSelect('user.posts', 'post')
    .where('user.isActive = :isActive', { isActive: true })
    .getMany();
}

// ✅ Good - Use pagination
async findAll(page: number, limit: number): Promise<User[]> {
  return this.userRepository.find({
    skip: (page - 1) * limit,
    take: limit,
  });
}
```

### Best Practices:

- **Caching**: Implement caching for frequently accessed data
- **Query optimization**: Use efficient database queries
- **Pagination**: Always paginate large datasets
- **Compression**: Enable gzip compression
- **Connection pooling**: Configure database connection pooling

## Code Style

### 1. Naming Conventions

```typescript
// ✅ Good naming
class UsersController {}
class UserService {}
interface CreateUserDto {}
enum UserRole {}

// File names
users.controller.ts;
user.service.ts;
create - user.dto.ts;
```

### 2. Code Organization

```typescript
// ✅ Good - Consistent import order
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';

import { User } from './entities/user.entity';
import { CreateUserDto } from './dto/create-user.dto';

@Injectable()
export class UsersService {
  // Constructor first
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  // Public methods first
  async create(createUserDto: CreateUserDto): Promise<User> {
    return this.userRepository.save(createUserDto);
  }

  // Private methods last
  private validateUser(user: User): boolean {
    return !!user.email;
  }
}
```

### Best Practices:

- **Consistent naming**: Use consistent naming conventions
- **Import organization**: Organize imports logically
- **Code formatting**: Use Prettier for consistent formatting
- **ESLint rules**: Configure ESLint for code quality
- **Type safety**: Use TypeScript features for type safety

## Conclusion

Following these best practices will help you build maintainable, scalable, and robust NestJS applications. Remember to:

1. **Keep it simple**: Start simple and refactor as needed
2. **Follow SOLID principles**: Write clean, maintainable code
3. **Test thoroughly**: Write comprehensive tests
4. **Document everything**: Keep documentation up to date
5. **Stay updated**: Keep up with NestJS updates and community best practices

For more information, refer to the [official NestJS documentation](https://docs.nestjs.com/).
