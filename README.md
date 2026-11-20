# SQL mistake 1分组top查询
WITH t1 AS (
    SELECT
        P.user_id,
        P.song_id,
        song_name,
        month(fdate) AS month,
        count(*)  AS play_pv
    FROM
        play_log AS P
        INNER JOIN song_info AS S ON P.song_id = S.song_id
        INNER JOIN user_info AS U ON P.user_id = U.user_id
    WHERE
        U.age BETWEEN 18 AND 25
        AND P.fdate >= '2022-01-01'
        AND P.fdate <'2023-01-01'
        
        AND S.singer_name = '周杰伦'
        GROUP BY user_id,month(fdate),song_name,song_id
)
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
                PARTITION BY
                    user_id,
                    month
                ORDER BY
                    play_pv DESC,
                    song_id
            ) AS ranking,
            song_name,
            play_pv
        FROM
            t1
       
    ) AS t2
WHERE
    ranking <= 3
ORDER BY user_id,month,ranking;
---错误思路是把排名窗口按 user_id, month 分区，算成了“每个用户每月”的前三，而不是“每月全体18-25岁用户播放周杰伦歌曲”的前三；
改正：为先在 t1 按 month, song_id, song_name 汇总总播放量，再用 ROW_NUMBER() OVER(PARTITION BY month ORDER BY play_pv DESC, song_id) 取前三
