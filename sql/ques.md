# SQL Interview Question Bank (Hirely Schema)

This question bank is designed to test SQL proficiency using the actual database schema from your project. It progresses through three levels: Single Line Queries, Subqueries, and Window/Partition functions. Each section contains a mix of Easy, Medium, and Hard questions.

---

## Level 1: Plain / Single Line Queries (Aggregations, Group By, Joins)

These questions test fundamental SQL knowledge, focusing on basic `SELECT`, `WHERE`, `GROUP BY`, and standard `JOIN` operations without complex nesting.

1. **(Easy)** Write a query to retrieve the total number of open job postings (`status = 'OPEN'`) for each company. The result should include the company name and the count of open jobs, ordered by the count in descending order.
2. **(Easy)** Fetch the full names (`first_name`, `last_name`) and emails of all candidates who have applied for at least one remote job (`work_mode = 'REMOTE'`). Ensure there are no duplicate candidates in the result.
3. **(Medium)** Write a query to find the average `salary_lower` and average `salary_higher` for job postings grouped by `JobType` (e.g., INTERN, FULL_TIME). Exclude any job postings where the salary lower bound is zero.
4. **(Medium)** Retrieve a list of all Hiring Managers (First Name, Last Name, and Department Name) who have posted jobs that currently have 0 applications. 
5. **(Hard)** Write a query to calculate the "interview conversion rate" for each company. This is defined as the total number of applications in the `'INTERVIEW'` status divided by the total number of applications for that company's jobs. Return the Company Name and the Conversion Rate percentage, rounded to two decimal places.

---

## Level 2: Subqueries & CTEs

These questions require using subqueries (in the `SELECT`, `FROM`, or `WHERE` clauses) or Common Table Expressions (CTEs) to break down multi-step logic.

1. **(Easy)** Using a subquery, find the titles and company names of all job postings whose `salary_higher` is strictly greater than the overall average `salary_higher` across the entire `job_postings` table.
2. **(Medium)** Write a query using a CTE to find candidates who possess *every single skill* required by the job posting with ID `2301`. Assume `job_skill_requirements` tracks required skills and `candidate_skills` tracks what the candidate knows.
3. **(Medium)** Find the company that has the highest average rating across all their conducted `job_rounds`. Use a subquery to first calculate the average rating per company, and then select the maximum from that list.
4. **(Hard)** Write a query to identify "Ghosted" candidates. A candidate is considered ghosted if they applied to a job more than 30 days ago, their application status is still `'APPLIED'` or `'SCREENING'`, and the job posting itself has been marked as `'CLOSED'`. Return the Candidate's Name, Email, and the Job Title.
5. **(Hard)** Using subqueries, find the names of departments within each company that have a higher number of hiring managers than the overall average number of hiring managers per department in that specific company.

---

## Level 3: Window Functions & Partitioning

These questions test advanced analytics using `OVER()`, `PARTITION BY`, `RANK()`, `ROW_NUMBER()`, `LEAD()`, and `LAG()`.

1. **(Easy)** Write a query to assign a ranking to candidates based on their total years of experience (or `minimum_experience_in_months` matched), partitioned by the role they applied for. Use `RANK()` so that candidates with the same experience get the same rank.
2. **(Medium)** For every job application that has gone through multiple `job_rounds`, write a query using `LEAD()` or `LAG()` to show the `rating` of the current round and the `rating` of the *previous* round side-by-side, ordered by the round's timestamp (`at`).
3. **(Medium)** Write a query using `ROW_NUMBER()` to return only the *most recent* job application submitted by each candidate. The result should include the Candidate Name, Job Title, and the `applied_at` date.
4. **(Hard)** Calculate the running total of job applications received by each company day-by-day for the last month. The result should show the Date, Company Name, Daily Applications, and the Cumulative (Running) Total of applications partitioned by the company and ordered by date.
5. **(Hard)** Identify the "Fastest Moving" job postings. Write a query to calculate the time difference (in days) between an application's `applied_at` date and the timestamp of its *first* `job_round`. Use window functions to find the average of this time difference partitioned by `job_posting_id`, and return the top 5 fastest jobs.
