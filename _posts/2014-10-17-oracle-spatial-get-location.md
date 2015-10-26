---
layout: post
title: Oracle Spitail根据经纬度获取结构化地址
description: 由经纬度经纬度得到结构化地址解析。例于：
categories: [html5]
tags: [HTML5,HTML5+,Javascript,Canvas,Image,移动开发]
---


{% highlight sql %}

CREATE OR REPLACE FUNCTION F_GET_LOCATION(lon NUMBER, lat NUMBER)
  RETURN VARCHAR2 IS
  -- 作者：万品佳
  -- 时间：2014-10-17
  -- 描述：根据经纬度获取结构化地址
  -- 返回值格式：省-市-区-街-门牌号
  -- 优先采取str_a的地址，其次为tra_cl的地址
  v_provice       varchar2(50) := '广东省'; -- 省
  v_city          varchar2(50) := '深圳市'; -- 市
  v_district      varchar2(50); -- 区
  v_street        varchar2(50); -- 街
  v_street_number varchar2(50); -- 门牌号
  v_point         sdo_geometry; -- 点
  v_str_radius    varchar2(50) := 'DISTANCE=100 UNIT=METER'; -- 结构化地址搜索距离半径（单位米）
  v_tra_radius    varchar2(50) := 'DISTANCE=1000 UNIT=METER'; -- 公路搜索距离半径（单位米）
BEGIN
  -- 定义点
  v_point := MDSYS.SDO_GEOMETRY(2001,
                                4326,
                                SDO_POINT_TYPE(lon, lat, NULL),
                                NULL,
                                NULL);

  -- 从附近的结构化地址查找       
  BEGIN
    SELECT B.ROAD, B.HOUSE_NUMBER
      INTO v_street, v_street_number
      FROM XXX_MAP_STR_A B,
           (SELECT A.ID,
                   SDO_GEOM.SDO_DISTANCE(A.GEOMETRY,
                                         v_point,
                                         0.0001,
                                         'UNIT=M') DISTANCE
              FROM XXX_MAP_STR_A A
             WHERE A.NAME IS NOT NULL
               AND SDO_WITHIN_DISTANCE(A.GEOMETRY, v_point, v_str_radius) =
                   'TRUE'
             ORDER BY DISTANCE) C
     WHERE C.ID = B.ID
       AND ROWNUM = 1;
  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      v_street        := NULL;
      v_street_number := '';
  END;
  IF v_street IS NULL THEN
    -- 如果结构化地址查找不到就查找附近的公路
    BEGIN
      SELECT B.NAME, ''
        INTO v_street, v_street_number
        FROM XXX_MAP_TRA_CL B,
             (SELECT A.ID,
                     SDO_GEOM.SDO_DISTANCE(A.GEOMETRY,
                                           v_point,
                                           0.0001,
                                           'UNIT=M') DISTANCE
                FROM XXX_MAP_TRA_CL A
               WHERE A.NAME IS NOT NULL
                 AND SDO_WITHIN_DISTANCE(A.GEOMETRY, v_point, v_tra_radius) =
                     'TRUE'
               ORDER BY DISTANCE) C
       WHERE C.ID = B.ID
         AND ROWNUM = 1;
    EXCEPTION
      WHEN NO_DATA_FOUND THEN
        v_street        := '';
        v_street_number := '';
    END;
  END IF;

  --查找区
  BEGIN
    SELECT B.NAME
      INTO v_district
      FROM XXX_MAP_CDIS_S B,
           (SELECT A.ID,
                   SDO_GEOM.SDO_DISTANCE(A.GEOMETRY,
                                         v_point,
                                         0.0001,
                                         'UNIT=M') DISTANCE
              FROM XXX_MAP_CDIS_S A
             WHERE A.NAME IS NOT NULL
               AND SDO_WITHIN_DISTANCE(A.GEOMETRY, v_point, v_str_radius) =
                   'TRUE'
             ORDER BY DISTANCE) C
     WHERE C.ID = B.ID
       AND ROWNUM = 1;
  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      v_district := '';
  END;

  RETURN '{"provice":"' || v_provice || '","city":"' || v_city || '","district":"' || v_district || '","street":"' || v_street || '","streetNumber":"' || v_street_number || '"}';
END F_GET_LOCATION;

{% endhighlight %}