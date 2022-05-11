# COVID-19 Cases, Deaths, and Vaccinations

This project is an exploration of open-source pandemic data. The data is a global summary of daily Covid-19 data over the past two years and includes raw numbers of cases, deaths, and vaccinations. Data is organized by country, continent, and date to provide a wide-lens picture of how the pandemic is progressing. 

**Citation:** Hannah Ritchie, Edouard Mathieu, Lucas Rodés-Guirao, Cameron Appel, Charlie Giattino, Esteban Ortiz-Ospina, Joe Hasell, Bobbie Macdonald, Diana Beltekian and Max Roser (2020) - "Coronavirus Pandemic (COVID-19)". Published online at OurWorldInData.org. Retrieved from: 'https://ourworldindata.org/coronavirus' [Online Resource]

<h3>SQL Exploration</h3>
To begin, data was organized into tables in MS Excel and then loaded into BigQuery for data exploration with SQL. Here, both uploaded tables 'covid_deaths' and 'covid_vaccinations' are being checked for accuracy and completeness. They are each ordered by the 3rd and 4th columns, continent and location.<br><br>

```sql
SELECT * 
FROM
`covid-deaths-348600.covid_data.covid_deaths`
ORDER BY 
3, 4
```

```sql
SELECT * 
FROM
`covid-deaths-348600.covid_data.covid_vaccinations`
ORDER BY 
3, 4           
```

Here, the data that will be used in the analysis is selected (cases, deaths, and population metrics).

```sql
SELECT 
location, date, total_cases, new_cases, total_deaths, population
FROM
`covid-deaths-348600.covid_data.covid_deaths`
ORDER BY 
1, 2
```

Now, total cases are evaluated against total deaths. This table shows the likelihood of death after infection by country. Here, the United States is selected to specifically look at data from this country.

```sql
SELECT 
location, date, total_cases, total_deaths, (total_deaths/total_cases)*100 AS death_rate
FROM
`covid-deaths-348600.covid_data.covid_deaths`
WHERE 
location = "United States"
ORDER BY 
1, 2 
```

Now, total cases are evaluated against population with a calculation to approximate the infection rate in the United States.

```sql
SELECT 
location, date, population, total_cases, (total_cases/population)*100 AS infection_rate
FROM
`covid-deaths-348600.covid_data.covid_deaths`
WHERE 
location = "United States"
ORDER BY 
1, 2 
```

Next, we can look at which countries have the highest infection rates by asking BigQuery to order the table by infection rate.

```sql
SELECT
location, population, MAX(total_cases) AS highest_infections, MAX(total_cases/population)*100 AS infection_rate
FROM
`covid-deaths-348600.covid_data.covid_deaths`
GROUP BY 
location, population
ORDER BY 
infection_rate desc
```

What countries have the highest total death counts? We can use the MAX function to ask BigQuery to show the highest numbers. A WHERE statement is included here to exclude some categorical rows of data that list metrics by continent.

```sql
SELECT
location, population, MAX(total_deaths) AS total_death_count
FROM
`covid-deaths-348600.covid_data.covid_deaths`  
WHERE 
continent is not null
GROUP BY 
location, population
ORDER BY 
total_death_count desc
```

We can also look at the data by continent; This query returns a table of total death counts by continent. We can use the MAX clause because the data is updated regularly and keeps a tally of the running total of deaths. The WHERE clauses here are filtering out additional categorical data aggregating countries by socioeconomic status.

```sql
SELECT
location, MAX(total_deaths) AS total_death_count
FROM
`covid-deaths-348600.covid_data.covid_deaths`  
WHERE 
continent is null AND
location != "Upper middle income" AND
location != "High income" AND
location != "Lower middle income" AND
location != "Lower income" AND
location != "Low income" AND
location != "International" 
GROUP BY 
location
ORDER BY 
total_death_count desc
```

We can also look at global summary statistics like the aggregate number of cases and deaths, as well as the global death rate.

```sql
SELECT 
SUM(new_cases) as total_cases, SUM(new_deaths) as total_deaths, SUM(new_deaths)/SUM(new_cases)*100 as global_death_rate
FROM
`covid-deaths-348600.covid_data.covid_deaths`
WHERE 
continent is not null
ORDER BY 
1, 2 
```

Let's break down the global summary statistics into the daily numbers to better understand how the pandemic is progressing over time. The data here begins on January 1, 2020 and continues daily until the present day.

