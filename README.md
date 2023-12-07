# Nest + Redis + 地图，实现附近共享充电宝

## redis

 不仅可以做缓存、很多临时的数据，比如验证码、token等，都可以放 redis 里。

redis 是 key-value 数据库，value 有很多类型：

- string：存数字、字符串，比如存验证码就是这种类型
- hash：存一个 map 的结构，比如文章的点赞数、收藏数、阅读量，就可以用 hash 存。
- set：存去重后的集合数据，支持交集、并集等计算。
- zset：排序的集合，可以指定一个分数，按照分数排序。我们每天看的文章热榜、微博热榜等各种排行榜，都是 zset 做的
- list：存列表数据
- geo：存地理位置，支持地理位置之间的距离计算、按照半径搜索附近的位置
## 步骤

1. 安装Doker
   ![image-20231207113002782](https://sunchenrui666.oss-cn-guangzhou.aliyuncs.com/img/image-20231207113002782.png)
2. 安装RedisInsight

 <div align=center><img src="https://sunchenrui666.oss-cn-guangzhou.aliyuncs.com/img/image-20231207113054856.png" alt="image-20231207113054856" style="zoom:50%;" /></div>

3. 创建nest项目

 ```bash
npm install g @nestjs/cli

nest new nearby-search
 ```

- 进入项目目录，启动服务

```bash
npm run start:dev
```

![image-20231207114520504](https://sunchenrui666.oss-cn-guangzhou.aliyuncs.com/img/image-20231207114520504.png)

http://localhost:3000  上nest服务跑起来了

- 安装连接 redis 的包

```bash
npm install --save redis
```

- 创建 redis 模块和 service

```bash
nest g module redis
nest g service redis
```

<img src="https://sunchenrui666.oss-cn-guangzhou.aliyuncs.com/img/image-20231207115407971.png" alt="image-20231207115407971" style="zoom:50%;" />

- 在 RedisModule 创建连接 redis 的 provider，导出 RedisService;

```typescript
import { Module } from '@nestjs/common';
import { createClient } from 'redis';
import { RedisService } from './redis.service';

@Module({
  providers: [
    RedisService,
    {
      provide: 'REDIS_CLIENT',
      async useFactory()  {
        const client = createClient({
          socket: {
            host: 'localhost',
            port: 6379
          }
        });
        await client.connect();
        return client;
      }
    }
  ],
  exports: [RedisService]
})
export class RedisModule {}
```

- 在 RedisService 里注入 REDIS_CLIENT，并封装一些操作 redis 的方法

```typescript
import { Inject, Injectable } from '@nestjs/common';
import { RedisClientType } from 'redis';

@Injectable()
export class RedisService {

    @Inject('REDIS_CLIENT') 
    private redisClient: RedisClientType;

    async geoAdd(key: string, posName: string, posLoc: [number, number]) {
        return await this.redisClient.geoAdd(key, {
            longitude: posLoc[0],
            latitude: posLoc[1],
            member: posName
        })
    }
}
```

- 添加geoAdd方法，传入 key 和位置信息，底层调用 redis 的 geoadd 来添加数据

  在 AppController 里注入 RedisService ，然后添加一个路由：

```typescript
import { BadRequestException, Controller, Get, Inject, Query } from '@nestjs/common';
import { AppService } from './app.service';
import { RedisService } from './redis/redis.service';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Inject(RedisService)
  private redisService: RedisService;

  @Get('addPos')
  async addPos(
    @Query('name') posName: string,
    @Query('longitude') longitude: number,
    @Query('latitude') latitude: number,
  ) {
    if (!posName || !longitude || !latitude) {
      throw new BadRequestException('位置信息不全');
    }
    try {
      await this.redisService.geoAdd('positions', posName, [longitude, latitude]);
    } catch(e) {
      throw new BadRequestException()
    }

    return {
      message: '添加成功',
      statusCode: 200
    }
  }
}
```

<img src="https://sunchenrui666.oss-cn-guangzhou.aliyuncs.com/img/image-20231207153125121.png" alt="image-20231207153125121" style="zoom:50%;" />

​			  接口测试：

<img src="C:\Users\78684\AppData\Roaming\Typora\typora-user-images\image-20231207153251802.png" alt="image-20231207153251802" style="zoom:50%;" />

AppController 添加两个路由:

```typescript
@Get('allPos')
async allPos() {
    return this.redisService.geoList('positions');
}

@Get('pos')
async pos(@Query('name') name: string) {
    return this.redisService.geoPos('positions', name);
}
```

访问pos：

<img src="https://sunchenrui666.oss-cn-guangzhou.aliyuncs.com/img/image-20231207161934815.png" alt="image-20231207161934815" style="zoom:50%;" />

访问allPos:

<img src="https://sunchenrui666.oss-cn-guangzhou.aliyuncs.com/img/image-20231207162106199.png" alt="image-20231207162106199" />

在 RedisInsight 里算下两点的距离：

```redis
GEODIST positions guang dong km
```

大概 5561 km

在 guang 附近搜索半径 5000km 内的位置:

![image-20231207162559862](https://sunchenrui666.oss-cn-guangzhou.aliyuncs.com/img/image-20231207162559862.png)

搜索半径 6000 km 内的位置：

![image-20231207162751261](https://sunchenrui666.oss-cn-guangzhou.aliyuncs.com/img/image-20231207162751261.png)

目前经纬度是随便给的。

可以用高德地图的坐标拾取工具来取几个位置：

>/addPos?name=天安门&longitude=116.397444&latitude=39.909183
>
>/addPos?name=文化宫科技馆&longitude=116.3993&latitude=39.908578
>
>/addPos?name=售票处&longitude=116.397283&latitude=39.90943
>
>/addPos?name=故宫彩扩部&longitude=116.398002&latitude=39.909175
>
>
>
>作者：zxg_神说要有光
>链接：https://juejin.cn/post/7283873796976754740
>来源：稀土掘金
>著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

- 注册高德API key

  - 创建应用，复制demo代码

  - 引入 axios，调用服务端接口

    ```html
    <!doctype html>
    <html>
    <head>
        <meta charset="utf-8">
        <meta http-equiv="X-UA-Compatible" content="IE=edge">
        <meta name="viewport" content="initial-scale=1.0, user-scalable=no, width=device-width">
        <title>覆盖物的添加与移除</title>
      <link rel="stylesheet" href="https://a.amap.com/jsapi_demos/static/demo-center/css/demo-center.css" />
        <script src="https://cache.amap.com/lbs/static/es5.min.js"></script>
        <script type="text/javascript" src="https://cache.amap.com/lbs/static/addToolbar.js"></script>
        <style>
            html,
            body,
            #container {
              width: 100%;
              height: 100%;
            }
        </style>
    </head>
    <body>
    <div id="container"></div>
    <script src="https://webapi.amap.com/maps?v=2.0&key=f96fa52474cedb7477302d4163b3aa09"></script>
    <script src="https://unpkg.com/axios@1.5.1/dist/axios.min.js"></script>
    <script>
    
        const radius = 0.2;
    
        axios.get('/nearbySearch', {
            params: {
                longitude: 116.397444,
                latitude: 39.909183,
                radius
            }
        }).then(res => {
            const data = res.data;
    
            var map = new AMap.Map('container', {
                resizeEnable: true,
                zoom: 6,
                center: [116.397444, 39.909183]
            });
    
            data.forEach(item => {
                var marker = new AMap.Marker({
                    icon: "https://webapi.amap.com/theme/v1.3/markers/n/mark_b.png",
                    position: [item.longitude, item.latitude],
                    anchor: 'bottom-center'
                });
                map.add(marker);
            });
    
    
            var circle = new AMap.Circle({
                center: new AMap.LngLat(116.397444, 39.909183), // 圆心位置
                radius: radius * 1000,
                strokeColor: "#F33",  //线颜色
                strokeOpacity: 1,  //线透明度
                strokeWeight: 3,  //线粗细度
                fillColor: "#ee2200",  //填充颜色
                fillOpacity: 0.35 //填充透明度
            });
    
            map.add(circle);
            map.setFitView();
        })
            
    </script>
    </body>
    </html>
    ```

    效果图：

    <img src="https://sunchenrui666.oss-cn-guangzhou.aliyuncs.com/img/image-20231207184249742.png" alt="image-20231207184249742" style="zoom:50%;" />

    ​	

总结：

>我们经常会使用基于位置的功能，比如附近的充电宝，酒店，打车，附近的人等功能。
>
>这些都是基于 redis 实现的，因为 redis 由 geo 的数据结构，可以方便的计算两点的距离，计算某个半径内的点。
>
>前端部分使用地图的 sdk 分别在搜出的点处绘制 marker 就好
>
>geo 的底层数据结构是 zset ，所以可以使用 zset 的命令
>
>我们在 Nest 里封装了 geoadd、geopos、zrange、georadius 等 redis 命令。实现了添加点，搜索附近的点的功能。
>
>作者：zxg_神说要有光
>链接：https://juejin.cn/post/7283873796976754740
>来源：稀土掘金
>著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。





