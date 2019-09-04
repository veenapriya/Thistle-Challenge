# Thistle-Challenge

Research: 
I looked into Thistle website on how users can sign up the app and build their plan for a time duration. Users can pause and resume their food plan based on their needs. 

Checkout workflow: 
1.	User sign in  
2.	Select Protein preference(Veg or meat) 
3.	Select the meals (Breakfast, lunch or dinner) 
4.	Select number days per week 
5.	Select any allergies 
6.	Select any boosters
7.	Set your address details 
8.	Fill the payment details.

# 1.	Please write a query for how many customers started the subscription flow each month as well as the number and % that completed.

Who could be the stakeholders asking the information?
- Product, Marketing - Try out different food plans through to A/B tests and find the most efficient food plan for users.
- UX - Try to identify the reasons why users are not completing the checkout flow.

Query:

WITH initializeddate (initialized,MONTH,YEAR)
AS
(SELECT COUNT(DISTINCT user_id)::DECIMAL AS initialized,
       DATE_PART('month',date_created::DATE) AS MONTH,
       DATE_PART('year',date_created::DATE) AS YEAR
FROM thistle_web.subscriptions_subscription
GROUP BY DATE_PART('month',date_created::DATE),
         DATE_PART('year',date_created::DATE)),
       
Completeddate(Completed,MONTH,YEAR) AS (SELECT COUNT(DISTINCT user_id)::DECIMAL AS Completed,
                                                        DATE_PART('month',date_initialized::DATE) AS MONTH,
                                                        DATE_PART('year',date_initialized::DATE) AS YEAR
FROM thistle_web.subscriptions_subscription
WHERE date_initialized::DATE IS NOT NULL
GROUP BY DATE_PART('month',date_initialized::DATE),                                                      DATE_PART('year',date_initialized::DATE))

SELECT initializeddate.year,
       initializeddate.month,
       COALESCE(initialized,0) AS initialized,
       COALESCE(Completed,0) AS Completed,
       COALESCE((Completed / initialized),0) AS PErcentage_Completed
FROM initializeddate
  JOIN Completeddate
    ON initializeddate.month = Completeddate.month
   AND initializeddate.year = Completeddate.year
ORDER BY 1,
         2





# 2.	What is the signup success rate (# of people signing up for a subscription vs. all people who enter the checkout flow) for meat vs. veg plans?

Who could be the stakeholders asking the information?
- Marketing and sales - Team can reach out to users who have successfully signed up promote their prefered food plans. 
- Operations team - This information helps the team to understand how much food and what kind of food is consumed by users.


Query:

SELECT ((SELECT COUNT(DISTINCT user_id)::decimal
      FROM thistle_web.subscriptions_subscription
      WHERE date_initialized::DATE IS NOT NULL
      AND   protein_type = 'vegan_protein') / (SELECT COUNT(DISTINCT user_id)::decimal
      FROM thistle_web.subscriptions_subscription
      WHERE date_created::DATE IS NOT NULL
      AND   protein_type = 'vegan_protein')) as success_Rate_Veg



 SELECT ((SELECT COUNT(DISTINCT user_id)::decimal
      FROM thistle_web.subscriptions_subscription
      WHERE date_initialized::DATE IS NOT NULL
      AND   protein_type = 'animal_protein') / (SELECT COUNT(DISTINCT user_id)::decimal
      FROM thistle_web.subscriptions_subscription
      WHERE date_created::DATE IS NOT NULL
      AND   protein_type = 'animal_protein')) as success_Rate_Meat


# 3.	Please calculate how many customers cancel within 14 days of signing up.

Who could be the stakeholders asking the information?
- Product management - This helps the product team to understand why users are canceling and probe further investigations on how to improve customers experience.

SELECT COUNT(user_id) as number_of_cancellations
FROM thistle_web.subscriptions_subscription s
  JOIN thistle_web.subscriptions_subscriptioncancellation c
    ON s.id = c.subscription_id
   AND c.date_cancelled::DATE<= s.date_created::DATE+INTERVAL '14 day'
   AND s.date_to_resume IS NULL;


# 4.	Please calculate retention by weekly cohort.

Who could be the stakeholders asking the information?
- Product & Marketing teams - This parameter will help keep track of the product offering. Product and marketing team can alter the food plans based on the needs of the user.
- Operations team - Retention rate will provide data points to the team on how to plan out food resources for the upcoming week. 


SELECT result.*,
       (result.active_subs / result.cohort_total) AS ACTIVE_percent
FROM (SELECT DATE_TRUNC('month',DAY::DATE) AS COHORT,
             DATE_TRUNC('week',DAY::DATE) AS week,
             DATE_PART('week',DAY::DATE) AS week_number,
             (SELECT COUNT(*)::DECIMAL
              FROM thistle_web.subscriptions_subscription
              WHERE date_initialized::DATE< cal.day::DATE+INTERVAL '7 day') AS cohort_total,
             (SELECT COUNT(*)::DECIMAL
              FROM thistle_web.subscriptions_subscription s
              WHERE s.date_initialized::DATE< cal.day::DATE+INTERVAL '7 day'
              AND   NOT EXISTS (SELECT *
                                FROM thistle_web.subscriptions_subscriptioncancellation c
                                WHERE s.id = c.subscription_id
                                AND   c.date_cancelled::DATE< cal.day::DATE+INTERVAL '7 day')) AS active_subs
      FROM (SELECT DATE_TRUNC('week',DAY::DATE) AS DAY
            FROM etl_calendar
            WHERE DAY::DATE<= CURRENT_DATE
            GROUP BY DATE_TRUNC('week',DAY::DATE)) AS cal) AS result
ORDER BY week ASC;



