SQL 错题集：每月 18-25 岁用户播放周杰伦歌曲 Top3
错题类型：窗口函数分区粒度错误 / 聚合粒度错误
关键词：ROW_NUMBER()、PARTITION BY、GROUP BY、TopN
一句话结论：先聚合到“排名粒度”，再开窗排名；窗口分区是“排名赛区”，不是随便分组。

1. 题目口径
统计 2022 年全年，18-25 岁用户播放 周杰伦歌曲，在 每个月 的 歌曲总播放量 Top3。

关键口径：

项目	口径
统计时间	2022-01-01 <= fdate < 2023-01-01
用户范围	age BETWEEN 18 AND 25
歌手	singer_name = '周杰伦'
排名粒度	每月全体用户合计后的歌曲播放量
排名对象	歌曲，不是“用户-歌曲”
输出	每月 Top3 歌曲
2. 错误代码：一眼定位错误
sql
-- ❌ 错误SQL：算成了“每个用户每月”的 Top3，不是“每月全体用户”的 Top3
SELECT
    user_id,
    month,
    ranking,
    song_name,
    play_pv
FROM
    (
        SELECT
            user_id,
            month,
            ROW_NUMBER() OVER (
                PARTITION BY user_id, month  -- ❌ 错误：排名赛区选成了“单个用户 + 月份”
                ORDER BY play_pv DESC, song_id
            ) AS ranking,
            song_name,
            play_pv
        FROM
            t1
    ) AS t2
WHERE
    ranking <= 3
ORDER BY user_id, month, ranking;
错误关键 diff
diff
  ROW_NUMBER() OVER (
-     PARTITION BY user_id, month
+     -- 正确：先把 user_id 汇总掉，按 month, song_id, song_name 聚合
+     PARTITION BY month
      ORDER BY play_pv DESC, song_id
  ) AS ranking
diff
- SELECT user_id, month, ranking, song_name, play_pv
+ SELECT month, ranking, song_id, song_name, play_pv
3. 错误思路复盘
原 SQL 中 t1 的粒度是：

text
user_id + month + song_id + song_name
也就是说，t1 中同一首歌会按不同用户拆成多行。

错误做法直接对 t1 开窗：

sql
ROW_NUMBER() OVER (
    PARTITION BY user_id, month
    ORDER BY play_pv DESC, song_id
)
这等价于：

在“每个用户、每个月”内部，给自己的歌曲播放量排名。

所以结果是：

用户 A 每月 Top3、用户 B 每月 Top3、用户 C 每月 Top3……
而不是：每月全体 18-25 岁用户播放周杰伦歌曲的总榜 Top3。

错误结果示意
user_id	month	ranking	song_name	play_pv
A	1	1	歌X	10
A	1	2	歌Y	8
B	1	1	歌X	6
B	1	2	歌Z	5
问题：

同一首歌 歌X 在多个用户下重复出现；

排名是用户内排名，不是全体总排名；

外层还 SELECT user_id，结果口径明显变成“用户榜”，与题目“歌曲总榜”冲突；

没有先把同一首歌在不同用户上的播放量汇总，导致总榜失真。

注意：t1 中 GROUP BY user_id, month(fdate), song_name, song_id 本身不是错，它只是“用户-月-歌曲”明细聚合。错在后续没有二次汇总到“月-歌曲”粒度。

4. 正确思路
正确链路：

text
原始播放明细
  -> t1：user_id + month + song_id 的播放量
  -> t2：去掉 user_id，按 month + song_id + song_name 汇总总播放量
  -> t3：按 month 分区，对歌曲总播放量排名
  -> 筛选 ranking <= 3
核心两步：

先聚合到排名粒度：GROUP BY month, song_id, song_name，用 SUM(play_pv) 汇总；

再开窗排名：PARTITION BY month，因为题目要的是“每月全体用户”的歌曲榜。

5. 修正 SQL
sql
WITH t1 AS (
    -- 原题已有的“用户-月-歌曲”粒度播放量
    -- 保留原题 t1 逻辑：按 user_id, month(fdate), song_name, song_id 分组
    SELECT
        user_id,
        month,
        song_id,
        song_name,
        play_pv
    FROM (
        -- 此处粘贴原题中生成 t1 的子查询
        -- 条件包含：
        -- U.age BETWEEN 18 AND 25
        -- AND P.fdate >= '2022-01-01'
        -- AND P.fdate < '2023-01-01'
        -- AND S.singer_name = '周杰伦'
        -- GROUP BY user_id, month(fdate), song_name, song_id
        ...
    ) AS base
),
t2 AS (
    -- ✅ 关键修正1：去掉 user_id，汇总到“月-歌曲”粒度
    SELECT
        month,
        song_id,
        song_name,
        SUM(play_pv) AS play_pv
    FROM t1
    GROUP BY
        month,
        song_id,
        song_name
),
t3 AS (
    -- ✅ 关键修正2：按 month 分区，排名范围是“每月全体歌曲”
    SELECT
        month,
        song_id,
        song_name,
        play_pv,
        ROW_NUMBER() OVER (
            PARTITION BY month
            ORDER BY play_pv DESC, song_id
        ) AS ranking
    FROM t2
)
SELECT
    month,
    ranking,
    song_id,
    song_name,
    play_pv
FROM t3
WHERE ranking <= 3
ORDER BY month, ranking;
正确结果示意
month	ranking	song_name	play_pv
1	1	歌X	100
1	2	歌Y	88
1	3	歌Z	76
2	1	歌Y	120
2	2	歌X	95
2	3	歌W	80
6. 错误 vs 正确对比
环节	❌ 错误做法	✅ 正确做法	后果
聚合粒度	保留 user_id	去掉 user_id，按 month, song_id, song_name 汇总	同一歌曲被拆成多行
窗口分区	PARTITION BY user_id, month	PARTITION BY month	排名赛区变成单个用户
排名对象	用户内歌曲排名	每月全体歌曲排名	结果不是全体 Top3
输出字段	输出 user_id	输出 month, ranking, song_name, play_pv	结果口径与题目冲突
播放量口径	用户级播放量直接排名	SUM(play_pv) 后再排名	总榜播放量不真实
7. 自检清单
□ 排名对象是谁？本题是“歌曲”，不是“用户-歌曲”。
□ PARTITION BY 是否等于题目要求的排名赛区？本题只应按 month 分区。
□ 是否先 GROUP BY 到排名粒度，再进行 ROW_NUMBER()？
□ 输出字段是否与排名粒度一致？全体歌曲榜不应输出单个 user_id。
□ 并列排序是否稳定？建议 ORDER BY play_pv DESC, song_id。
□ 如果统计跨年，月份是否应使用 DATE_FORMAT(fdate, '%Y-%m') 而不是仅 MONTH(fdate)？
8. 一句话记忆
ROW_NUMBER() 的 PARTITION BY 是“排名赛区”。
本题参赛选手是“每月每首歌”，赛区是“月份”，不是“用户”。
先聚合到排名粒度，再开窗排名。
