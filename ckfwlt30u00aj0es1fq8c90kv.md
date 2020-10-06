## Nest JS: Caching Service Methods with Custom Decorators

After searching the internet for a long time and checking with the Nest JS community, I realised that there is no out of the box solution available to provide Caching/Memoization capabilities to Service level methods in Nest JS. So I decided to build one.

## **Problem Statement**

Nest JS provides decorators out of the box that can be plugged into the Controller Methods and hence caching can be achieved for a particular route. What about the situation where we want to cache the output of a service method that makes an API call or a DB query.

## ** Custom Cache Decorator as a Solution**

Since every method can be cached, a custom decorator is the easiest solution to tackle this problem.
So let's take a look at a sample service file.


```
import { HttpService, Injectable } from '@nestjs/common';
import { ConfigProperties } from '@nestjs/configuration';
import { Observable, of } from 'rxjs';
import { catchError, map } from 'rxjs/operators';
import { AxiosError } from 'axios';
import { logger } from 'http-logging-core';

@Injectable()
export class SearchService {
  public constructor(private readonly configProperties: ConfigProperties, private readonly httpService: HttpService) {}

  public getResults(query: string): Observable<Array<any>> {
    const baseApi = this.configProperties.get('endpoints.search');
    const searchApi = `${baseApi}$?q={query}`;
    const searchResults$ = this.httpService.get(searchApi).pipe(
      map((item) => {
        return item.data.routes;
      }),
      catchError((err: AxiosError) => {
        logger.error(err);

        // Error handling Logic skipped for brevity. 
        return of([]);
      }),
    );

    return searchResults$;
  }
}

```

So now this method `getResults` makes an HTTP request. It can do anything, like a DB query using ORM as well. This makes it a perfect example of a caching use case.

Ideally what we want is minimal code change and achieve Caching with just a Decorator Annotation.

So the only logical code change we should do is - 

```
// Add a caching decorator with a fixed Prefix
@Cache({ key: 'SEARCH_RESULTS' })
  public getResults(query: string): Observable<Array<any>> {
```

The Decorator takes an optional `key` parameter in case you want to filter all your Cache Keys using a `startsWith` operator.

Now we need a Cache Service which will actually be called in the decorator to store the response of the method. In our example, I am using Redis for Caching.
Let's have a look at the `redis-service.ts` file

```
import { Inject, Injectable } from '@nestjs/common';
import { RedisService } from 'nestjs-redis';

@Injectable()
export class RedisCacheService {
  public client;
  public constructor(@Inject(RedisService) private readonly redisService: RedisService) {}

  public getClient() {
    this.client = this.redisService.getClient();
  }

  public async set(key: string, value: any) {
    const stringifiedValue = JSON.stringify(value);

    if (!this.client) {
      this.getClient();
    }

    await this.client.set(key, stringifiedValue);
  }

  public async get(key: string) {
    if (!this.client) {
      this.getClient();
    }

    const data: any = await this.client.get(key);

    if (!data) {
      return [];
    }

    return JSON.parse(data);
  }
}
```

The reason I chose `'nestjs-redis'` npm module is it's built on top of `ioredis` module which provides a lot of functionalities like support for `Redis-Sentinel`.

**What Redis Sentinel is? **

*It's a highly available Redis Cluster Setup with a Primary and Replica Concept*

Now we have our normal Service and Redis Service ready, lets dive into the `cache-decorator.ts`

```
import { Inject } from '@nestjs/common';
import { logger } from 'http-logging-core';
import { from, merge, of } from 'rxjs';
import { catchError, filter, tap, first, map } from 'rxjs/operators';
import { RedisCacheService } from '../services/redis.service';

export function Cache({ key }: { key?: string }) {
  const redisInjection = Inject(RedisCacheService);

  return function (target: Record<string, any>, _, descriptor: PropertyDescriptor) {
    redisInjection(target, 'redisService');

    const method = descriptor.value;

    descriptor.value = function (...args: Array<any>) {
      const entryKey = `${key}[${args.map((res) => JSON.stringify(res)).join(',')}]`;

      logger.info(entryKey);

      const { redisService } = this;

      const cacheCall$ = from(redisService.get(entryKey)).pipe(
        map((cacheData: Array<string> | string | object) => {
          return cacheData ?? [];
        }),
        catchError((e) => {
          logger.info('Cache call errored', e);

          // Should always return null only then Origin Call will be made in case of error
          return of(null);
        }),
      );

      const originCall$ = method.apply(this, args).pipe(
        tap((originValue) => {
          redisService.set(entryKey, originValue);
        }),
        catchError((e) => {
          logger.info('Origin call errored', e);

          return of(null);
        }),
      );

      // call .toPromise to ensure cache is always updated after origin fetch
      const originWithUpdate$ = from(originCall$.toPromise());

      return merge(cacheCall$, originWithUpdate$).pipe(filter(Boolean), first());
    };
  };
}

```

