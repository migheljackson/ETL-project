# Program Community Data ETL Project

The research project that I have been working with for the past several years is a multi-sided platform called [CitiesLearn](http://citieslearn.com/).  The platform provides several tools with the primary goal of helping stakeholders build, engage with and better understand out of school time (OST) learning landscapes.  One of our current clients is the City of Chicago, and through the [MyChi. My Future.](https://explore.mychimyfuture.org/) site, the city is able to capture data about OST programming happening throughout the city and provide a [searchable catalog of opportunities to the public](https://mychimyfuture.org/explore?bookmark_id=&query=&community%5B%5D=any&scheduledProgram=scheduledProgram&online=online&free=on&paid=on&ageRange=any&startDate=&endDate=&topic=any&page=0&org_ids=&pp=&sort_field=&sort_order=&no_reg=&search_filters=&field_filters=&starting_field=&next_x_days=&quick_search=&program_type=).

One of the overarching ideas that our platform tries to help clients understand is whether or not their programming is being equitably distributed.  We collect equity-factor metadata specific to the opportunities on our site, but it is often necessary to combine our data with external datasets in order to get a true picture of whether or not stakeholders are locating their programs where they are most needed or will have the greatest impact.  For this ETL demonstration, I am using a typical program data set from our database and showing how we extend it with data from [Chicago Metropolitan Agency for Planning (CMAP)](https://datahub.cmap.illinois.gov/dataset/community-data-snapshots-raw-data) so that our researchers can understand impact at the community level.


## Extract Process

There are two data sets that were utilized for this demo:

1. [Chicago Coomunity Area Community Data Snapshots dataset](https://datahub.cmap.illinois.gov/dataset/community-data-snapshots-raw-data/resource/8c4e096e-c90c-4bef-9cf1-9028d094296e).  This is a dataset released by CMAP in July 2021 that consolidates census data, American Community Survey data, CMAP reporting data and other sources and aggregates it at the Community level for Chicago.  This was particularly useful because the City of Chicago uses Community as one of the key [geographic groupings of the data that they collect through our platform](https://www.mychimyfuture.org/community/back-of-the-yards).  This set was available as a csv.

2. Platform Data.  We have a MySQL database that we use to run queries of specific data when necessary.  Our database is a mixture of a production replica and a few reporting tables that join data that our researchers use fairly regularly.  
```sql
   select *
   from scheduled_programs as sp
   left join reporting.rpt_program_geographic_distribution as rgd
   on sp.id = rgd.col_program_id
   where sp.site_id=2 and
   sp.created_at>'2021-05-01'
   and sp.meeting_type='face_to_face';
```


   The scheduled_programs table contains the OST programs that the city has collected, and the rpt_program_geographic_distribution table is a reporting table that    contains geographic and category data for each scheduled program.  The query was run and the results were exported to a CSV.
   
   
# Transform Process

I used a Jupyter Notebook for the transform process to make the interim results easier to 