# BigQuery UI Tutorial

The aim of this tutorial is to get you familiarized with BigQuery web UI to query/filter/aggregate/export data.

## Prerequisites

* You should already have had a valid Gmail account registered with the instructors or event organizers.
  * If you do not have a Gmail account, you can create one at http://www.gmail.com. 
  You need to notify the instructors or organizers to register your new account for data access.
  * If you have not yet signed the data use agreement (DUA) sent by the instructors or organizers, please do so immediately to get access to the datasets.

## Executing Queries

The data-hosting project `physionet-data` has read-only access, as a result, you need to set a default project that you have BigQuery access to. 
A shared project `team-00` has already been created, visit **https://bigquery.cloud.google.com/welcome/team-00** to access the BigQuery web interface, all the queries in this tutorial will be run through this web interface.

Note that during the class or event, all students/participants will be divided into teams and a Google Cloud project will be created for each team specifically. 
That project would be the preferred project to use. 
For now we'll stick with the shared project for the purpose of the tutorial.

## TLDR

In this section we are going to run a query to briefly showcase BigQuery's capability.

Run the following query from BigQuery web interface (See "Executing Queries" section above for how to access BigQuery web interface) to find the correlation between age and the average length of stay in hours in ICU.

![Run a query](img/run_query.png)

```SQL
#standardSQL
WITH re AS (
SELECT
  DATETIME_DIFF(icu.outtime, icu.intime, HOUR) AS icu_length_of_stay,
  DATE_DIFF(DATE(icu.intime), DATE(pat.dob), YEAR) AS age
FROM `physionet-data.mimic_demo.icustays` AS icu
INNER JOIN `physionet-data.mimic_demo.patients` AS pat
  ON icu.subject_id = pat.subject_id)
SELECT
  age,
  AVG(icu_length_of_stay) AS stay
FROM re
WHERE age < 100
GROUP BY age
ORDER BY age
```

You can download the returned result as a CSV file and processed the table with your preferred tools.

![Download result as a CSV file](img/save_csv.png)

## Detailed Tutorial

### BigQuery Basics

Feel free to skip this section if you are already familiar with BigQuery.

#### BigQuery Table Name

A BigQuery table is uniquely identified by the three-layer hierarchy of project ID, dataset ID and table name. For example in the following query:

```SQL
SELECT
  subject_id
FROM
  `physionet-data.mimic_demo.icustays`
LIMIT 10
```