```sql
SELECT 
date, SUM(new_cases) as daily_new_cases, SUM(new_deaths) as daily_new_deaths, SUM(new_deaths)/SUM(new_cases)*100 as global_death_rate
FROM
`covid-deaths-348600.covid_data.covid_deaths`
WHERE 
continent is not null
GROUP BY date
ORDER BY 
1, 2 
```

Now we have a better understanding of cases and deaths, let's look at vaccination and testing data. This data emerged later on in the pandemic as tests and vaccines became available.

```sql
SELECT * 
FROM
`covid-deaths-348600.covid_data.covid_vaccinations`
WHERE
continent is not null
ORDER BY 
3, 4
```

Let's join these tables so we can get a look at all the data in BigQuery, excluding categorical values for socioeconomic status and continent. We will give each table an alias to keep the query from becoming too unwieldy.

```sql
SELECT *
FROM
`covid-deaths-348600.covid_data.covid_deaths` as dea
JOIN 
`covid-deaths-348600.covid_data.covid_vaccinations` as vac
  ON dea.location = vac.location
  and dea.date = vac.date
WHERE
dea.continent is not null
```

Now we can look at columns from either table. Let's look at total population data versus vaccinations and make a new column to keep a running tally of vaccinations:

```sql
SELECT dea.continent, dea.location, dea.date, dea.population, vac.new_vaccinations, SUM(vac.new_vaccinations) OVER (PARTITION BY dea.location ORDER BY dea.location, dea.date) AS running_total_vaccinations
FROM
`covid-deaths-348600.covid_data.covid_deaths` as dea
JOIN 
`covid-deaths-348600.covid_data.covid_vaccinations` as vac
  ON dea.location = vac.location
  and dea.date = vac.date
WHERE
dea.continent is not null AND
--dea.location = "United States"
ORDER BY
2, 3
```

In order to calculate vaccination as a percentage of population with the running_total_vaccinations column, we need to create a temp table:

```sql
DROP TABLE IF EXISTS `covid-deaths-348600.covid_data.PercentPopulationVaccinated`;
CREATE TABLE `covid-deaths-348600.covid_data.PercentPopulationVaccinated`
  (
  continent string,
  location string,
  date date,
  population bigint,
  new_vaccinations bigint,
  running_total_vaccinations bigint
  )
  ;
INSERT INTO `covid-deaths-348600.covid_data.PercentPopulationVaccinated`
SELECT dea.continent, dea.location, dea.date, dea.population, vac.new_vaccinations, SUM(vac.new_vaccinations) OVER (PARTITION BY dea.location ORDER BY dea.location, dea.date) AS running_total_vaccinations
FROM
`covid-deaths-348600.covid_data.covid_deaths` as dea
JOIN 
`covid-deaths-348600.covid_data.covid_vaccinations` as vac
  ON dea.location = vac.location
  and dea.date = vac.date
WHERE dea.continent is not null 
;
SELECT*, (running_total_vaccinations/population)*100
FROM `covid-deaths-348600.covid_data.PercentPopulationVaccinated`
```

<h3>Data from the following queries will be downloaded for use in Tableau visualizations:</h3>
<b>Tableau Query 1:</b> Worldwide summary of cases, deaths, and death rate<br><br>

```sql
SELECT 
SUM(new_cases) as total_cases, SUM(new_deaths) as total_deaths, SUM(new_deaths)/SUM(new_cases)*100 as global_death_rate
FROM
`covid-deaths-348600.covid_data.covid_deaths`
WHERE 
continent is not null
ORDER BY 
1, 2 
```

**Tableau Query 2:** Summary of deaths by continent

```sql
SELECT
location, MAX(total_deaths) AS total_death_count
FROM
`covid-deaths-348600.covid_data.covid_deaths`  
WHERE 
continent is null AND
location != "Upper middle income" AND
location != "High income" AND
location != "Lower middle income" AND
location != "Lower income" AND
location != "Low income" AND
location != "International" AND
location != "World" AND
location != "European Union"
GROUP BY 
location
ORDER BY 
total_death_count desc
```

**Tableau Query 3:** Infection rate overall by country

```sql
SELECT
location, population, MAX(total_cases) AS highest_infections, MAX(total_cases/population)*100 AS infection_rate
FROM
`covid-deaths-348600.covid_data.covid_deaths`
GROUP BY 
location, population
ORDER BY 
infection_rate desc
```

**Tableau Query 4:** Infection rate by country and date

```sql
SELECT
location, population, date, MAX(total_cases) AS highest_infections, MAX(total_cases/population)*100 AS infection_rate
FROM
`covid-deaths-348600.covid_data.covid_deaths`
GROUP BY 
location, population, date
ORDER BY 
infection_rate desc
```


