# Assignment journal

Start date: **Friday, July 21, 2023**</br>
End date: ****

## Data source

- [FitBit Fitness Tracker Data](https://www.kaggle.com/datasets/arashnic/fitbit)
  - Public domain, dataset made available through [Mobius](https://www.kaggle.com/arashnic)
  - Acknowledgement: Furbert, Robert; Brinton, Julia; Keating, Michael; Ortiz, Alex
    - [Access to data on Zenodo](https://zenodo.org/record/53894#.YMoUpnVKiP9)

## Upload of files to BigQuery

- **dailyActivity**: No issues with automatic schema
- **dailyCalories**: No issues with automatic schema
- **dailyIntensities**: No issues with automatic schema
- **dailySteps**: No issues with automatic schema
- **heartrate_seconds**: Issues
  - *ID*: as an integer
  - *Date_Time*: as a string
    - File too big to make correction in Excel
  - *Value*: as an integer
- **hourlyCalories**: Issues
  - *ID*: as an integer
  - *ActivityHour*: Added *Act_DateTime* as datetime format
    - Within MSExcel, use `int` and `mod([value],1)` to parse out the date
    - Within MSExcel, use `text([value],"yyyy-mm-dd")&" "&text([value],"hh:mm:ss")` to put together as text
  - *Calories*: as an integer
- **hourlyIntensities**: Issues
  - *ID*: as an integer
  - *ActivityHour*: Added *Act_DateTime* as datetime format
    - Within MSExcel, use `int` and `mod([value],1)` to parse out the date
    - Within MSExcel, use `text([value],"yyyy-mm-dd")&" "&text([value],"hh:mm:ss")` to put together as text
  - *TotalIntensity*: as an integer
  - *AverageIntensity*: as a numeric
- **hourlySteps**: Issues
  - *ID*: as an integer
  - *ActivityHour*: Added *Act_DateTime* as datetime format
    - Within MSExcel, use `int` and `mod([value],1)` to parse out the date
    - Within MSExcel, use `text([value],"yyyy-mm-dd")&" "&text([value],"hh:mm:ss")` to put together as text
  - *StepTotal*: as an integer
- **minuteCaloriesNarrow**: Issues
  - *ID*: as an integer
  - *ActivityMinute*: Added as a string
  - *Calories*: as a bignumeric
- **minuteIntensitiesNarrow**: Issues
  - *ID*: as an integer
  - *ActivityMinute*: as a string
  - *Intensity*: as an integer
- **minuteMETsNarrow**: Issues
  - *ID*: as an integer
  - *ActivityMinute*: as a string
  - *METs*: as an integer
- **minuteSleep**: Issues
  - *ID*: as an integer
  - *Date*: as a string
  - *Value*: as an integer
  - *logID*: as an integer
- **minuteStepsNarrow**: Issues
  - *ID*: as an integer
  - *ActivityMinute*: as a string
  - *Steps*: as an integer
- **sleepDay**: Issues
  - *Id*: as an integer
  - *SleepDate*: Added as date format
    - Within MSExcel, use `int` to parse out the date
    - Within MSExcel, use `text([value],"yyyy-mm-dd")to put together as text
  - *TotalSleepRecords*: as an integer
  - *TotalMinutesAsleep*: as an integer
  - *TotalTimeInBed*: as an integer
  - *Calories*: as an integer
- **weightLogInfo**: Issues
- *Id*: as integer
- *Date*: as string
- *WeightKg*: as bignumeric
- *WeightPounds*: as bignumeric
- *Fat*: as bignumeric
- *BMI*: as bignumeric
- *IsManualReport*: as boolean
- *LogId*: as integer

## Processing of tables in BigQuery

The minute and second tables are too big to convert the dates in Excel. I can't convert the strings during import, so I'll need to write it into the query using `PARSE_DATETIME`.

- `PARSE_DATETIME`
  - Looks something like
    - `SELECT PARSE_DATETIME(format_string, header_of_string) AS new_header`
  - The following website has literals rather than headers
    - [`PARSE_DATETIME` guidance](https://roboquery.com/app/syntax-parse-datetime-function-bigquery)

### Step 1: Combine all minute tables into one table for use in R

Here is the final SQL query that combined the data from the various minutes tables.

```SQL
SELECT 
  calories.ID,
  PARSE_DATETIME('%m/%d/%Y %H:%M:%S %p', calories.ActivityMinute) AS DateAndTime,
  calories.Calories,
  intensities.Intensity, 
  METs.METs,
  sleep.Value AS SleepMinutes,
  steps.Steps
FROM
  `moonlit-palace-393623.Fitabase_2016.Minute_Calories` AS calories
  JOIN `moonlit-palace-393623.Fitabase_2016.Minute_Intensities` AS intensities
    ON calories.ID = intensities.ID
    AND PARSE_DATETIME('%m/%d/%Y %H:%M:%S %p', calories.ActivityMinute) = 
        PARSE_DATETIME('%m/%d/%Y %H:%M:%S %p', intensities.ActivityMinute)
  JOIN `moonlit-palace-393623.Fitabase_2016.Minute_METs` AS METs
    ON calories.ID = METs.ID
    AND PARSE_DATETIME('%m/%d/%Y %H:%M:%S %p', calories.ActivityMinute) = 
        PARSE_DATETIME('%m/%d/%Y %H:%M:%S %p', METs.ActivityMinute)
  JOIN `moonlit-palace-393623.Fitabase_2016.Minute_Sleep` AS sleep
    ON calories.ID = sleep.ID
    AND PARSE_DATETIME('%m/%d/%Y %H:%M:%S %p', calories.ActivityMinute) = 
        PARSE_DATETIME('%m/%d/%Y %H:%M:%S %p', sleep.Date)
  JOIN `moonlit-palace-393623.Fitabase_2016.Minute_Steps` AS steps
    ON calories.ID = steps.ID
    AND PARSE_DATETIME('%m/%d/%Y %H:%M:%S %p', calories.ActivityMinute) = 
        PARSE_DATETIME('%m/%d/%Y %H:%M:%S %p', steps.ActivityMinute)
```

### Step 2: Combine all daily tables into one table for use in R

Here is the final SQL query that combined the data from the various daily tables.

```SQL
SELECT 
  Activity.Id,
  Activity.ActivityDate,
  Calories.Calories,
  Intensities.SedentaryMinutes,
  Intensities.LightlyActiveMinutes,
  Intensities.FairlyActiveMinutes,
  Intensities.VeryActiveMinutes,
  Sleep.TotalMinutesAsleep,
  Sleep.TotalTimeInBed,
  Steps.StepTotal
FROM
  `moonlit-palace-393623.Fitabase_2016.Daily_Activity` AS Activity
  JOIN `moonlit-palace-393623.Fitabase_2016.Daily_Calories` AS Calories
  ON Activity.Id = Calories.Id
    AND Activity.ActivityDate = Calories.ActivityDay 
  JOIN `moonlit-palace-393623.Fitabase_2016.Daily_Intensities` AS Intensities
    ON Activity.Id = Intensities.Id
    AND Activity.ActivityDate = Intensities.ActivityDay
  JOIN `moonlit-palace-393623.Fitabase_2016.Daily_Sleep` AS Sleep
    ON Activity.Id = Sleep.Id
    AND Activity.ActivityDate = Sleep.SleepDate
  JOIN `moonlit-palace-393623.Fitabase_2016.Daily_Steps` AS Steps
    ON Activity.Id = Steps.ID
    AND Activity.ActivityDate = Steps.ActivityDay
```

### Step 3: Combine all hourly tables into one table for use in R

```SQL
SELECT
  Calories.ID,
  Calories.Act_DateTime,
  Calories.Calories,
  Intensities.AverageIntensity,
  Intensities.TotalIntensity,
  Steps.StepTotal
FROM
  `moonlit-palace-393623.Fitabase_2016.Hourly_Calories` AS Calories
  JOIN `moonlit-palace-393623.Fitabase_2016.Hourly_Intensities` AS Intensities
  ON Calories.Id = Intensities.Id
    AND Calories.Act_DateTime = Intensities.Act_DateTime
  JOIN `moonlit-palace-393623.Fitabase_2016.Hourly_Steps` AS Steps
  ON Calories.Id = Steps.Id
    AND Calories.Act_DateTime = Steps.Act_DateTime
```

## Generating graphics in R

I created an R markdown file to capture the analysis work.

[Google DA Capstone Project Analyses](Google-DA-analyses.RMD)

## Comparing with other data

I compared my activity findings with data found on the a [CDC webpage](https://www.cdc.gov/nccdphp/dnpao/data-trends-maps/index.html). The data from my analysis aligns well with the CDC findings. 
