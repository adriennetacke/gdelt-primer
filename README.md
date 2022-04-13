# GDELT Primer

## What is GDELT?

An acronym for **G**lobal **D**atabase of **E**vents, **L**anguage, and **T**one, the [GDELT](https://www.gdeltproject.org/) dataset is a comprehensive database of news happening around the world, updated near real-time every 15 minutes.

## How to download data from GDELT dataset and import into MongoDB Atlas
These are the steps I performed to grab data from the main GDELT dataset, import it into MongoDB Atlas, and shape it into a compressed document to make it easier to work with:

### Get GDELT data

1. Download main GDELT file

    I used [gdelttools](https://github.com/jdrumgoole/gdelttools) to do this. Once you've cloned the gdelttools repo, navigate to that directory and run the following command:

    ```python
    python gdelttools\gdeltloader.py --ziplist master
    ```

2. Reduce main file to the amount of days you want (number passed to the `tail` command) and only export archives. I went with `120` days:

    ```bash
    grep export gdelt_master-file-*.txt | tail -n 120 > last_120_days.txt
    ```

3. Extract URLs from the condensed file (`last_120_days.txt` file) and download them via [cURL](https://everything.curl.dev/) commands:

    ```bash
    cat last_120_days.txt | awk '{ print $3 }' | xargs -I % curl -O %
    ```

4. Extract all zipfiles to CSVs. Delete all zips. I did this through the explorer (literally selected all zipfiles, right-click, and extract all). Feel free to update this with the actual command!

5. Join CSVs to create a single file. NOTE: extension is case sensitive!

    ```bash
    awk '{print}' *.CSV > news.csv
    ```

### Import GDELT data into MongoDB Atlas
Now that we have the subset of data we want (`news.csv`), we can use [`mongoimport`](https://www.mongodb.com/docs/database-tools/mongoimport/) to import it into our [MongoDB Atlas](https://www.mongodb.com/atlas/database) cluster. 

1. Open another terminal, navigate to the same directory, and run the following `mongoimport` command (be sure to update with your own connection string and credentials!):

    ```mongodb
    mongoimport --uri mongodb+srv://USERNAME:PASSWORD@CLUSTER.mongodb.net/DATABASE --collection COLLECTION --type=tsv --file news.csv --fieldFile=fields.txt --columnsHaveTypes
    ```

    Some notes: 

    - `--fieldFile` uses the `fields.txt` file that is available in this repo. This maps the raw GDELT records to the appropriate headers.
    - Be sure to update the `USERNAME`, `PASSWORD`, `CLUSTER`, `DATABASE`, and `COLLECTION` parameters with your own information/names!


### Shaping data via Aggregation Pipelines

With our data now in MongoDB Atlas, we have some cleanup to do. The next steps refine and reshape the data via Aggregation Pipelines. To run these aggregations, I used the [MongoDB Extension for VSCode](https://code.visualstudio.com/docs). Just connect to your cluster, then run the aggregations in this repo (in order!).

1. Convert raw latitude, longitude strings to double type. Then, create GeoJSON points out of them. You can do this by running the `convert-geojson-pipeline.mongodb` aggregation. 

2. Compress the documents. You can do this by running the `compress-pipeline.mongodb` aggregation.


## News Record Walkthrough

After importing the subset of GDELT data into MongoDB Atlas and running the provided aggregation pipelines, the documents may look something like this:

```json
{
    "_id": "623cb868b8e6a89b358c5bf7",
    "GlobalEventId": 1035443882,
    "Day": "20210323",
    "MonthYear": "202103",
    "Year": 2021,
    "FractionDate": 2021.2274,
    "IsRootEvent": 1,
    "EventCode": "020",
    "EventBaseCode": "020",
    "EventRootCode": "02",
    "QuadClass": 1,
    "GoldsteinScale": 3.0,
    "NumMentions": 4,
    "NumSources": 1,
    "NumArticles": 4,
    "AvgTone": -5.17241379310345,
    "DATEADDED": "20220323113000",
    "SOURCEURL": "https://www.wbur.org/cognoscenti/2022/03/23/dont-look-up-climate-change-oscars-academy-awards-frederick-hewett",
    "Actor1": {
        "Name": "UNITED STATES",
        "Code": "USA",
        "CountryCode": "USA",
        "KnownGroupCode": "",
        "EthnicCode": "",
        "Religion1Code": "",
        "Religion2Code": "",
        "Type1Code": "",
        "Type2Code": "",
        "Type3Code": "",
        "Geo": {
            "type": "Point",
            "coordinates": [-118.327, 34.0983]
        },
        "Geo_Type": 3,
        "Geo_Fullname": "Hollywood, California, United States",
        "Geo_CountryCode": "US",
        "Geo_ADM1Code": "USCA",
        "Geo_ADM2Code": "CA037",
        "Geo_FeatureID": "1660757"
    },
    "Actor2": {
        "Name": "SCIENTIST",
        "Code": "CVL",
        "CountryCode": "",
        "KnownGroupCode": "",
        "EthnicCode": "",
        "Religion1Code": "",
        "Religion2Code": "",
        "Type1Code": "CVL",
        "Type2Code": "",
        "Type3Code": "",
        "Geo": {
            "type": "Point",
            "coordinates": [-118.327, 34.0983]
        },
        "Geo_Type": 3,
        "Geo_Fullname": "Hollywood, California, United States",
        "Geo_CountryCode": "US",
        "Geo_ADM1Code": "USCA",
        "Geo_ADM2Code": "CA037",
        "Geo_FeatureID": "1660757"
    },
    "Action": {
        "Geo": {
            "type": "Point",
            "coordinates": [-118.327, 34.0983]
        },
        "Geo_Type": 3,
        "Geo_Fullname": "Hollywood, California, United States",
        "Geo_CountryCode": "US",
        "Geo_ADM1Code": "USCA",
        "Geo_ADM2Code": "CA037",
        "Geo_FeatureID": "1660757"
    }
}
```

The full description of each field can be found in the official [GDELT Event Codebook](http://data.gdeltproject.org/documentation/GDELT-Event_Codebook-V2.0.pdf). For now, we've condensed each field's description into the following sections to help you quickly make sense of the document. 


### Event
- `GlobalEventId`: Globally unique identifier assigned to each event record in the main GDELT dataset.
- `Day`: Date event took place in `YYYYMMDD` format.
- `MonthYear`: Alternative formatting of event date in `YYYYMM` format.
- `Year`: Alternative formatting of event date in `YYYY` format.
- `FractionDate`: Alternative formatting of the event date, computed as `YYYY.FFFF`, where `FFFF` is the percentage of the year completed by that day. The fractional component (`FFFF`) is computed as `(MONTH * 30 + DAY) / 365`.

### Actor Attributes
For every event, two Actors may be identified as being involved. Respectively, these are Actor1 and Actor2. Each Actor will have attributes and characteristics that describe them, those of which will be in it's own subdocument. [CAMEO codes](https://parusanalytics.com/eventdata/cameo.dir/CAMEO.Manual.1.1b3.pdf) are used for many of these attributes and can be found in the CAMEO code collection.

`Actor1/Actor2: {`

- `Code`: Raw CAMEO code that describes the Actor.
- `Name`: Actual name of the Actor.
- `CountryCode`: 3-character CAMEO code for the country affiliation of the Actor.
- `KnownGroupCode`: If Actor is a known IGO/NGO/rebel organization with it's own CAMEO code, this field will contain that code.
- `EthnicCode`: If specified in source document and has a CAMEO entry, the CAMEO code denoting the ethnic affiliation of the Actor is here.
- `Religion1Code`: If specified in source document and has a CAMEO entry, the CAMEO code denoting the religious affiliation of the Actor is here.
- `Religion2Code`: If there are multiple religious codes specified for the Actor, this contains the secondary CAMEO code.
- `Type1Code`: 3-Character CAMEO code of the CAMEO "type" or "role" of the Actor.
- `Type2Code`:  If multiple type/role codes are specified for Actor, this returns the second CAMEO code.
- `Type3Code`: If multiple type/role codes are specified for Actor, this returns the third CAMEO code.
- `Geo`: GeoJson point holding the [`longitude`, `latitude`] for the location.
- `Geo_Type`: Geographic match type. Possible values are: 1=COUNTRY (match was at the country level), 2=USSTATE (match was to a US state), 3=USCITY (match was to a US city or landmark), 4=WORLDCITY (match was to a city or landmark outside the US), 5=WORLDSTATE (match was to an Administrative Division 1 outside the US – roughly equivalent to a US state).
- `Geo_Fullname`: Full, human-readable name of matched location. 
- `Geo_CountryCode`: 2-character FIPS10-4 country code for the location.
- `Geo_ADM1Code`: 
- `Geo_ADM2Code`: 
- `Geo_FeatureID`: 

`}`


### Event Action Attributes
These fields describe the attributes of the event "action" (what Actor1 did to Actor2). They also include several fields to help assess the "importance" or immediate-term "impact" of an event.

- `IsRootEvent`:
- `EventCode`: Raw CAMEO action code describing the action that Actor1 performed upon Actor2.
- `EventBaseCode`:
- `EventRootCode`:
- `QuadClass`: Primary classification for event type. There are four possible classifications (equivalent numeric code in parenthesis): Verbal Cooperation (`1`), Material Cooeperation (`2`), Verbal Conflict (`3`), and Material Conflict (`4`).
- `GoldSteinScale`: Score that measures the theoretical potential impact the type of event will have on the stability of a country. Range is from `-10` to `+10`.
- `NumMentions`:  Total number of mentions of this event across all source documents **during the 15 minute update in which it was first seen**. Multiple references to an event within a single document also contribute to this count.
- `NumSources`: Total number of information sources containing one or more mentions of this event **during the 15 minute update in which it was first seen**.
- `NumArticles`:  Total number of source documents containing one or more mentions of this event **during the 15 minute update in which it was first seen**. Can be used to assess "importance" of an event: the more discussion of that event, the more likely it is to be significant.
- `AvgTone`: Score that measures the average "tone" of all documents mentioning the event **in the first 15 minute update in which it was first seen**. Range is from `-100` (extremely negative) to `+100` (extremely positive), with common values ranging between `-10` and `+10`. A `0` score indicates `neutral`.

### Additional Data Management Attributes

- `DATEADDED`: Date event was added to main GDELT database in `YYYYMMDDHHMMSS` format in the UTC timezone.
- `SOURCEURL`: URL or citation of first news report the event was found in.

## Additional Resources

Some relevant resources to help you along the way:

- [GDELT Event Codebook (v2.0)](http://data.gdeltproject.org/documentation/GDELT-Event_Codebook-V2.0.pdf) - Official resource for raw GDELT record fields and their description.
- [CAMEO Manual](https://parusanalytics.com/eventdata/cameo.dir/CAMEO.Manual.1.1b3.pdf) - Official resource for CAMEO codes used in GDELT records.
