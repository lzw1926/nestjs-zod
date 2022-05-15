<p align="center">
  <img src="logo.png" align="center" alt="NestJS + Zod logo" />
  <h1></h1>
  <p align="center">
    A superior validation solution for your NestJS application
  </p>
</p>
<br/>
<p align="center">
  <a href="https://github.com/risenforces/nestjs-zod/actions?query=branch%3Amain">
    <img src="https://github.com/risenforces/nestjs-zod/actions/workflows/test-and-build.yml/badge.svg?event=push&branch=main" alt="nestjs-zod CI Status" />
  </a>
  <a href="https://opensource.org/licenses/MIT" rel="nofollow">
    <img src="https://img.shields.io/github/license/risenforces/nestjs-zod" alt="License">
  </a>
  <a href="https://www.npmjs.com/package/nestjs-zod" rel="nofollow">
    <img src="https://img.shields.io/npm/dw/nestjs-zod.svg" alt="npm">
  </a>
  <a href="https://www.npmjs.com/package/nestjs-zod" rel="nofollow">
    <img src="https://img.shields.io/github/stars/risenforces/nestjs-zod" alt="stars">
  </a>
</p>

## Core library features

- `createZodDto` - create DTO classes from Zod schemas
- `ZodValidationPipe` - validate `body` / `query` / `params` using Zod DTOs
- `ZodGuard` - guard routes by validating `body` / `query` / `params`  
  (it can be useful when you want to do that before other guards)
- `UseZodGuard` - alias for `@UseGuards(new ZodGuard(source, schema))`
- `ZodValidationException` - BadRequestException extended with Zod errors
- `zodToOpenAPI` - create OpenAPI declarations from Zod schemas
- `@nestjs/swagger` integration, but you need to apply patch
  - `nestjs-zod` generates highly accurate Swagger Schema
- Extended Zod schemas for NestJS (work in progress)
- Customization - you freely can replace `ZodValidationException`

## Installation

```
yarn add nestjs-zod
```

Included dependencies:
- `zod` -  `3.14.3`

Peer dependencies:
- `@nestjs/common` -  `^8.0.0`
- `@nestjs/core` -  `^8.0.0`
- `@nestjs/swagger` -  `^5.0.0` (optional)

## Navigation

