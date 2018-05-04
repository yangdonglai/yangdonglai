# docker

## docker管理后台版本

分支分为 `develop` `release` `test` `master`
在develop上执行开发进度，版本发布时在release版本,release分支触发docker打包,
其中为了避免每个版本docker都重新安装`node_modules`,v2版本应继承v1版本,代码全部覆盖，并重新执行一次`npm install`,因为在 v1 的版本基础上，所以如果没有包的版本变化的话，应该不会占用太多时间

## docker-compose

### network

一般后台项目都要有数据库,实用`docker-compose`可以方便的同时启动一套`container`，一般数据库单独维护一个image，每个项目自己维护一个服务image，在`docker-compose`中配置两个`container`。
同时在docker-compose v3 版本之后，默认开启一个 default network，所以无需过多配置(expose)，多个容器间可以方便的链接。

```yml
version: "3"
services:
  zgy-server:
    container_name: zgy-server
    image: zgy-server:v1
    restart: always
    ports:
    - "4011:4011"
    depends_on:
      - zgy-server-postgres
    command: ["sh", "bin/start_from_docker.sh" ]
    environment:
      PG_HOST: zgy-pg
      PG_DATABASE: zgy-server
      PG_USER: postgres
      PG_PASSWORD: postgres
      PG_PORT: 5432
  zgy-server-postgres:
    container_name: zgy-pg
    image: harbor.gagogroup.cn/postgres/postgis
    restart: always
    volumes:
      - /Users/yangdonglai/learn/zgy-server/db:/var/lib/postgresql/data
    expose:
      - 5432
    environment:
     POSTGRES_PASSWORD: postgres
     POSTGRES_USER: postgres
     POSTGRES_DB: zgy-server
```

### 数据库容器完全启动

`depends_on`无法保证会在数据库容器`完全启动`（包括数据库初始化配置等）后再启动服务容器，因此在服务容器中必须处理数据库未启动的情况。
官网提供两种思路，一种是在代码中增加处理数据库连接失败的情况，避免过早尝试连接数据库，多次错误后服务停止，容器关闭。第二种是启动服务代码之前，确保前置服务可以正常运行。比如：

```sh
#!/bin/bash
# wait-for-postgres.sh

set -e

host="$1"
shift
cmd="$@"

until PGPASSWORD=$POSTGRES_PASSWORD psql -h "$host" -U "postgres" -c '\q'; do
  >&2 echo "Postgres is unavailable - sleeping"
  sleep 1
done

>&2 echo "Postgres is up - executing command"
exec $cmd

```

```yml
version: "2"
services:
  web:
    build: .
    ports:
      - "80:8000"
    depends_on:
      - "db"
    command: ["./wait-for-it.sh", "db:5432", "--", "python", "app.py"]
  db:
    image: postgres
```

这种前置的代码不仅仅是可以shell脚本，也可以是node代码，比如使用sequilize库等

```ts
#!/usr/bin/env ts-node
import * as Sequelize from 'sequelize';
import { STRING, ENUM, INTEGER,Sequelize as SequelizeIntance } from 'sequelize';
const config = {
  //....
};
async function delay(time:number){
  return new Promise((res)=>{
    setTimeout(res,time);
  })
}

function initSequelize() {
  return new Sequelize(config.sequelize.database  , config.sequelize.username, config.sequelize.password, {
    host: config.sequelize.host,
    dialect: config.sequelize.dialect,
    pool: {
      max: 5,
      min: 0,
      acquire: 30000,
      idle: 10000
    },
    define: {
      underscored: true,
    }
  });
}

async function syncUtilSucceed(sequelize:SequelizeIntance){
  sequelize.sync().then(()=>{
    console.log('finish');
    process.exit();
  }).catch(async (e)=>{
    console.log('error',e.error);
    await delay(1000);
    await syncUtilSucceed(sequelize);
  });
}

(async () => {
  let sequelize:SequelizeIntance = initSequelize();

  const yearReg = /^20[0-9]{2}$/;

  sequelize.define('source', {
    id: {
      type: INTEGER,
      primaryKey: true,
      autoIncrement: true,
    },
    name: {
      type: ENUM,
      values: ['开发区识别地块', '开发区影像地块'],
    },
    url: {
      type: STRING,
      validate: {
        isUrl: true,
      }
    },
    year: {
      type: STRING,
      validate: {
        isYear(value) {
          if (!yearReg.test(value)) {
            throw new Error('year is invalid!');
          }
        }
      }
    },
  });
  await syncUtilSucceed(sequelize);
})();
```