# ETL-Project_Harry_Jaime

ETL Report
Harry Xing / Jaime Carrillo
The team performed two ETLs to correlate COVID-19 cases with socioeconomic data in Santa Clara County and a third ETL to extract ethnicity distribution in the same county.
ETL-1
E_xtract 
The sources of data:
•	Census data was extracted from a JSON API at https://www.census.gov/data/developers/updates/new-discovery-tool.html (https://api.census.gov/data/2018/acs/acs5/cprofile)
[{'B19013_001E': 13092.0,
 'B01003_001E': 17242.0,
  'B01002_001E': 40.5,
  'B19301_001E': 6999.0,
  'B17001_002E': 10772.0,
  'B23025_004E': 3495.0,
  'B23025_002E': 5811.0,
  'B23025_005E': 2316.0,
  'zip code tabulation area': '00601'},

•	COVID-19 data was extracted from the Santa Clara County in JSON format- https://data.sccgov.org/resource/j2gj-bg6c.json
 
Requests functions were performed to extract census and COVID-19 data.
T_ransform
Data was transformed by cleaning, filtering and joining primarily
•	The “census_zip” dataframe was transformed by renaming columns 
census_zip = census_zip.rename(columns={
                                  "B01003_001E": "Population",
                                      "B01002_001E": "Median Age",
                                      "B19013_001E": "Household Income",
                                      "B19301_001E": "Per Capita Income",
                                      "B17001_002E": "Poverty Count",
                                      "B23025_004E": "employment_employed",
                                      "B23025_002E": "Labor_force",
                                      "B23025_005E": "employment_unemployed",
                                    "zip code tabulation area": "zipcode"})

•	and by dropping NaN (dropna). - census_zip = census_zip.dropna()
•	In the “santa_clara_covid_zipcode” table, the “cases” column was renamed to “infected count” – 
santa_clara_covid_zipcode = santa_clara_covid_zipcode.rename(\
                  columns={"cases": "infected count" 
                            })
•	empty rows were filled with ‘0’ (fillna), 
santa_clara_covid_zipcode['infected count'] = \
    santa_clara_covid_zipcode['infected count'].fillna(0)

•	and a new column; “infected percent,” was inserted and percentages calculated.
santa_clara_covid_zipcode["infected percent"] =\
    100 * santa_clara_covid_zipcode["infected count"].astype(int)/santa_clara_covid_zipcode["population"].astype(int)

•	“santa_clara_covid_zipcode” and “census_zip” were merged with a ‘left’ join into the “santa_clara_covid_demographics”
santa_clara_covid_demographics = pd.merge(santa_clara_covid_zipcode, census_zip, how="left", on = ["zipcode","zipcode"])
 
L_oad
Data was loaded to postgresql database
•	local connection (create_engine) to postgresql was used to create covid19_santa_claraDB 
engine = create_engine(f'postgresql://postgres:@localhost/covid19_santa_claraDB')

•	santa_clara_covid_demographics was loaded to the covid19_santa_claraDB as the “covid19” table (relational)
santa_clara_covid_demographics.to_sql(name='covid19', con=engine, if_exists='replace', index=False)
 
ETL-2
E_xtract 
Source of data:
•	Zip code, population, and density data for Santa Clara County was extracted by scraping:  http://www.mapszipcode.com/california/ladera%20ranch/  + zipcode
response = requests.get(base_url + zipcode)
    soup = bs(response.text, 'html.parser')
    results = soup.find_all('div', class_="dat")

•	family income was also scraping from http://www.mapszipcode.com/california/ladera%20ranch/' + zipcode +  '/demographics/' 
response = requests.get('http://www.mapszipcode.com/california/ladera%20ranch/' + zipcodeDict['zipcode'] +  '/demographics/')
    #BeautifulSoup object
    soup = bs(response.text, 'html.parser')
    #Retrieve the latest subject and content from the Mars website
    results = soup.find_all('div', class_="col span_12_of_12")

T_ransform
•	A list of dictionaries (“zipcode_list”).  One dictionary was created by scraping zip codes, population and density for Santa Clara County.
zipcodes = santa_clara_covid_zipcode['zipcode'].to_list()

•	Text was split (text.split) to remove unnecessary text from results and using a replace function
zipcodeDict["population"] = results[0].text.split(' ', 1 )[0]
    zipcodeDict["density"] = results[1].text.split(' ', 1 )[0].replace('\t', '').replace('/', '')

•	Results were appended to the zipcode_list
zipcode_list.append(zipcodeDict)
[{'zipcode': '94022', 'population': '19,310', 'density': '1,104.17'},
	 {'zipcode': '94024', 'population': '22,536', 'density': '3,086.45'},

•	Mapszipcode.com was also scraped to create the ‘family income’ dictionary. 

•	Results were cleaned by replacing ‘$’ with ‘USD’ 
income[dictResult[0].contents[0].replace('$', 'USD')] = dictResult[1].contents[0]    
[{'zipcode': '94022',
  'population': '19,310',
  'density': '1,104.17',
  'income': {'USD30,000 or less': '4.51%',
   'USD30,000 and USD50,000': '5.67%',
   'USD50,000 and USD100,000': '11.6%',
   'USD100,000 and USD200,000': '28.45%',
   'USD200,000 or more': '49.77%'}},

L_oad
•	zipcode_list including results from the two scrapings were uploaded to MongoDB into a demographics_zip_DB database and populated a demographics_zip collection. (non relational).
{"_id":{"$oid":"5f4f003b34ba69f8fdb5197e"},"zipcode":"94022","population":"19,310","density":"1,104.17","income":{"USD30,000 or less":"4.51%","USD30,000 and USD50,000":"5.67%","USD50,000 and USD100,000":"11.6%","USD100,000 and USD200,000":"28.45%","USD200,000 or more":"49.77%"}}

ETL-3
E_xtract 
•	Use a dataframe to request dataset from mapszipcode.com 
tables = pd.read_html('http://www.mapszipcode.com/california/ladera%20ranch/'+ zipcode + '/demographics/')
T_ransform
ETL-3
•	Rename column “Racial makeup” to “population
df2.columns = ['Racial makeup', 'Population']

•	 dataframe to extract the data and reverse from rows to columns
dict1= (df2.to_dict())['Population']

•	Data was cleaned by dropping NaN
zipcode_data.dropna()
L_oad
•	Data was uploaded to a the covid19_santa_claraDB (postgresql database) in the demographic_zipcode table. 
 

We wanted to use PostgreSQL and mongoDB to practice what we learned in class. 
