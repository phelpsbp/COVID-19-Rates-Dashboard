# COVID Public Health Analysis

## Table of Contents

- [Project Overview](#project-overview)
- [Data Sources](#data-sources)
- [Data Cleaning and Preparation](#data-cleaning-and-preparation)
- [Exploratory Data Analysis](#exploratory-data-analysis)
- [Data Analysis](#data-analysis)
- [Table Merging](#table-merging)
- [Results and Findings](#results-and-findings)
- [Deliverables](#deliverables)

  
### Project Overview
---

In this project I am exploring international and domestic public health data related to covid deaths and vaccinations from Jan. 2020 - Oct. 2021, and creating a dashboard in Tableau visualizing the results.

### Data Sources

* [CovidDeaths](https://github.com/AlexTheAnalyst/PortfolioProjects/blob/2dbf63f2f2e8f7c3ff458abc8dc90ddd555f3e38/CovidDeaths.xlsx)
* [CovidVaccinations](https://github.com/AlexTheAnalyst/PortfolioProjects/blob/2dbf63f2f2e8f7c3ff458abc8dc90ddd555f3e38/CovidVaccinations.xlsx)


### Tools

Tools Used:
* Excel - Data Cleaning
* Microsoft's SQL Management Studio - Data Analysis
* Tableau - Dashboard

---
## Data Cleaning and Preparation

In the initial data preparation phase, we performed the following tasks:

1. Filtered for nulls, mispelled wording, and innapropriate spacing.
2. Removed unnecessary columns using `SELECT`, `CTRL`+`ALT`+`RIGHT CLICK`, `DELETE`.
3. Moved `population` next to `date` and `location` using `CUT` and `INSERT`

## Exploratory Data Analysis

EDA involved exploring the locational data to answer key questions, such as:

- What was the likelihood of dying if you contracted Covid in your country?
- How much of the population contracted Covid?
- Which countries had the highest infection rates?
- Which countries had the highest death counts?

## Data Analysis


### COVID Data by Country

First, I am going to look at total deaths, total cases, highest infection rates, and highest death counts by country. 
#### Total Cases vs. Total Deaths

```sql
select location, date, total_cases, total_deaths, 
(total_deaths/total_cases)*100 as DeathPercentage
from PortfolioProject..CovidDeaths$
where location like '%states%'
and continent is not null
order by 1,2
```

#### Total Cases vs. Population
```sql
select location, date, total_cases, population, 
(total_cases/population)*100 as PercentPopulationInfected
from PortfolioProject..CovidDeaths$
where location like '%states%'
order by 1,2 
```

#### Infection Rates vs. Population

```sql
select location, population, max(total_cases) as HighestInfectionCount, 
Max((total_cases/population))*100 as PercentPopulationInfected
from PortfolioProject..CovidDeaths$
-- Where location like '%states%'
group by location, population
order by PercentPopulationInfected desc
```
#### Highest Death Counts
To calculate for maximum deaths, I converted `total_deaths` from nvarchar(255) to an interger using the `CAST` function.
```sql
select location, max(cast(total_deaths as int)) as TotalDeathCount
from PortfolioProject..CovidDeaths$
-- Where location like '%states%'
where continent is not null
group by location
order by TotalDeathCount desc
```

### COVID Data by Continent
I ran the same queries from examining the country data, but set the `SELECT` function to `continent` instead of `location`, and used `GROUP BY` to sort the data by continent instead of location (country).

## Table Merging
With the data obtained from the analysis, I'll be joining both datasets on `continent`, `location`, `date`, and `population`, as well as adding `new_vaccinations` to our new master dataset. 
```sql
select dea.continent, dea.location, dea.date, dea.population, vac.new_vaccinations,
sum(convert(int, vac.new_vaccinations)) OVER (Partition by dea.location 
order by dea.location,dea.date) as RollingPeopleVaccinated
from PortfolioProject..CovidDeaths$ dea
join PortfolioProject..CovidVaccinations$ vac 
on dea.location = vac.location
and dea.date = vac.date
where dea.continent is not null
order by 2,3;
```

### CTE Table
To avoid getting the total sum of vaccinations and calculate a rolling count to represent vaccinations continuing over time, I partitioned by location **and** date. 
```sql
With PopvsVac (Continent, Location, Date, Population, New_Vaccinations, 
RollingPeopleVaccinated) as
(
select dea.continent, dea.location, dea.date, dea.population, vac.new_vaccinations, 
sum(convert(int, vac.new_vaccinations)) OVER (Partition by dea.location 
order by dea.location, dea.date) as RollingPeopleVaccinated
--, (RollingPeopleVaccinated/population)*100
from PortfolioProject..CovidDeaths$ dea
join PortfolioProject..CovidVaccinations$ vac
on dea.location = vac.location
and dea.date = vac.date
where dea.continent is not null
)
select *, (RollingPeopleVaccinated/Population)*100
from PopvsVac
```
### Temp Table
The information resulted in an unorganized mess. To resolve this I tried commenting out "where `dea.continent` is not null" which resulted in an error due to the table already existing, whoops! Let's go ahead and use the `DROP TABLE` function to fix this. 
```sql
Drop Table if exists #PercentPopulationVaccinated
Create Table #PercentPopulationVaccinated
(
Continent nvarchar(255),
Location nvarchar(255),
Date datetime, 
Population numeric,
New_Vaccinations numeric,
RollingPeopleVaccinated numeric
)

insert into #PercentPopulationVaccinated
select dea.continent, dea.location, dea.date, dea.population, vac.new_vaccinations,
sum(convert(int, vac.new_vaccinations)) OVER (Partition by dea.location 
order by dea.location, dea.date) as RollingPeopleVaccinated
--, (RollingPeopleVaccinated/population)*100
from PortfolioProject..CovidDeaths$ dea
join PortfolioProject..CovidVaccinations$ vac
on dea.location = vac.location
and dea.date = vac.date
--where dea.continent is not null
--order by 2,3
select *, (RollingPeopleVaccinated/Population)*100
from #PercentPopulationVaccinated
```

## Results and Findings
### Global Numbers - Total Cases vs. Total Deaths
```sql
select sum(new_cases) as total_cases, sum(cast(new_deaths as int)) as total_deaths, 
sum(cast(new_deaths as int))/sum(new_cases)*100 as DeathPercentage
from PortfolioProject..CovidDeaths$
--where location like '%states%'
where continent is not null
--group by date
order by 1,2
```
![Screenshot 2023-11-28 170811](https://github.com/phelpsbp/COVID-19-Rates-Dashboard/assets/150976820/49ca8293-c4cc-455c-a4c3-c807ca42412f)


### Continents with the Highest Death Counts
```sql
Select location, sum(cast(new_deaths as int)) as TotalDeathCount
from PortfolioProject..CovidDeaths$
--where location like '%states%'
where continent is null
and location not in ('World', 'European Union', 'International')
group by location
order by TotalDeathCount desc
```
![Screenshot 2023-11-28 170830](https://github.com/phelpsbp/COVID-19-Rates-Dashboard/assets/150976820/8a0cf728-be4f-47f8-99b6-a176b35efd95)

### Highest Infection Rates
```sql
select location, population, max(total_cases) as HighestInfectionCount, 
Max((total_cases/population))*100 as PercentPopulationInfected
from PortfolioProject..CovidDeaths$
-- Where location like '%states%'
group by location, population
order by PercentPopulationInfected desc
```
![Screenshot 2023-11-28 170902](https://github.com/phelpsbp/COVID-19-Rates-Dashboard/assets/150976820/e6fe10e9-6152-4227-abcf-9b042a883253)

### Highest Infection Rates by Country, Represented Over Time. 
```sql
select location, population, date, max(total_cases) as HighestInfectionCount, 
Max((total_cases/population))*100 as PercentPopulationInfected
from PortfolioProject..CovidDeaths$
-- Where location like '%states%'
group by location, population, date
order by PercentPopulationInfected desc
```
![Screenshot 2023-11-28 171015](https://github.com/phelpsbp/COVID-19-Rates-Dashboard/assets/150976820/8df36137-f37c-4652-9280-2e4ec51ae747)

## Deliverables


The Full, interactive Tableau Dashboard can be viewed [here](https://public.tableau.com/app/profile/brittany.everette/viz/CovidDashboard_17001768757930/Dashboard1)
![TableauDashboard](https://github.com/phelpsbp/COVID-19-Rates-Dashboard/assets/150976820/99e0e2cb-6924-4d8a-800a-9352c5796fbe)




---