`physionet-data.mimic_demo.icustays` specifies the table we are querying, where `physionet-data` is the project that hosts the datasets, `mimic_demo` is the name of the dataset, and `icustays` is the table name. Backticks (`) are used as there is a non-standard character (-) in the project name. If the dataset resides in the same project, you can safely omit the project name, e.g. `my-project.my_dataset.my_table` can be written as `my_dataset.my_table` instead.

#### SQL Dialect

BigQuery supports 2 SQL dialects, legacy and standard. We recommend using standard SQL dialect.

**Follow the steps below to make sure the StandardSQL dialect is used**:

1. Click "COMPOSE QUERY" on top left corner;
2. Click "Show Options" below the input area;
3. Lastly, make sure "Use Legacy SQL" is **NOT** checked, and click "Hide Options".

![Uncheck "Use Legacy SQL"](img/dialect.png)

Alternatively, ["#standardSQL" tag](https://cloud.google.com/bigquery/docs/reference/standard-sql/enabling-standard-sql) can be prepended to each query to tell BigQuery the dialect you are using, which is what we used in the TLDR section above.

### Example: Using MIMIC-III

#### Dataset Exploration

As mentioned previously, the datasets are hosted in a different project. For example, MIMIC-III demo dataset can be accessed [here](https://bigquery.cloud.google.com/dataset/physionet-data:mimic_demo). On the left panel, you will see the `mimic_demo` dataset, under which you will see the table names.

To view the details of a table, simply click on it (for example the `icustays` table). Then, on the right side of the window, you will have to option to see the schema, metadata and preview of rows tabs.

#### Queries

Most of the following queries are adapted from the [MIMIC cohort selection tutorial](https://github.com/MIT-LCP/mimic-code/blob/master/tutorials/cohort-selection.ipynb).

#### Analysis

Let's take a look at a few queries. To run the queries yourself, copy the SQL statement to the input area on top of the web interface and click the red "RUN QUERY" button.

```SQL
SELECT
  subject_id,
  hadm_id,
  icustay_id,
  intime,
  outtime,
  DATETIME_DIFF(outtime, intime, DAY) AS icu_length_of_stay
FROM `physionet-data.mimic_demo.icustays`
```

Let's save the result of previous query to an intermediate table for later analysis:

1. Create a dataset by clicking the caret below the search box on the left sidebar, and choose "Create new dataset";
  * Set dataset ID to "temp" and data expiration to 2 days;
  * Click "OK" to save the dataset.
2. Click "Save to table" button on the right;
  * Set destination dataset to "temp" and table to "icustays", use the default value for project;
  * Click "OK" to save the table, it usually takes less than a few seconds for demo tables.

![Create a dataset](img/create_dataset.png)

Now let's take a look at a query that requires table joining: include the patient's age at the time of ICU admittance. This is computed by the date difference in years between the ICU intime and the patient's date of birth. The former is available in the icustays table, and the latter resides in the dob column of the patients table.

```SQL
SELECT
  icu.subject_id,
  icu.hadm_id,
  icu.icustay_id,
  pat.dob,
  icu.icu_length_of_stay,
  DATE_DIFF(DATE(icu.intime), DATE(pat.dob), YEAR) AS age
FROM `physionet-data.mimic_demo.patients` AS pat
INNER JOIN `temp.icustays` AS icu
  ON icu.subject_id = pat.subject_id
```

Again, let's save the table as "pat_icustays" in the "temp" dataset for use later. Briefly look at the age of patients when they are admitted with the following query.

Now let's run the following query to produce data to generate a histrogram graph to show the distribution of patient ages in ten-year buckets (i.e. [0, 10), [10, 20), ..., [90, ∞).

```SQL
WITH bu AS (
  SELECT
    CAST(FLOOR(ps.age / 10) AS INT64) AS bucket
  FROM `temp.pat_icustays`)
SELECT
  IF(bucket >= 9, ">= 90", FORMAT("%d - %d", bucket * 10, (bucket + 1) * 10)) AS age,
  COUNT(bucket) AS total
FROM bu
GROUP BY bucket
ORDER BY bucket ASC
```

Now click "Save to Google Sheets" button and wait 1-2 seconds until a yellow notification shows up, click "Click to view" which leads you to Google Spreadsheet in a new browser window. As you can see, the data from our last query is dumped into a spreadsheet. By clicking "Insert -> Chart" from the menu bar on top, a nice histrogram graph is automatically created for us!

![Save to Google Sheets](img/save_to_sheets.png)

If you prefer using other tools to process the final result, a CSV file can be downloaded by clicking the "Downed as CSV" button. 

If downloading fails because the file is too large (we highly recommend aggregating the data to a small enough result before downloading though), you can save it to a temporary table, click the caret then "Export table" button from the dropdown menu and save it to Google Cloud Storage, then you can download the file from [GCS](https://console.cloud.google.com/storage).

![Save to Google Cloud Storage](img/save_to_gcs.png)

Now let's see if there is correlation between age and average length of stay in hours. Since we are using the age of patients when they get admitted, so we don't need to worry about multiple admissions of patients. Note that we treat the redacted ages (> 90) as noises and filter them out.

```SQL
WITH re AS (
SELECT
  DATETIME_DIFF(icu.outtime, icu.intime, HOUR) AS icu_length_of_stay,
  DATE_DIFF(DATE(icu.intime), DATE(pat.dob), YEAR) AS age
FROM `physionet-data.mimic_demo.icustays` AS icu
INNER JOIN `physionet-data.mimic_demo.patients` AS pat
  ON icu.subject_id = pat.subject_id)
SELECT
  icu_length_of_stay AS stay,
  age
FROM re
WHERE age < 100
```

Follow the same steps to save the result to Google Spreadsheet, by default a linear chart is generate. We will need to change the chart type to scatter chart through the chart editor on the right.

If you would like to explore the real data, please use ` mimiciii_clinical` as the dataset name. 
For example, the table `mimic_demo.icustays` becomes `mimiciii_clinical.icustays` when you need the actual MIMIC data.

Please take a look at more comprehensive examples [here](tutorial_py.ipynb) such as creating charts and training machine learning models in an interactive and fully online way (or copy the queries over to web UI to execute if you prefer) if you are interested.