- [pgRouting教程八：使用pl/pgsql写存储过程 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/136690455)

PostgreSQL支持多种语言来写存储过程，本文将使用**pl/pgsql**来写一个新的存储过程（函数）。

## 一、规划从A点到B点的路径

以下函数以基于EPSG:3857坐标系的坐标点作为输入参数，并返回可在QGIS或支持WMS服务的WebGIS服务器（如Mapserver和Geoserver）中显示的路径信息：

**1）输入参数：**

- 表名或视图名
- 起点（x1, y1）和终点（x2, y2）

**2）输出参数：**

- seq —— 结果路径的排序号
- gid —— shenzhen_roads表中对应路段的id
- name —— 路径中的路段名
- costs —— 路径的长度（单位公里）
- azimuth —— 路段的方位角
- route_geom —— 路径的几何信息

## 二、顶点表

图有一组边和一组与其相关联的顶点。之前创建路网拓扑时生成了与**路径表（shenzhen_raods）**相关联的**顶点表（shenzhen_roads_vertices_pgr）**。当使用路径表的子集vehicle_net或little_net时，必须也使用与这些子集相关联的顶点集，以便于找到和上面所说的函数的输入起点和输入终点最近的顶点。

**2.1、练习1 —— 顶点的数量**

找到和以下路径表和路径表的子集相联系的顶点的数量：

- shenzhen_roads
- vehicle_net
- little_net

```sql
-- 和shenzhen_roads表相联系的顶点的数量
SELECT COUNT(*) FROM shenzhen_roads_vertices_pgr;

-- 和vehicle_net视图相联系的顶点的数量
SELECT COUNT(*) FROM shenzhen_roads_vertices_pgr
WHERE id IN (
	SELECT source FROM vehicle_net
	UNION
	SELECT target FROM vehicle_net
);

-- 和little_net视图相联系的顶点的数量
SELECT COUNT(*) FROM shenzhen_roads_vertices_pgr
WHERE id IN (
	SELECT source FROM little_net
	UNION
	SELECT target FROM little_net
);
```

**2.2、练习2 —— 最近顶点**

计算距离坐标点（12677354.9,2578172.3）最近的顶点。

```sql
-- 基于全部顶点计算
SELECT id 
FROM shenzhen_roads_vertices_pgr
ORDER BY the_geom <-> ST_SetSRID(ST_Point(12677354.9,2578172.3), 3857)
LIMIT 1;

-- 基于vehicle_net视图计算
WITH
vertices AS (
    SELECT * FROM shenzhen_roads_vertices_pgr
    WHERE id IN (
        SELECT source FROM vehicle_net
        UNION
        SELECT target FROM vehicle_net)
)
SELECT id FROM vertices
    ORDER BY the_geom <-> ST_SetSRID(ST_Point(12677354.9,2578172.3), 3857) LIMIT 1;

-- 基于little_net视图计算
WITH
vertices AS (
    SELECT * FROM shenzhen_roads_vertices_pgr
    WHERE id IN (
        SELECT source FROM little_net
        UNION
        SELECT target FROM little_net)
)
SELECT id FROM vertices
    ORDER BY the_geom <-> ST_SetSRID(ST_Point(12677354.9,2578172.3), 3857) LIMIT 1;
```

以上三个计算的结果都是id=11345。

## 三、编写函数wrk_fromAtoB

可以将上面的业务逻辑写到函数wrk_fromAtoB中。

**3.1、练习3 —— 创建函数**

创建函数wrk_fromAtoB：

```sql
CREATE OR REPLACE FUNCTION wrk_fromAtoB(
	IN edges_subset regclass,
	IN x1 NUMERIC, IN y1 numeric,
	IN x2 NUMERIC, IN y2 NUMERIC,
	OUT seq INTEGER,
	OUT gid BIGINT,
	OUT name TEXT,
	OUT costs FLOAT,
	OUT azimuth FLOAT,
	OUT geom GEOMETRY
)
RETURNS SETOF record AS
$BODY$
DECLARE 
	final_query TEXT;
BEGIN
	final_query := 
		FORMAT($$
			WITH
			vertices AS (
				SELECT * FROM shenzhen_roads_vertices_pgr
				WHERE id IN (
					SELECT source FROM %1$I
					UNION
					SELECT target FROM %1$I
				)
			),
			dijkstra AS (
				SELECT * 
				FROM wrk_dijkstra(
					'%1$I',
					-- source
					(SELECT id FROM vertices
                        ORDER BY the_geom <-> ST_SetSRID(ST_Point(%2$s, %3$s), 3857) LIMIT 1),
					-- target
					(SELECT id FROM vertices
                        ORDER BY the_geom <-> ST_SetSRID(ST_Point(%4$s, %5$s), 3857) LIMIT 1)
				)
			)
			SELECT 
			   seq,
			   dijkstra.gid,
			   dijkstra.name,
			   dijkstra.cost / 1000.0 AS costs,
			   azimuth,
			   route_geom AS geom
			FROM dijkstra 
			JOIN shenzhen_roads ways USING(gid);$$,
		edges_subset, x1,y1,x2,y2
		);
	RAISE notice '%', final_query; -- 执行该函数时显示函数的逻辑代码信息
	RETURN QUERY EXECUTE final_query;
END;
$BODY$
LANGUAGE 'plpgsql';
```

这里使用了PostgreSQL的FORMAT函数，它用于格式化字符串，它和C函数`sprintf`相似。它的函数签名是这样的：

```sql
format(formatstr text [, formatarg "any" [, ...] ])
```

比如可以来看这样一个示例：

```sql
format('Hello %s, %1$s', 'World') -- 结果：Hello World, World
```

**3.2、练习4 —— 使用函数**

使用刚创建的函数**wrk_fromAtoB**计算从坐标点（12677354.9, 2578172.3）到坐标点（12677441.2, 2577908.3）的路径信息。

```sql
SELECT * FROM wrk_fromAtoB(
	'vehicle_net',
	12677354.9,2578172.3,
	12677441.2, 2577908.3
);

SELECT * FROM wrk_fromAtoB(
	'little_net',
	12677354.9,2578172.3,
	12677441.2, 2577908.3
);

SELECT * FROM wrk_fromAtoB(
	'shenzhen_roads',
	12677354.9,2578172.3,
	12677441.2, 2577908.3
);
```

当函数执行完毕时，我们可以点击pgAdmin结果栏中的Messages标签查看函数的逻辑代码信息：

![img](https://pic4.zhimg.com/80/v2-597940ad4b757736a7c80e87af28b14f_720w.jpg)

另外当然也能点击Data Output标签中的geom列的按钮查看路径的可视化：

![img](https://pic2.zhimg.com/80/v2-0fe6dbd2b3a0ed4ea1691a063ffe227d_720w.jpg)

![img](https://pic4.zhimg.com/80/v2-23c4298bf5f3fb45d4bcfdcf9aee3207_720w.jpg)