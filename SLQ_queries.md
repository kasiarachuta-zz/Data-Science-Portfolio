/*
QUERY 1:
The names of the tables and columns have been altered for confidentiality purposes.

This query looks for the first visit of each user at a sample location 12345. Table user_checkins contains information on all the checkins made by each user at a particular location in GMT.
*/


WITH all_checkins AS 
    (SELECT 
        user_id,
        DATE(MIN(CONVERT_TIMEZONE('CST', created_at))) AS created_at_local
    FROM user_checkins
    WHERE business_id = 12345
    GROUP BY 1)

SELECT created_at_local,
COUNT(*)
FROM all_checkins
GROUP BY 1

/*
QUERY 2: 
This query looks up businesses, their status, provides the number of online days and the sum of daily signups and checkins that occurred after 18th July for businesses that joined after 13th July. They also need to have been active for 14 days.
*/

SELECT 
	loc.business_id,
	loc.status,
	COUNT(CASE WHEN stats.daily_checkins > 0 THEN 1 ELSE NULL END) AS online_day,
	SUM(stats.daily_signups) AS sum_daily_signups,
	SUM(stats.daily_checkins) AS sum_daily_checkins
FROM business_locations AS loc
LEFT JOIN business_stats as STATS on loc.business_id = stats.business_id
WHERE date > '2016-07-18'
AND loc.first_visit > '2016-07-13'
AND stats.total_days_active > 14
GROUP BY 1, 2


/*
This query looks up all of the active users and counts how many have more than 2 rewards they can redeem right now.
*/

-- counting active members that visited after 9th November 2016.
WITH members AS (
    SELECT 
		id AS member_id,
		business_group_id,
		points
	FROM memberships
	WHERE deactivated_on IS NULL
	AND business_group_id IN (SELECT DISTINCT business_group_id FROM locations WHERE status = 'Active')
	AND visit_count > 1
	AND last_visited_at >= '2016-09-11'),

-- choosing active rewards for each group of businesses
	
rewards AS (
	SELECT
		business_group_id,
		id AS reward_id,
		point_cost
		FROM rewards
		WHERE active = 'TRUE'),

-- joining the tables together

finaldf AS (
	SELECT 
		members.business_group_id,
		member_id,
		points,
		reward_id,
		point_cost,
		CASE WHEN points >= point_cost THEN 1 ELSE 0 END AS reward
	FROM members
	INNER JOIN rewards ON rewards.business_group_id = members.business_group_id),

-- counting rewards for each member
	
df AS (
	SELECT
		business_group_id,
		member_id,
		SUM(reward) AS sum_rewards
	FROM finaldf
	GROUP BY 1, 2)
	
-- calculating the percentage of users that have 2 or more active rewards

SELECT 1.0 * COUNT(CASE WHEN sum_rewards >= 2 THEN 1 ELSE NULL END) / COUNT(*)
FROM df

/*


*/ 
