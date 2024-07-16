---
title: MapLibre/Martin | 使用Martin发布MBTiles地图切片包
pubDate: 2024-07-2
categories: ["GIS", "MARTIN", "MBTILES"]
description: "新手向|使用Martin发布MBTiles地图切片包"
---

## 什么是 Martin

[Martin](https://github.com/maplibre/martin) 是一个高性能的地图切片服务器，使用`Rust`编写，支持`PostGIS`，[MBTiles](https://github.com/mapbox/mbtiles-spec)，[PMTiles](https://github.com/protomaps/PMTiles)。

## 什么是 MBTiles

[MBTiles](https://github.com/mapbox/mbtiles-spec) 是个`sqlite`文件。截至本文写作时，最新标准是[1.3](https://github.com/mapbox/mbtiles-spec/blob/master/1.3/spec.md).
`MBTIles`利用了数据库的索引机制，避免相同内容的切片重复占用空间，同时也有了 `SQLITE` 数据库单文件的优点，非常方便传输与利用。

### Tiles

```sql
CREATE TABLE tiles (
    zoom_level  INTEGER, -- Z
    tile_column INTEGER, -- Y
    tile_row    INTEGER, -- X
    tile_data   BLOB);   -- 切片数据

CREATE UNIQUE INDEX tile_index on tiles (
    zoom_level, tile_column, tile_row);
```

### Metadata

```sql
CREATE TABLE metadata (name text, value text);
```

- name
- format
- bounds
- center
- minzoom
- maxzoom
- attribution
- description
- type
- version
- json

## 为什么使用 MBTiles

- 单文件，就是爽（对比下 ArcGIS 生成的 [Bundles](https://github.com/Esri/raster-tiles-compactcache)中无数个小切片文件...🥶)
- 紧凑,配合索引机制，可以避免相同内容的切片重复出现，节省磁盘空间
- `MBTiles` 本质上还是个 `SQLITE` 数据库，解析利用都非常方便，生态良好，有大量的配套软件

|                                        | MBTiles | PMTiles | ArcGIS Bundle(即 raster-tiles-compactcache) | COG(Cloud Optimized GeoTIFF) |
| -------------------------------------- | ------- | ------- | ------------------------------------------- | ---------------------------- |
| 单文件                                 | 😄      | 😄      | 🥶                                          | 😄                           |
| 支持无服务器（`serverless`）的方式使用 | 🥶      | 😄      | 🥶                                          | 😄                           |
| 矢量                                   | 😄      | 😄      | 🥶                                          | 🥶                           |
| 栅格                                   | 😄      | 😄      | 😄                                          | 😄                           |
| 开源社区生态                           | 😄      | 😄      | 🥶                                          | 😄                           |

## 开始实验(WINDOWS)

本实验用到的所有内容地址：https://gitee.com/mapnote/blog_demo_data 。

## 准备实验数据

- 实验数据：[world_cities.mbtiles](https://gitee.com/mapnote/blog_demo_data/raw/master/martin_mbtiles/world_cities.mbtiles)
- 实验软件：[martin.exe](https://gitee.com/mapnote/blog_demo_data/raw/master/martin_mbtiles/martin.exe)

## 发布服务

1. 新建文件夹: `martin_demo`
2. 将 w`orld_cities.mbtiles` 放到 `martin_demo` 文件夹中
3. 将 `martin.exe` 放到 `martin_demo` 文件夹中
4. 按住 SHIFT 键不放，在文件夹内右击，点击“在此处打开 `Powershell` 窗口
5. 在 Powershell 窗口内输入`.\martin.exe .\world_cities.mbtiles`

```Powershell
.\martin.exe .\world_cities.mbtiles
```

运行该命令后，`martin` 会在您电脑的 `3000` 端口上启动地图服务。如果需要修改端口，可以用`--listen-addresses` 修改端口号。（完整的启动命令参数请参阅[官方文档](https://maplibre.org/martin/run-with-cli.html)）

```powershll
.\martin.exe .\world_cities.mbtiles --listen-addresses *:5000
```

## 查看服务信息

### 服务目录

访问`http://localhost:3000/catalog`查看切片服务目录

```json
{
  "tiles": {
    "world_cities": {
      //切片服务名称，即SourceID
      "content_type": "application/x-protobuf", //切片格式的MIME，application/x-protobuf表示是MVT(mapbox vector tile)
      "content_encoding": "gzip", // 是否压缩及其用到的压缩编码
      "name": "Major cities from Natural Earth data",
      "description": "Major cities from Natural Earth data"
    }
  },
  "sprites": {},
  "fonts": {}
}
```

`tiles` 下就是所有的切片服务目录了，根据以上属性信息可以知道：

1. 只发布了一个切片服务，它的服务名称（`SourceId`）是 `world_cities`
2. `content_type` 是 `application/x-protobuf`，表明切片格式 `MVT`

### 服务 TileJson

切片服务跟客户端主要是用 `TileJson` `沟通，TileJson` 中详细写明了 `OpenLayer、Mapbox、MapLibre` 等客户端切片服务需要了解的参数，要查看 `Martin` 发布的服务的 `tiljeson，只需要访问` http://ip 地址:端口/服务名称

在这次实验中，对应的地址就是 `http://localhost:3000/world_cities`

会返回如下 `Json`：

```json
{
  "tilejson": "3.0.0",
  "tiles": [
    "http://localhost:3000/world_cities/{z}/{x}/{y}" //切片服务地址，OpenLayers等客户端要用
  ],
  "vector_layers": [
    {
      "id": "cities", // 要素ID字段名
      "fields": {
        // 要素中所有的字段
        "name": "String"
      },
      "description": "",
      "maxzoom": 6,
      "minzoom": 0
    }
  ],
  "bounds": [
    //切片数据的地理覆盖范围
    -123.12359, -37.818085, 174.763027, 59.352706
  ],
  "center": [
    // 切片数据的中心点
    -75.9375, 38.788894, 6
  ],
  "description": "Major cities from Natural Earth data",
  "maxzoom": 6, // 最大缩放层级
  "minzoom": 0, // 最小缩放层级
  "name": "Major cities from Natural Earth data",
  "version": "2",
  "format": "pbf"
}
```

## 使用 QGIS 查看服务

1. 可以使用 `QGIS` 快速查看这次实验发布的切片，打开 `QGIS`
2. 右击左侧的 `Vector Tiles`，点击新建，
3. 名称任意填写，我填的是 `world_cities`
4. `URL` 地址填写 `TileJson` 中的地址 `http://localhost:3000/world_cities/{z}/{x}/{y}`
5. 最小缩放层级填写 `0`
6. 最大缩放层级填写 `6`
7. 填写完成后点击确定
8. 双击自己新建的服务，后会自动加载到地图

## 使用 OpenLayers 加载服务

1. 在 `martin_demo` 文件夹内新建一个文件夹`ol`用来编写我们的前端代码
2. 使用 `vscode` 打开我们的 ol 文件夹
3. 打开命令行输入 `npm create ol-app`，等待命令执行完成
4. 删除 main.js 中所有内容，输入以下内容。

   ```js
   import "./style.css";
   import { Map, View } from "ol";
   import MVT from "ol/format/MVT.js";
   import TileLayer from "ol/layer/Tile";
   import OSM from "ol/source/OSM";
   import VectorTileLayer from "ol/layer/VectorTile.js";
   import VectorTileSource from "ol/source/VectorTile.js";

   let my_layer = new VectorTileLayer({
     source: new VectorTileSource({
       format: new MVT(),
       url: "http://localhost:3000/world_cities/{z}/{x}/{y}",
     }),
   });

   const map = new Map({
     target: "map",
     layers: [
       new TileLayer({
         source: new OSM(),
       }),
       my_layer,
     ],
     view: new View({
       center: [0, 0],
       zoom: 2,
     }),
   });
   ```

5. vscode 的命令行中输入`npm install`，安装所有依赖
6. vscode 的命令行中输入`npm start`运行前端代码服务
7. 命令行中会提示服务地址，我这里是`http://localhost:5174/`,访问该地址即可看到效果。
