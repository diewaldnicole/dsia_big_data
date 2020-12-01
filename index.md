# Small Data
## How to speed up your Application without the need for Big Data Technologies?

The following work deals with performance issues in an existing program for data extraction from an API for timeseries measurement values. The code was growing over time and has therefore never been rethought to improve it from a basis. In addition, no performance analysis has ever been conducted for this piece of code.

The following work is showing how and how much I could improve...

*...the performance of data retrieval via REST API*

*...parsing of JSON encoded timeseries data*

*...and saving it into a .csv file?*

![png](https://github.com/diewaldnicole/dsia_big_data/blob/gh-pages/flowchart_small_data.png)

**Initial Thoughts and Plan:** 
- Analysis of the state of the Art - which part takes how much time?
- Improve Performance of REST API request by varying request time horizons
- Improve parsing of json files
- Improve overall Performance by using Multithreading

## Benchmark
In order to have a baseline for comparison, at first the existing functions used are shown. THis includes:
- function for the api request (datapi_channels_fields)
- function for parsing (parse_json)
- main function (baseline)

The script is based on some older version starting from a time with no python knowledge at all, and and was evolving over time. For example, is was taken as a given fact that only two days at once can be retrieved from the API, based on an older version of this API. However, this restriction changed and also longer time horizons can be retrieved at once now. The 'baseline' function above is already adapted so that it supports later used varying time horizons (days to request at once).

The **benchmark time** for this initial version can be found in the later chapter 1.2.1.

```python
# function to parse the json string
def parse_json(j):
    # define complete array with timestamps
    interval = datetime.timedelta(minutes=30)
    interval_current = datetime.timedelta(minutes=datetime.datetime.strptime(j['Records'][2]['TargetDuration'], '%H:%M:%S').time().minute)
    if interval_current < interval:
        interval = interval_current
    data_twodays = pd.DataFrame()

    for a in j['Records']:
        if a['Value'] is not None:
            rec_deviceid = a['DeviceId']

            rec_key = str(a['ChannelType'] + '_' + str(a['NodeType']) + str(' [') + str(a['Value']['Unit']) + str(']'))
            rec_value = a['Value']['Value']
            rec_logdt_dt = datetime.datetime.strptime(a['LogDt'], '%Y-%m-%dT%H:%M:%SZ')
            rec_logdt_UTC = rec_logdt_dt.replace(tzinfo=tz.tzutc())
            data_twodays.at[rec_logdt_UTC, rec_key] = rec_value

    return data_twodays, interval
```


```python
def baseline(days):

    filename = pvsystemid + '.csv'

    #######################################################################################################################
    # init arrays for request timestamp
    timespan=int(np.ceil((until_day + datetime.timedelta(days=1) - start_day).days/days))
    print(timespan)
    period=datetime.timedelta(days = days, hours=0, minutes=0)
    print(period)
    startdate_list=[]
    for x in range(0, (timespan)):
        startdate_list.append([datetime.datetime.combine(start_day, datetime.time(0, 0)) + period*x][0])

    # init data frame for export (overall data)
    df_all = pd.DataFrame()
    ######################################################################################################################
    soa_api_requ = []
    soa_parsing = []
    soa_parsing_days = []
    soa_api_requ_days = []
    #print(startdate_list)
    # loop over list with startdates
    for requ_start in startdate_list:
        requ_end = requ_start + period - datetime.timedelta(minutes=1)
        if requ_end.date() > until_day:
            print('yes')
            requ_end = datetime.datetime.combine(until_day, datetime.time(23, 59))
        # print(f'{requ_start} to {requ_end}')
        delta = (requ_end + datetime.timedelta(minutes=1) -requ_start).days
        # request from api
        starttime = timeit.default_timer()
        j = datapi_channels_fields(requ_start, requ_end, channels, pvsystemid)
        end_api_requ = timeit.default_timer() - starttime
        soa_api_requ.append([end_api_requ])
        soa_api_requ_days.append([end_api_requ/delta])

        # check output
        if j['Records'] == []:
            continue

        # get tz
        olson_tz = tz.gettz(j['Olson'])

        #print(j)
        
        # parse json to data frame & add timezone info (UTC)
        starttime_parsing = timeit.default_timer()
        data_twodays, interval = parse_json(j)
        end_parse = timeit.default_timer() - starttime_parsing
        soa_parsing.append([end_parse])
        soa_parsing_days.append([end_parse/delta])

        # add to overall data frame
        df_all = pd.concat([df_all, data_twodays], sort=True)

    ######################################################################################################################
    # fill missing timestamps
    test = pd.date_range(start=min(df_all.index), freq=interval, end=max(df_all.index))

    df_all=df_all.reindex(index=test)

    # convert to local time
    df_all = df_all.tz_convert(olson_tz)
    df_all.index.name = "DateTime"
    
    return df_all, soa_api_requ, soa_parsing, soa_api_requ_days, soa_parsing_days
```

## Performance improvements

In order to improve the performance of this application, at first it has to be found out which part takes longest and a benchmark has to be set. Therefore, the Python **"timeit"** module is used to track the durations of the request itself as well as the parsing. This task is combined with the first analysis of possible improvements:

### Improve Performance of REST API request by varying request time horizons

Firstly, the time horizon for the request is varied - so that not only just two days at once are retrieved from the API, but also longer time horizons. Therefore, the initial program is adapted to enable this time horizons instead of a hard coded two days interval. **The 2_days result is the benchmark for the analysis**.

```markdown
*code*
```

The table below shows the difference of the time needed for the API request as well as for parsing, for varying time horizons from 1 day to 120 days within one request:
- api_per_day_s are the seconds needed to get the data from the api for one day (mean)
- parsing_per_day_s are the seconds needed to parse the data for one day (mean)
- api_sum_s is the overall time taken by the api request (sum)
- parsing_sum_s is the overall time taken for parsing (sum)
- parsing/api is the ratio between parsing and the api duration sums
- summe_s is the overall sum to get the data back as a data frame

    +----------+-----------------+---------------------+-------------+-----------------+---------------+-----------+
    |          |   api_per_day_s |   parsing_per_day_s |   api_sum_s |   parsing_sum_s |   parsing/api |   summe_s |
    |----------+-----------------+---------------------+-------------+-----------------+---------------+-----------|
    | 1_days   |        0.799743 |            0.398334 |     292.706 |         145.79  |      0.498078 |   438.496 |
    | 2_days   |        0.561995 |            0.457617 |     205.69  |         167.488 |      0.814273 |   373.178 |
    | 7_days   |        0.463274 |            0.434537 |     167.204 |         159.471 |      0.953754 |   326.675 |
    | 14_days  |        0.432004 |            0.388936 |     157.535 |         142.91  |      0.907161 |   300.445 |
    | 30_days  |        0.414554 |            0.538641 |     146.972 |         193.557 |      1.31696  |   340.529 |
    | 60_days  |        0.392523 |            0.551602 |     140.68  |         212.32  |      1.50924  |   353.001 |
    | 120_days |        0.452316 |            0.611337 |     141.047 |         251.768 |      1.785    |   392.815 |
    +----------+-----------------+---------------------+-------------+-----------------+---------------+-----------+

The diagrams above show a request of **60 days** as the best option. (Altough there is not a really clear trend). The time needed for a request is increasing again for 120 days.

###  Improve parsing of json files

The JSON string retrieved from the API is nested and therefore not easy to parse to a tabular format, as seen in the example below:

``` markdown
{
  "Records": [
    {
      "ChannelType": "UACMeanL1",
      "Value": {
        "Value": 0,
        "Unit": "Voltage_100mV"
      },
      "Validity": "Calculated",
      "TargetDuration": "00:05:00",
      "Duration": "00:04:59",
      "TemplateId": null,
      "DataSourceIdentity": {
        "DataSourceId": "240.198074",
        "DataSourceType": "Datamanager"
      },
      "LastModifiedDt": "2019-06-01T04:01:07.9053",
      "LogDt": "2019-05-31T22:00:00Z",
      "NodeType": 97,
      "DeviceType": 123,
      "Idx": 28,
      "ComponentId": null,
      "DeviceId": "2142085f-a929-4e29-adbc-aa4b00acd810"
    },
  ],
  "Olson": "Europe/Berlin",
  "Error": null
}
```

It contains the "Records" with all the required data, and the **timezone** ("Olson"). The timezone is needed to convert the data to local time from UTC. One **ChannelType** should be one column, the **LogDt** should be the datetime index. There is also a **NodeType** which enumerates the number of the inverter in the PvSystem. If there are multiple inverters in the system, the UACMeanL1 channel exists multiple times, once for each inverter and therefore there must be a column in the dataframe depending on the ChannelType AND NodeType combination. Also the **Unit** is an important information to be added to the export file.

The procedure of the initial parsing solution works as follows:
1. data of two days is requested
2. JSON is given to the parsing function
3. the time interval (can be 5, 15 or 30 minutes) is read from the first entries
4. an empty dataframe for the two days data is initiated
5. a for loop iterates over each "Records" entry in the JSON string
    - it defines a columns key with the ChannelType, NodeType and Unit of the record
    - it uses the LogDt as the index for the record
    - it adds utc timezone info to every LogDt
    - it uses the pandas.at method to save the value at the corresponding column and index
6. the dataframe is returned to the main function and there concatenated to the dataframe containing all the values (attaching two more days of data in each loop)
7. At the end, missing timestamps are filled with NaN values and the timezone is changed to local time for the overall dataframe.

As an improvement, a performance comparison is made by replacing the use of the **pandas "at"** function. Instead, the data is being **parsed in a dictionary** and transformed to a dataframe in the end. For the experiment, a locally saved JSON file is read in which is equal to the data requested via the API.

### Improve overall Performance by using Parallel Computation

An additional improvement in performance is expected if the process can be parallelized. Therefore, parallelization based on a Medium article (https://medium.com/@mjschillawski/quick-and-easy-parallelization-in-python-32cb9027e490) is tested. 

The block below uses the multiprocessing package to get the number of cores available on the hardware.

```python
import multiprocessing
from joblib import Parallel, delayed
from tqdm import tqdm

num_cores = multiprocessing.cpu_count()
print(f'{num_cores} Cores available!')
```

### Using Dask

### Saving to .csv

## Summary and Conclusion
### General
### Boost yourself to boost your code


# delete

You can use the [editor on GitHub](https://github.com/diewaldnicole/dsia_big_data/edit/gh-pages/index.md) to maintain and preview the content for your website in Markdown files.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### Markdown

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/diewaldnicole/dsia_big_data/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