- [Writing Zod schemas](#writing-zod-schemas)
- [Creating DTO from Zod schema](#creating-dto-from-zod-schema)
  - [Using DTO](#using-dto)
- [Using ZodValidationPipe](#using-zodvalidationpipe)
  - [Globally](#globally-recommended)
  - [Locally](#locally)
  - [Creating custom validation pipe](#creating-custom-validation-pipe)
- [Using ZodGuard](#using-zodguard)
  - [Creating custom guard](#creating-custom-guard)
- [Validation Exceptions](#validation-exceptions)
- [Extended Zod](#extended-zod)
  - [ZodDateString](#zoddatestring)
  - [Extended Zod Errors](#extended-zod-errors)
  - [Working with errors on the client side](#working-with-errors-on-the-client-side)
- [OpenAPI (Swagger) support](#openapi-swagger-support)
  - [Setup](#setup)
  - [Writing more Swagger-compatible schemas](#writing-more-swagger-compatible-schemas)
  - [Using zodToOpenAPI](#using-zodtoopenapi)

## Writing Zod schemas

Extended Zod and Swagger integration are bound to the internal API, so even the patch updates can cause errors.

For that reason, `nestjs-zod` uses specific `zod` version inside and re-exports it under `/z` scope:

```ts
import { z, ZodString, ZodError } from 'nestjs-zod/z'

const CredentialsSchema = z.schema({
  username: z.string(),
  password: z.string(),
})
```

Zod's classes and types are re-exported too, but under `/z` scope for more clarity:

```ts
import { ZodString, ZodError, ZodIssue } from 'nestjs-zod/z' 
```

## Creating DTO from Zod schema

```ts
import { createZodDto } from 'nestjs-zod'
import { z } from 'nestjs-zod/z'

const CredentialsSchema = z.schema({
  username: z.string(),
  password: z.string(),
})

// class is required for using DTO as a type
class CredentialsDto extends createZodDto(CredentialsSchema) {}
```

### Using DTO

DTO does two things:
- Provides a schema for `ZodValidationPipe`
- Provides a type from Zod schema for you

```ts
@Controller('auth')
class AuthController {
  // with global ZodValidationPipe (recommended)
  async signIn(@Body() credentials: CredentialsDto) {}
  async signIn(@Param() signInParams: SignInParamsDto) {}
  async signIn(@Query() signInQuery: SignInQueryDto) {}

  // with route-level ZodValidationPipe
  @UsePipes(ZodValidationPipe)
  async signIn(@Body() credentials: CredentialsDto) {}
}

// with controller-level ZodValidationPipe
@UsePipes(ZodValidationPipe)
@Controller('auth')
class AuthController {
  async signIn(@Body() credentials: CredentialsDto) {}
}
```

## Using ZodValidationPipe

The validation pipe uses your Zod schema to parse data from parameter decorator.

When the data is invalid - it throws [ZodValidationException](#validation-exceptions).

### Globally (recommended)

```ts
import { ZodValidationPipe } from 'nestjs-zod'

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ZodValidationPipe,
    },
  ],
})
export class AppModule {}
```

### Locally

```ts
import { ZodValidationPipe } from 'nestjs-zod'

// controller-level
@UsePipes(ZodValidationPipe)
class MyController {}

class MyController {
  // route-level
  @UsePipes(ZodValidationPipe)
  async signIn() {}
}
```

### Creating custom validation pipe

```ts
import { createZodValidationPipe } from 'nestjs-zod'

const MyZodValidationPipe = createZodValidationPipe({
  // provide custom validation exception factory
  createValidationException: (error: ZodError) => new BadRequestException('Ooops')
})
```

## Using ZodGuard

Sometimes, we need to validate user input before specific Guards. We can't use Validation Pipe since NestJS Pipes are always executed after Guards.

The solution is `ZodGuard`. It works just like `ZodValidationPipe`, except for that is doesn't transform the input.

It has 2 syntax forms:
- `@UseGuards(new ZodGuard('body', CredentialsSchema))`
- `@UseZodGuard('body', CredentialsSchema)`

The first parameter is `Source`: `'body' | 'query' | 'params'`

When the data is invalid - it throws [ZodValidationException](#validation-exceptions).

```ts
import { ZodGuard } from 'nestjs-zod'

// controller-level
@UseZodGuard('body', CredentialsSchema)
class MyController {}

class MyController {
  // route-level
  @UseZodGuard('body', CredentialsSchema)
  async signIn() {}
}
```

### Creating custom guard

```ts
import { createZodGuard } from 'nestjs-zod'

const MyZodGuard = createZodGuard({
  // provide custom validation exception factory
  createValidationException: (error: ZodError) => new BadRequestException('Ooops')
})
```

## Validation Exceptions

The default server response on validation error looks like that:

```json
{
  "statusCode": 400,
  "message": "Validation failed",
  "errors": [
    {
      "code": "too_small",
      "minimum": 8,
      "type": "string",
      "inclusive": true,
      "message": "String must contain at least 8 character(s)",
      "path": ["password"]
    }
  ]
}
```

The reason of this structure is default `ZodValidationException`.

You can customize the exception by creating custom `nestjs-zod` entities using the factories:
- [Validation Pipe](#creating-custom-validation-pipe)
- [Guard](#creating-custom-guard)

You can create `ZodValidationException` manually by providing `ZodError`:

```ts
const exception = new ZodValidationException(error)
```

Also, `ZodValidationException` has an additional API for better usage in NestJS Exception Filters:

```ts
@Catch(ZodValidationException)
export class ZodValidationExceptionFilter implements ExceptionFilter {
  catch(exception: ZodValidationException) {
    exception.getZodError() // -> ZodError
  }
}
```

## Extended Zod

As you learned in [Writing Zod Schemas](#writing-zod-schemas) section, `nestjs-zod` provides a special version of Zod. It helps you to validate the user input more accurately by using our custom schemas and methods.

### ZodDateString

In HTTP, we always accept Dates as strings. But default Zod doesn't have methods to validate such type of strings. `ZodDateString` was created to address this issue.

```ts
// 1. Expect user input to be a "string" type
// 2. Expect user input to be a valid date (by using new Date)
z.dateString()

// Cast to Date instance
// (use it on end of the chain, but before "describe")
z.dateString().cast()

// Expect string in "full-date" format from RFC3339
z.dateString().format('date')

// [default format]
// Expect string in "date-time" format from RFC3339
z.dateString().format('date-time')

// Expect date to be the past
z.dateString().past()

// Expect date to be the future
z.dateString().future()

// Expect year to be greater or equal to 2000
z.dateString().minYear(2000)

// Expect year to be less or equal to 2025
z.dateString().maxYear(2025)

// Expect day to be a week day
z.dateString().weekDay()

// Expect year to be a weekend
z.dateString().weekend()
```

Valid `date` format examples:
- `2022-05-15`

Valid `date-time` format examples:
- `2022-05-02:08:33Z`
- `2022-05-02:08:33.000Z`
- `2022-05-02:08:33+00:00`
- `2022-05-02:08:33-00:00`
- `2022-05-02:08:33.000+00:00`

Errors:
- `invalid_date_string` - invalid date

- `invalid_date_string_format` - wrong format

  Payload:
  - `expected` - `'date' | 'date-time'`

- `invalid_date_string_direction` - not past/future

  Payload:
  - `expected` - `'past' | 'future'`

- `invalid_date_string_day` - not weekDay/weekend

  Payload:
  - `expected` - `'weekDay' | 'weekend'`

### Extended Zod Errors

Currently, we use `custom` error code due to some Zod limitations (`errorMap` priorities)

Therefore, the error details is located inside `params` property:

```json
{
  "code": "custom",
  "message": "Invalid date, expected it to be the past",
  "params": {
    "isNestJsZod": true,
    "code": "invalid_date_string_direction",

    // payload is always located here in a flat view
    "expected": "past"
  },
  "path": [
    "date"
  ]
}
```

### Working with errors on the client side

Optionally, you can install `nestjs-zod` on the client side.

The library provides you a `/frontend` scope, that can be used to detect custom NestJS Zod issues and process them the way you want. 

```ts
import { isNestJsZodIssue, NestJsZodIssue, ZodIssue } from 'nestjs-zod/frontend'

function mapToFormErrors(issues: ZodIssue[]) {
  for (const issue of issues) {
    if (isNestJsZodIssue(issue)) {
      // issue is NestJsZodIssue
    }
  }
}
```

## OpenAPI (Swagger) support

### Setup

Prerequisites:
- `@nestjs/swagger` with version `^5.0.0` installed

Apply a patch:

```ts
import { patchNestjsSwagger } from 'nestjs-zod'

patchNestjsSwagger()
```

Then follow the [Nest.js' Swagger Module Guide](https://docs.nestjs.com/openapi/introduction).

### Writing more Swagger-compatible schemas

Use `.describe()` method to add Swagger description:

```ts
import { z } from 'nestjs-zod/z'

const CredentialsSchema = z.schema({
  username: z.string().describe('This is an username'),
  password: z.string().describe('This is a password'),
})
```

### Using zodToOpenAPI

You can convert any Zod schema to an OpenAPI JSON object:

```ts
import { zodToOpenAPI } from 'nestjs-zod'
import { z } from 'nestjs-zod/z'

const SignUpSchema = z.object({
  username: z.string().min(8).max(20),
  password: z.string().min(8).max(20),
  sex: z
    .enum(['male', 'female', 'Apache Attack Helicopter'])
    .describe('We respect your gender choice'),
  social: z.record(z.string().url()),
  birthDate: z.dateString().past(),
})

const openapi = zodToOpenAPI(SignUpSchema)
```

The output will be the following:

```json
{
  "type": "object",
  "properties": {
    "username": {
      "type": "string",
      "minLength": 8,
      "maxLength": 20
    },
    "password": {
      "type": "string",
      "minLength": 8,
      "maxLength": 20
    },
    "sex": {
      "description": "We respect your gender choice",
      "type": "string",
      "enum": [
        "male",
        "female",
        "Apache Attack Helicopter"
      ]
    },
    "social": {
      "type": "object",
      "additionalProperties": {
        "type": "string",
        "format": "uri"
      }
    },
    "birthDate": {
      "type": "string",
      "format": "date-time"
    }
  },
  "required": [
    "username",
    "password",
    "sex",
    "social",
    "birthDate"
  ]
}
```

## Credits

- [zod-dto](https://github.com/kbkk/abitia/tree/master/packages/zod-dto)  
  `nestjs-zod` includes a lot of refactored code from `zod-dto`.

- [zod-nestjs](https://github.com/anatine/zod-plugins/tree/main/libs/zod-nestjs) and [zod-openapi](https://github.com/anatine/zod-plugins/tree/main/libs/zod-openapi)
  These libraries bring some new features compared to `zod-dto`.  
  `nestjs-zod` has used them too.