Well that looks like a lot of code, so let's go Line by Line

```
const redisInjection = Inject(RedisCacheService);
```

Since Nest JS works on the concept of Dependency Injection, we want access to the Redis Service instance inside our decorator. But the way Decorator's work, they only get access to those instances which are available to the original method on which they are applied.

This means, in our `SearchService.ts` we will have to inject `RedisService.ts`, and only then it will pass the Redis Service instance to the decorator. Now for sure, we don't want this as the Decorator Responsible for Caching should encapsulate all logic around caching and not the caller itself. We don't want to inject Redis Service in all our Services, just because the Decorator can access it easily from `this` keyword.

So what we do is Dynamic Dependency Injection. Nest JS provides us with this API called `Inject` which we can use to create a DI at runtime.
This will at runtime inject the Redis Service in the context of execution.

```
const method = descriptor.value;
```

We get the reference to the original method from the `descriptor.value` parameter.

```
descriptor.value = function (...args: Array<any>) {
```

Well, this is the famous concept of **Monkey Patching**. We store the original reference 1st in a temporary variable and then overwrite the original variable with new functionality having both old and new mappings. 

Now in the `descriptor.value`, we do the actual Logic which will get invoked as if the service method which we annotated with decorator would have.

```
const entryKey = `${key}[${args.map((res) => JSON.stringify(res)).join(',')}]`;
```

We want to create an entry key based on the prefix provided at Decorator Annotation level and parameters passed to the original function so that every different call to the same function can be cached.

```
const { redisService } = this;
```

Since in the top we used `Inject` to DI our service, now its available inside the `this` context.

We now declare 2 Observables, `cacheCall$`, and `originCall$` and can have custom logic to further decide how we handle both the data stream.
My use case was, `originCall$` should always update the cache. So I am using the `tap` operator to update the cache.

```
return merge(cacheCall$, originWithUpdate$).pipe(filter(Boolean), first());
```

My use case was to always call both Cache Service and Origin Service and return the fastest data with eventual consistency. Hence I was using a `merge` operator. You can definitely choose a completely different approach to how you want to tackle cache and origin calls.

Now we have the decorator ready to be annotated to any function in our Services.

To complete the final piece of code, let's dive into the `app.module.ts`. I spent a good time figuring this out as well from Redis Configuration perspective.

```
...
import { RedisModule } from 'nestjs-redis';
import { SearchController } from './controllers/search.controller';
import { RedisCacheService } from './shared/services/redis.service';
import { SearchService } from './services/search.service';
...

@Module({
  providers: [
    SearchService
    RedisCacheService,
  ],
  controllers: [
    SearchController,
  ],
  imports: [
    ConfigModule,
    RedisModule.forRootAsync({
      imports: [ConfigModule],
      useFactory(configProperties: ConfigProperties) {
        let redisOptions;

        // Generic Error Handling logic
        const onClientReady = async (client) => {
          client.on('error', () => {
            logger.error('Connection to Redis Failed');
          });
        };

        // As i am using Redis Sentinel in Production, `NODE_ENV` gets set during build time with a value 
        if (process.env.NODE_ENV) {
          const sentinels = configProperties
            .get('redis.host')
            .split(' ')
            .map((endpoint: string) => ({
              host: endpoint,
              port: configProperties.get('redis.port'),
            }));

          redisOptions = {
            sentinels,
            name: configProperties.get('redis.name') || 'mymaster',
            showFriendlyErrorStack: true,
            onClientReady,
          };
        // For local development I am using a single instance Redis Docker Container
        } else {
          redisOptions = {
            host: configProperties.get('redis.host'),
            port: configProperties.get('redis.port'),
            onClientReady,
          };
        }

        return redisOptions;
      },
      inject: [ConfigProperties],
    }),
    HttpModule.registerAsync({
      imports: [ConfigModule],
      useFactory: (configProperties: ConfigProperties) => ({
        timeout: configProperties.get('http.timeout'),
      }),
      inject: [ConfigProperties],
    }),
  ],
})
export class ApplicationModule {}
```

**With this now we have the complete setup ready to run our application with external caching using Redis and Redis Sentinel which supports Service Method Caching using custom decorators.**

Let me know what you think in the comments below. 