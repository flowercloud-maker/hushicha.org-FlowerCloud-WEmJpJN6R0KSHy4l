**目录**

* [前言](#_label0)
* [安装consul](#_label1)
* [在GoodApi项目中修改Program.cs](#_label2)
	+ [加Dockerfile文件](#_label2_0)
* [在OcelotGA中加consul配置与代码](#_label3)
	+ [改ocelot.json配置](#_label3_0)
	+ [修改Program.cs](#_label3_1)
* [爬坑记录](#_label4)
* [作者](#_label5)
 



---


[回到顶部](#_labelTop)# 前言


没看[dotnet微服务之API网关Ocelot](https://github.com)的请先看，这篇文章接上面文章


[回到顶部](#_labelTop):[樱花宇宙官网](https://yzygzn.com)# 安装consul



```
#自定义网络，自定义网络可以指定容器IP，这样服务器重启consul集群也可以正常运行。
docker network create --driver bridge --subnet=172.21.0.0/16 --gateway=172.21.0.16 adnc_consul

docker run -d -p 8500:8500 -p 8600:8600 -p 8301:8301 --restart=always --network=adnc_consul --ip 172.21.0.1 --privileged=true --name=consul_server_1  --name consul consul:1.15.4 agent -server -bootstrap -ui -node=consul_server_1 -client='0.0.0.0'

```

[回到顶部](#_labelTop)# 在GoodApi项目中修改Program.cs


先要添加Consul包


再添加Consul注册于注销等相关代码



```
using Consul;
using System.Linq;
using System.Net;
using System.Net.Sockets;

// 创建Consul客户端
var consulAddress = "http://10.75.174.43:8500";// Environment.GetEnvironmentVariable("CONSUL_ADDRESS"); //10.75.174.43
var consulUri = new Uri(consulAddress);
var client = new ConsulClient(config =>
{
    config.Address = consulUri;
});

// 配置服务的健康检查
var check = new AgentServiceCheck()
{
    HTTP = $"http://{GetLocalIpAddress("InterNetwork").FirstOrDefault()}:8080/health", // 健康检查地址
    Interval = TimeSpan.FromSeconds(10) // 检查间隔
};
var serviceId = "goodapi-1"; // 要注销的服务的ID
// 注册一个服务
var registration = new AgentServiceRegistration()
{
    ID = serviceId,
    Name = "goodapi",
    Address = GetLocalIpAddress("InterNetwork").FirstOrDefault(),
    Port = 8080,
    Check = check
};

await client.Agent.ServiceDeregister(serviceId);
client.Agent.ServiceRegister(registration);

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.

builder.Services.AddControllers();
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure the HTTP request pipeline.
//if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.MapControllers();

app.MapGet("/health", async context =>
{
    context.Response.StatusCode = 200;
    await context.Response.WriteAsync("health");
});

app.Run();

// 注销服务

await client.Agent.ServiceDeregister(serviceId);

List<string> GetLocalIpAddress(string netType)
{
    string hostName = Dns.GetHostName();
    IPAddress[] addresses = Dns.GetHostAddresses(hostName);

    var IPList = new List<string>();
    if (netType == string.Empty)
    {
        for (int i = 0; i < addresses.Length; i++)
        {
            IPList.Add(addresses[i].ToString());
        }
    }
    else
    {
        //AddressFamily.InterNetwork = IPv4,
        //AddressFamily.InterNetworkV6= IPv6
        for (int i = 0; i < addresses.Length; i++)
        {
            if (addresses[i].AddressFamily.ToString() == netType)
            {
                IPList.Add(addresses[i].ToString());
            }
        }
    }
    return IPList;
}

```

## 加Dockerfile文件



```
#使用asp.net 6作为基础镜像，起一个别名为base
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
#设置容器的工作目录为/app
WORKDIR /app
#COPY 文件
COPY . /app
ENV ASPNETCORE_ENVIRONMENT Production

#设置时区为东八区
ENV TZ Asia/Shanghai
#启动服务
ENTRYPOINT ["dotnet", "GoodApi.dll"]

```


```
# 运行goodapi项目
docker stop goodapi
docker rm -f goodapi
docker build -t goodapi .
docker run --name=goodapi -d -p 8080:8080 --network=adnc_consul goodapi

```

[回到顶部](#_labelTop)# 在OcelotGA中加consul配置与代码


加包Consul，Ocelot.Provider.Consul


## 改ocelot.json配置



```
{
  "Routes": [
    {
      "UseServiceDiscovery": true,
      "UpstreamPathTemplate": "/good/{everything}",
      "UpstreamHttpMethod": [ "Get" ],
      "DownstreamPathTemplate": "/{everything}",
      "DownstreamScheme": "http",
      "ServiceName": "goodapi",
      "LoadBalancerOptions": {
        "Type": "LeastConnection"
      }
    },
    {
      "UseServiceDiscovery": true,
      "UpstreamPathTemplate": "/order/{everything}",
      "UpstreamHttpMethod": [ "Get" ],
      "DownstreamPathTemplate": "/{everything}",
      "DownstreamScheme": "http",
      "ServiceName": "orderapi",
      "LoadBalancerOptions": {
        "Type": "LeastConnection"
      }
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "http://gw.wxy.ink",
    "ServiceDiscoveryProvider": {
      "Scheme": "http",
      "Host": "10.75.174.43", // 这里是您Consul的地址
      "Port": 8500, // Consul的端口
      "Type": "Consul"
    }
  }
}

```

## 修改Program.cs



```
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Ocelot.DependencyInjection;
using Ocelot.Middleware;
using Ocelot.Provider.Consul;
using System;
using System.Collections.Generic;
using System.Net;

using Consul;
using OcelotGA;

// 创建Consul客户端
var consulAddress = "http://10.75.174.43:8500";// Environment.GetEnvironmentVariable("CONSUL_ADDRESS"); //10.75.174.43
var consulUri = new Uri(consulAddress);
var client = new ConsulClient(config =>
{
    config.Address = consulUri;
});

// 配置服务的健康检查
var check = new AgentServiceCheck()
{
    HTTP = $"http://{GetLocalIpAddress("InterNetwork").FirstOrDefault()}:8080/health", // 健康检查地址
    Interval = TimeSpan.FromSeconds(10) // 检查间隔
};
var serviceId = "gw-1"; // 要注销的服务的ID
// 注册一个服务
var registration = new AgentServiceRegistration()
{
    ID = serviceId,
    Name = "gw",
    Address = GetLocalIpAddress("InterNetwork").FirstOrDefault(),
    Port = 8080,
    //Check = check
};

await client.Agent.ServiceDeregister(serviceId);
client.Agent.ServiceRegister(registration);

var builder = WebApplication.CreateBuilder(args);

builder.Configuration.AddJsonFile("ocelot.json", optional: false, reloadOnChange: true);

builder.Services.AddOcelot().AddConsul(); //这里要注意MyConsulServiceBuilder

var app = builder.Build();

app.UseOcelot().Wait();

app.UseRouting();

app.MapGet("/Health", async context =>
{
    context.Response.StatusCode = 200;
    await context.Response.WriteAsync("Health");
});

app.Run();

// 注销服务

await client.Agent.ServiceDeregister(serviceId);

List<string> GetLocalIpAddress(string netType)
{
    string hostName = Dns.GetHostName();
    IPAddress[] addresses = Dns.GetHostAddresses(hostName);

    var IPList = new List<string>();
    if (netType == string.Empty)
    {
        for (int i = 0; i < addresses.Length; i++)
        {
            IPList.Add(addresses[i].ToString());
        }
    }
    else
    {
        //AddressFamily.InterNetwork = IPv4,
        //AddressFamily.InterNetworkV6= IPv6
        for (int i = 0; i < addresses.Length; i++)
        {
            if (addresses[i].AddressFamily.ToString() == netType)
            {
                IPList.Add(addresses[i].ToString());
            }
        }
    }
    return IPList;
}

```

添加MyConsulServiceBuilder.cs



```
using Consul;
using Ocelot.Logging;
using Ocelot.Provider.Consul;
using Ocelot.Provider.Consul.Interfaces;

namespace OcelotGA
{
    public class MyConsulServiceBuilder : DefaultConsulServiceBuilder
    {
        public MyConsulServiceBuilder(IHttpContextAccessor contextAccessor, IConsulClientFactory clientFactory, IOcelotLoggerFactory loggerFactory)
            : base(contextAccessor, clientFactory, loggerFactory) { }

        // I want to use the agent service IP address as the downstream hostname
        protected override string GetDownstreamHost(ServiceEntry entry, Node node)
            => entry.Service.Address;
    }
}

```

运行网关项目


![](https://wxy-blog.oss-cn-hangzhou.aliyuncs.com/wxy-blog/2024/202411201656483.png)


我们访问[gw.wxy.ink/good/health](https://github.com)


![](https://wxy-blog.oss-cn-hangzhou.aliyuncs.com/wxy-blog/2024/202411201656533.png)


[回到顶部](#_labelTop)# 爬坑记录


Program.cs中的



```
builder.Services.AddOcelot().AddConsul(); //这里要注意MyConsulServiceBuilder
若为
builder.Services.AddOcelot().AddConsul();

```

则会出现下面问题


![](https://wxy-blog.oss-cn-hangzhou.aliyuncs.com/wxy-blog/2024/202411201659563.png)


服务解析出来是node的名称，而非服务的IP


解决方法：[Service Discovery — Ocelot Gateway 23\.4 documentation](https://github.com)


![](https://wxy-blog.oss-cn-hangzhou.aliyuncs.com/wxy-blog/2024/202411201701213.png)


![](https://wxy-blog.oss-cn-hangzhou.aliyuncs.com/wxy-blog/2024/202411201701358.png)


就是说默认的DefaultConsulServiceBuilder会这样处理



```
protected virtual string GetDownstreamHost(ServiceEntry entry, Node node)
    => node != null ? node.Name : entry.Service.Address;

```

而我们需要的是



```
protected override string GetDownstreamHost(ServiceEntry entry, Node node)
        => entry.Service.Address;

```

[回到顶部](#_labelTop)# 作者


吴晓阳（手机：13736969112微信同号）


