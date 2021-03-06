# White House Salaries: 2012 to 2019

This repo contains the code and data for compiling White House salaries from 2012 through 2019.

The wrangled data is published on this repo in several formats:

- CSV [data/wrangled/white_house_salaries.csv](data/wrangled/white_house_salaries.csv). 
- SQLite3 [data/white_house_salaries.sqlite](data/white_house_salaries.sqlite)
- Google Sheets (this is not automatically updated with the repo): https://docs.google.com/spreadsheets/d/1dVjbdr7PzsmJ36WemlXWyZ2h-xuO9d0S7BFPzdIEvYI/edit#gid=0



> ## DISCLAIMER
> 
> This data has not been bulletproofed or verified beyond the work of extracting and collating from the original files, a process which is definitely not immune to human error from myself. 
> 
> Use/analyze/publish it at your own risk.



Besides compiling the multiyear dataset into a single file, the wrangled data has a few extra convenience fields:

- `president` to make it easier to pivot by administration
- the `name` field is parsed into `last_name`, `first_name`, `middle_name`, and `suffix`

A pivot summary of the data so far (not adjusted for inflation):

| Row Labels  |  employees   | Average salary | Total salaries|
| ----------- | ------------ | ---------------| ------------- |
| 2012        |          468 |       80,843   |    37,834,589 |
| 2013        |          460 |       82,303   |    37,859,780 |
| 2014        |          456 |       82,844   |    37,776,925 |
| 2015        |          474 |       84,864   |    40,225,595 |
| 2016        |          472 |       84,223   |    39,753,551 |
| 2017        |          377 |       94,872   |    35,766,744 |
| 2018        |          374 |       94,246   |    35,248,194 |
| 2019        |          418 |       98,766   |    41,284,244 |
| Total       |        3,499 |       87,382   |   305,749,622 |


# Example usecases and queries


### Salary changes and turnover, from 2017 to 2019

Given a multi-year list of salaries and job titles, it's always interesting to look up how salaries changed, i.e. who got a raise from year X to year Y. And relatedly, who left employment from year X to year Y. We can do this in one **SQLite** query; first, a screenshot of what the results look like for the years 2017 and 2019 (the `NULL` values in `salary_change` indicate 2017 employee *names* that don't appear in the 2019 data). First, a screenshot preview of the results:

<img src="assets/images/sample-2017-2019-turnover.png" alt="sample-2017-2019-turnover.png">

To do this:

- Download and install [DB Browser for SQLite](https://sqlitebrowser.org/) (if you don't already have a SQLite GUI)
- Download from this repo the [data/white_house_salaries.sqlite](https://github.com/storydrivendatasets/white_house_salaries/raw/master/data/white_house_salaries.sqlite) file
- Open `white_house_salaries.sqlite` in DB Browser
- Then run the following query (change out `2017` and `2019` for whatever years you prefer):


```sql
WITH tx AS (
        SELECT 
            full_name
            , position_title
            , salary
        FROM employee
        WHERE year = '2017'
    ), ty AS (
        SELECT 
            full_name
            , position_title
            , salary
        FROM employee
        WHERE year = '2019'
)

SELECT
    tx.full_name AS full_name
    , ty.salary - tx.salary AS salary_change
    , tx.salary AS "x_salary"
    , ty.salary AS "y_salary"
    , tx.position_title AS "x_job"
    , ty.position_title AS "y_job"
FROM tx
LEFT JOIN ty ON
    tx.full_name = ty.full_name
ORDER BY
    "salary_change" DESC
    , tx.salary DESC
    , full_name ASC
;
```



# Inventory


The list of original data URLs and their corresponding years can be found in [data_inventory.csv](data_inventory.csv).

For safekeeping the original files are kept in [data/collected](data/collected).


## Scripts and code

All the code (mostly Python 3.x scripts) used to organize and wrangle the data can be found in the [whsa/](whsa/) subdirectory:

- [whsa/collect_data.py](whsa/collect_data.py): the **collect** phase, simply fetching the data files from their original sources and saving them exactly as is, whether they come as CSV, PDF, or ZIP.

- [whsa/convert](whsa/convert): the **convert** phase, which is basically the steps needed to get the collected data into CSV format. Very few changes to the actual data are made in this phase; the point is to handoff CSV files so that the subsequent phases don't have to worry about it.

- [whsa/fuse.py](whsa/fuse.py): the **fuse** phase, in which we attempt to unify all the data in a single CSV file, with common headers. Again, few changes to the actual data are made, though we do alter the data structure and scheme (e.g. standardizing header names)

- [whsa/wrangle.py](whsa/wrangle.py): where the real data transformation is done

TKTODO: explain the other files




# Developing fun 

To clone this repo and change things:

```sh
$ git clone https://github.com/storydrivendatasets/white_house_salaries.git
$ cd white_house_salaries
$ pip install -e .
```

Read the [Makefile](Makefile) to see some sparse documentation on how the data pipeline works. 

To rebuild the sqlite database from scratch:

```sh
$ make ALL
```



# About PDFs and CSVs

The Trump White House has chosen to publish its salaries list as PDF – you can see this in the 2017.pdf, 2018.pdf, and 2019.pdf files in [data/collected/originals](data/collected/originals). Converting the PDF documents into usable CSV data was by far the hardest and time-consuming part of this mini-project.

In [data/collected/originals/abbyy](data/collected/originals/abbyy), you can see my attempts to use ABBYY FineReader, which in the past has been my go-to tool (maybe the only commercial data tool I pay for). It did quite badly. Not only (understandably) making mistakes when trying to figure out the tabular layout, but also producing OCR-quality (i.e. **bad**) text, despite the PDFs being native text documents.

I then tried [tabula](https://github.com/tabulapdf/tabula-java), which is a solid tool but I haven't used much in the past because of ABBYY. It did a good job as expected in getting the text out, but when skimming the results, I saw more table-structure-mistakes than I wanted to deal with. 

I ended up using `pdftotext -layout`, a hacky trick [I used back in the Dollars for Docs days](https://www.propublica.org/nerds/turning-pdfs-to-text-doc-dollars-guide). Surprisingly, it worked very well, and seems to have correctly parsed all but ~15 of ~3500 rows. 

In [collected/handcleaned](collected/handcleaned) are the hand-cleaned text files for the pdf files for years 2017 through 2019. This handcleaned version of the data is what the rest of the data-processing pipeline uses, e.g. [whsa/collect/pdftotext_to_csv.py](whsa/collect/pdftotext_to_csv.py). 

I kept a log of the manual handcleaned changes, in YAML format: [data/handcleaned.log.yaml](data/handcleaned.log.yaml)


In conclusion: f–-k pdfs


## Other readings


[Forbes: Trump's Lean White House 2018 Payroll On-Track To Save Taxpayers $22 Million](https://www.forbes.com/sites/adamandrzejewski/2018/06/29/trumps-lean-white-house-2018-payroll-on-track-to-save-taxpayers-22-million/#1245d5094e4f)

This article, while authored by someone from OpenTheBooks.com who has a curious interpretation of things, does have the same numbers that I have:

- 2017: 377 employees, $35.7M total payroll, $89K median salary
- 2018: 374 employees, $35.2M total payroll, $84.4K median salary

They also have this chart:

<a href="https://www.forbes.com/sites/adamandrzejewski/2018/06/29/trumps-lean-white-house-2018-payroll-on-track-to-save-taxpayers-22-million">
    <img src="https://thumbor.forbes.com/thumbor/960x0/https%3A%2F%2Fblogs-images.forbes.com%2Fadamandrzejewski%2Ffiles%2F2018%2F06%2FForbes_TrumpVSTrump_payroll.jpg" alt="forbes chart">
</a>

There's some Forbes-contributor-LOL-quality analysis too:

> The rest of the Trump family is also leading by example by foregoing their salaries. First Daughter and Presidential Advisor Ivanka Trump and Son-in-Law Senior Advisor Jared Kushner both refused a salary.

> Although the White House personnel budget is an infinitesimal part of the $3.9 trillion federal budget, it is an important forecasting indicator showing Trump’s deep commitment to cut the size, scope and power of the federal government and reign in waste.


Here's a 2019 Followup: [Trump's Leaner White House 2019 Payroll Has Already Saved Taxpayers $20 Million](https://www.forbes.com/sites/adamandrzejewski/2019/06/28/trumps-leaner-white-house-2019-payroll-has-already-saved-taxpayers-20-million/)
