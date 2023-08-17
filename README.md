# Redis Learning Project

This guide will walk you through using Redis in a Blazor project using C#. Redis is an in-memory data structure store that can be used as a cache and for other data storage and retrieval tasks.

## Prerequisites

Before you begin, make sure you have Docker installed on your system. You can download Docker from the official website: [Docker](https://www.docker.com/)

## 1. Start Redis Docker Container

Open a terminal and run the following command to start a Redis Docker container:

```bash
docker run --name Redis -p 6379:6379 -d redis
```

Explanation of the command:

- `--name Redis`: Assign the name "Redis" to the container.
- `-p 6379:6379`: Map port 6379 on the host to Redis port 6379. This port is used to access the Redis server.
- `-d redis`: Use the Redis Docker image.

## 2. Gain Control Over Redis Container Shell

To gain control over the Redis container shell, use the following command:

```bash
docker exec -it Redis sh
```

## 3. Connect to Redis Command Line

Once inside the container shell, use the following command to connect to the Redis command line:

```bash
redis-cli
```

After connecting to the Redis command line, you're free to use various commands like `ping`, `select 0`, `dbsize`, `scan 0` to interact with the Redis server.

So far Redis should return an empty result for `dbsize` and `scan 0` requests. But it is a great indicator that the Redis is up and running:

![CMD](https://github.com/HordeBies/Redis-Learning/assets/73644073/f6c81d9d-6808-41f5-a932-9b4c5decd2df)

## 4. Blazor Project Setup

After ensuring the Redis server is up and running, let's create a Blazor project using Visual Studio.

1. Create a new Blazor project in Visual Studio.

![Blazor](https://github.com/HordeBies/Redis-Learning/assets/73644073/8dc8c980-3eeb-491d-8d17-d5d62811d23e)

## 5. Configure Blazor Project to Use Redis

In order to use Redis in your Blazor project, you'll need to perform the following steps:

1. Install the required NuGet packages by adding the following lines to your project's `.csproj` file:
```xml
<ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Caching.StackExchangeRedis" Version="x.x.x" />
    <PackageReference Include="StackExchange.Redis" Version="x.x.x" />
</ItemGroup>
```
Replace `x.x.x` with the appropriate version numbers, or use NuGet Package Manager to install.

![NuGet](https://github.com/HordeBies/Redis-Learning/assets/73644073/8ce06f37-39a1-444c-8e04-40895dc82d6c)

2. Add the Redis connection string to your `appsettings.json`:
```json
"ConnectionStrings": {
    "Redis": "localhost:6379"
}
```

3. Configure the Redis service in your `Program.cs` file:
```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "RedisDemo_";
});
```

## 6. Extension Methods for Redis

To facilitate saving and loading data from Redis, you'll need to create extension methods. Create a static class named `DistributedCacheExtensions.cs` in your project's `Extensions` folder and add the following methods:

```csharp
public static async Task SetAsync<T>(this IDistributedCache cache, string key, T value, TimeSpan? absoluteExpireTime = null, TimeSpan? unusedExpireTime = null)
{
    DistributedCacheEntryOptions options = new()
    {
        AbsoluteExpirationRelativeToNow = absoluteExpireTime ?? TimeSpan.FromSeconds(60),
        SlidingExpiration = unusedExpireTime
    };
    var json = JsonSerializer.Serialize(value);
    await cache.SetStringAsync(key, json, options);
}

public static async Task<T?> GetAsync<T>(this IDistributedCache cache, string key)
{
    var json = await cache.GetStringAsync(key);
    return json is null ? default : JsonSerializer.Deserialize<T>(json);
}
```

## 7. Using Redis in Blazor

To use Redis in your Blazor project, follow these steps:

1. Create a method in your `FetchData.razor` file that will fetch data from a service and cache it (it will use default expiration). Note that key is changing every minute:
```csharp
private WeatherForecast[]? forecasts;
private string? loadLocation = "";
private string isCacheData = "";

private async Task LoadForecast()
{
    forecasts = null;
    loadLocation = null;
    var key = "WeatherForecast_" + DateTime.Now.ToString("yyyyMMdd_hhmm");

    forecasts = await Cache.GetAsync<WeatherForecast[]>(key);
    if (forecasts is null)
    {
        forecasts = await ForecastService.GetForecastAsync(DateOnly.FromDateTime(DateTime.Now));
        loadLocation = $"Loaded from API at {DateTime.Now}";
        isCacheData = "";
        await Cache.SetAsync(key, forecasts);
    }
    else
    {
        loadLocation = $"Loaded from CACHE at {DateTime.Now}";
        isCacheData = "text-danger";
    }
}
```

2. Bind the method to your `FetchData.razor` page:
```razor
<PageTitle>Weather forecast</PageTitle>

<h1>Weather forecast</h1>

<p>This component demonstrates fetching data from a service.</p>

<button class="btn btn-primary mb-3 " @onclick="LoadForecast" >Load Forecast</button>
@if (forecasts is null && loadLocation == "")
{
    <p><em>Click the button to load the forecast</em></p>
}
else if (forecasts is null)
{
    <p><em>Loading...</em></p>
}
else
{
    <div class="h3 @isCacheData">@loadLocation</div>
    <table class="table">
        <thead>
            <tr>
                <th>Date</th>
                <th>Temp. (C)</th>
                <th>Temp. (F)</th>
                <th>Summary</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var forecast in forecasts)
            {
                <tr>
                    <td>@forecast.Date.ToShortDateString()</td>
                    <td>@forecast.TemperatureC</td>
                    <td>@forecast.TemperatureF</td>
                    <td>@forecast.Summary</td>
                </tr>
            }
        </tbody>
    </table>
}
```

Now, when you run your project, you can click to load the weather forecast. It will load from the API the first time, and subsequent clicks will load the data from the Redis cache. 
Lets see how visiting same page with 3 different browsers(or reloading from same browser) result (first one hits the api and returns the data and caches it, other browsers just use the same cached data): 

![Browsers](https://github.com/HordeBies/Redis-Learning/assets/73644073/4da9a148-a470-4943-a08b-22f8dfb81a11)

Lets see how that is stored in redis container. First you connect to redis container shell `docker exec -it Redis sh` then you connect to Redis command line with `redis-cli`. Now use `scan 0` to see your cached data:

![Scan0](https://github.com/HordeBies/Redis-Learning/assets/73644073/aa06e6eb-ff91-4198-910a-f6953db5fe83)

(P.S. Cache is only valid for 1 minutes and key is valid for the same minute, so if you cache the data at XX:45 it will be live for 15 seconds then new request will create another cache for the new minute but the old cache will still exist in redis for 45 more seconds which will create unusable cache, therefore you might want to tweak key name and expiration timings to better suit your needs.)

![UnusableCache](https://github.com/HordeBies/Redis-Learning/assets/73644073/d1bf4a37-1a2f-4d4d-a172-676c5c87e691)

## 8. Conclusion
This concludes the basic setup and usage guide for using Redis in a Blazor project. You've learned how to set up Redis using Docker, integrate Redis with your Blazor project, and use it as a caching mechanism. Feel free to change caching parameters (i.e. cache key, caching duration with sliding and absolute expiration times), explore more advanced features and use cases for Redis as you continue your learning journey. Happy coding!
