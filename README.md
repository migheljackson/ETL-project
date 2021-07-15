# Program Community Data ETL Project

The research project that I have been working with for the past several years is a multi-sided platform called [CitiesLearn](http://citieslearn.com/).  The platform provides several tools with the primary goal of helping stakeholders build, engage with and better understand out of school time (OST) learning landscapes.  One of our current clients is the City of Chicago, and through the [MyChi. My Future.](https://explore.mychimyfuture.org/) site, the city is able to capture data about OST programming happening throughout the city and provide a [searchable catalog of opportunities to the public](https://mychimyfuture.org/explore?bookmark_id=&query=&community%5B%5D=any&scheduledProgram=scheduledProgram&online=online&free=on&paid=on&ageRange=any&startDate=&endDate=&topic=any&page=0&org_ids=&pp=&sort_field=&sort_order=&no_reg=&search_filters=&field_filters=&starting_field=&next_x_days=&quick_search=&program_type=).

One of the overarching ideas that our platform tries to help clients understand is whether or not their programming is being equitably distributed.  We collect equity-factor metadata specific to the opportunities on our site, but it is often necessary to combine our data with external datasets in order to get a true picture of whether or not stakeholders are locating their programs where they are most needed or will have the greatest impact.  For this ETL demonstration, I am using a typical program data set from our database and showing how we extend it with data from [Chicago Metropolitan Agency for Planning (CMAP)](https://datahub.cmap.illinois.gov/dataset/community-data-snapshots-raw-data) so that our researchers can understand impact at the community level.


## Extract Process

There are two data sets that were utilized for this demo:

1. [Chicago Community Area Community Data Snapshots dataset](https://datahub.cmap.illinois.gov/dataset/community-data-snapshots-raw-data/resource/8c4e096e-c90c-4bef-9cf1-9028d094296e).  This is a dataset released by CMAP in July 2021 that consolidates census data, American Community Survey data, CMAP reporting data and other sources and aggregates it at the Community level for Chicago.  This was particularly useful because the City of Chicago uses Community as one of the key [geographic groupings of the data that they collect through our platform](https://www.mychimyfuture.org/community/back-of-the-yards).  This set was available as a csv.

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

I used a Jupyter Notebook for the transform process to make the interim results easier to check quickly.  My process was as follows:
1. After importing the separate CSVs as data frames, one for program data, one for community statistical data, I checked to see what columns would be useful for looking at equity.  The program data dataframe contained 133 columns while the community stats dataframe contained 245 columns.  Ultimately I was able to reduce the program data dataframe to 14 columns including metadata such as name, description, min/max age, equity factors such as price range and whether food, scholarships or transportation are provided, and the Chicago-defined geographic community the program is based in.  The name of the column that contains the community, "geographic_cluster_name" was also updated to "community_name" to improve readability.  The community stats dataframe was reduced to 27 columns encompassing, Chicago-defined georgaphic community, total population, population by age and income bands, access to devices and internet, english language proficiency and transit/walkability access.

2. Once I generated reduced dataframes, I dropped rows that contained NULL values from each so that we could ultimately have a clean set of data that contained all of the variables that we wanted to consider.  This reduced the program data dataframe from 4676 to 636 rows, mainly because much of the equity factor data we collect through the platform is optional.  Running the drop on the community stat data frame reduced it from 104 rows to 77 which was expected because Chicago officially has 77 communities, and each row in the CSV we imported represents one community.  The additional rows were all completely empty.

3. The key step in the transformation was identifying the column to merge on.  Because our platform uses Community definitions that are directly from the City of Chicago and the CMAP data also uses the same definitions, I was able to use community name as the value to merge the two dataframes on.  However, since this value is a string, I ran a line of code on the community stats dataframe to make sure all of the community names there were all upper case to match what was in the program data dataframe:

```python
   comm_study_data['GEOG']=comm_study_data['GEOG'].str.upper()
```

4. With the column that I planned to merge on now in matching formats, I performed a left join using program data dataframe as the left table since it had multiple entries for each community, and the community stats dataframe as the right table.  This created a single program community stats dataframe that contains a program and its associated metadata in each row as well as the statistics for the community that program is in.

5. There is a slight quirk to the data that is not readily apparent at first.  Although the column col_program_id looks like it could be an index, upon further inspection it actually contains duplicates.  This is because we included the "category_name" field in our initial query, and some programs have multiple categories, which in turn, generates multiple rows for the same program in our dataset.  Because we work with people that are doing a few different types of research, we often will generate data for different audiences.  The first dataframe that I generated with the duplicates is good for our Tableau users because it requires no parsing to understand what categories are represented.  However, for our more data savvy researchers, I generated another dataframe that removes the duplicate program_IDs, and merges them down into a comma separated field for category_name:

```python
prog_comm_flat_category_df=prog_comm_study_df.groupby(['col_program_id', 'name', 'org_id', 'description', 'community_name',
       'min_age', 'max_age', 'capacity', 'program_price',
       'program_has_scholarships', 'program_provides_free_food',
       'program_pays_participants', 'program_provides_transportation','2010_POP', 'TOT_POP', 'UND5', 'A5_19',
       'POP_16OV', 'INC_LT_25K', 'INC_25_50K', 'INC_50_75K', 'INC_75_100K',
       'INC_100_150K', 'INC_GT_150', 'MEDINC', 'COMPUTER', 'ONLY_SMARTPHONE',
       'NO_COMPUTER', 'INTERNET', 'BROADBAND', 'NO_INTERNET', 'NOT_ENGLISH',
       'LING_ISO', 'ENGLISH', 'TRANSIT_LOW_PCT', 'TRANSIT_MOD_PCT',
       'TRANSIT_HIGH_PCT', 'WALKABLE_LOW_PCT', 'WALKABLE_MOD_PCT',
       'WALKABLE_HIGH_PCT'])['category_name'].apply(','.join).reset_index()
```


This makes it easier to perform aggregations because it is not as reliant on "Count Distinct" sorts of operations, but requires parsing to retrieve some data cleanly.


## Load Process

Typically when we run these sort of query and merge operations, we generate a csv that can be used in Tableau or the platform of choice for the researcher.  I have done that here, and the file is available for download or distribution.  For the purposes of this ETL demonstration and to make the data available more broadly, I have created a database using MongoDB containing two separate collections: the first contains the full "non-flattened" category data with duplicate col_program_ids, and the second collection contains the "flattened" category data that unique col_program_ids and comma separated category field.


## Additional Notes

I initially attempted to use our platform's program data API in order to pull the program data that I needed in a more seamless fashion, but the structure of the API is not currently meant to handle bulk requests that span multiple organizations simultaneously. I am hoping to work with with our developers to better understand and potentially refine the API, and hopefully will update the code herein.
