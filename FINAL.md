

```python
from geopy.geocoders import Nominatim
geolocator = Nominatim()
import requests as req
import pandas as pd
import numpy as np
import tweepy
import time
import json
import plotly.plotly as py
import plotly
from plotly.graph_objs import *
import config as c
```


```python
def request(host, path, api_key, url_params=None):
    url_params = url_params or {}
    url = '{0}{1}'.format(host, quote(path.encode('utf8')))
    headers = {
        'Authorization': 'Bearer %s' % api_key,
    }
    print(u'Querying {0} ...'.format(url))
    response = req.request('GET', url, headers=headers, params=url_params)
    return response.json()
```


```python
def search(api_key, term, location):
    OFFSET = 0
    url_params = {
        'term': term.replace(' ', '+'),
        'location': location.replace(' ', '+'),
        'sort_by': 'rating',
        'limit':SEARCH_LIMIT,
        'offset': OFFSET,
    }
    return request(API_HOST, SEARCH_PATH, api_key, url_params=url_params)
```


```python
# yelp info
API_KEY= c.yKey
# API constants, you shouldn't have to change these.
API_HOST = 'https://api.yelp.com'
SEARCH_PATH = '/v3/businesses/search'
BUSINESS_PATH = '/v3/businesses/'  # Business ID will come after slash.
SEARCH_LIMIT = 50
OFFSET = 0
```


```python
try:
    # For Python 3.0 and later
    from urllib.error import HTTPError
    from urllib.parse import quote
    from urllib.parse import urlencode
except:
    pass
```


```python
# tweet credentials

# Twitter API Keys
consumer_key = c.consumer_key
consumer_secret = c.consumer_secret
access_token = c.access_token
access_token_secret = c.access_token_secret

# Setup Tweepy API Authentication
auth = tweepy.OAuthHandler(consumer_key, consumer_secret)
auth.set_access_token(access_token, access_token_secret)
api = tweepy.API(auth, parser=tweepy.parsers.JSONParser())
```


```python
# 1. BUILD DATA FRAME #################################################################################################
# Zomato key
zKey = c.zKey

# prompt user for input city, generate url for response, get city ID to use in loop
city_lookup = input('Enter a city to query:')
url = "https://developers.zomato.com/api/v2.1/cities?&q=%s&results=100" % (city_lookup)
response = req.get(url, headers={"user-key" : zKey}).json()
city_id = response['location_suggestions'][0]['id']
city_name = response['location_suggestions'][0]['name']

# build empty lists to hold restaurant info
names2 = []
lngs2 = []
lats2 = []
addresses2 = []
ratings2 = []
counts2 = []
cities = []
#cuisine_types = []

# start loop to request restaurant info
start = 0
for x in range(5):
    url = 'https://developers.zomato.com/api/v2.1/search?entity_id=%s&entity_type=city&sort=rating&order=desc&start=%s&count=500' % (city_id, start)
    response = req.get(url, headers={'user-key':zKey}).json()
    for x in range(len(response['restaurants'])):
        names2.append(response['restaurants'][x]['restaurant']['name'])
        ratings2.append(response['restaurants'][x]['restaurant']['user_rating']['aggregate_rating'])
        counts2.append(response['restaurants'][x]['restaurant']['user_rating']['votes'])
        addresses2.append(response['restaurants'][x]['restaurant']['location']['address'])
        lngs2.append(response['restaurants'][x]['restaurant']['location']['longitude'])
        lats2.append(response['restaurants'][x]['restaurant']['location']['latitude'])
        #cuisine_types.append(response['restaurants'][x]['restaurant']['cuisines'])
        cities.append(city_name)
    start = start + 20

# convert lists to dataframe, export
df2 = pd.DataFrame(columns={})      
df2['Name'] = names2
df2['Zomato Rating'] = ratings2
df2['Zomato Review Count'] = counts2
df2['Address'] = addresses2
df2['Longitude'] = lngs2
df2['Latitude'] = lats2
#df2['Cuisine Type'] = cuisine_types
df2['City Name'] = cities

# query Zomato results in Google API (get Google ratings for each restaurant)
google_ratings = []
input_city = city_lookup
location = geolocator.geocode(input_city)
latitude = location.latitude
longitude = location.longitude
target_city = {"lat": latitude, "lng": longitude}
radius = 8000
gkey = c.gKey
for x in range(len(df2)):
    keyword = df2['Name'][x]
    keyword = keyword.replace(" ", "+")
    url = "https://maps.googleapis.com/maps/api/place/nearbysearch/json?key=%s&location=%s,%s&radius=%s&keyword=%s" % (gkey, target_city["lat"],
                                                                                                            target_city["lng"], 
                                                                                                            radius, keyword)
    response = req.get(url).json()
    try:
        google_ratings.append(response['results'][0]['rating'])
    except:
        google_ratings.append('N/A')

# query Zomato results in Yelp API (get Yelp ratings for each restaurant)
yelp_ratings = []
yelp_reviews = []
for name in names2:
    term = name
    location = city_lookup
    response = search(API_KEY, term, location)
    yelp_ratings.append(response['businesses'][0]['rating'])
    yelp_reviews.append(response['businesses'][0]['review_count'])

# combine data 
df2['Google Rating'] = google_ratings
df2['Yelp Rating'] = yelp_ratings
df2['Yelp Review Count'] = yelp_reviews
df2 = df2.replace('N/A', np.NaN)
for index, row in df2.iterrows():
    zomato = df2['Zomato Rating'].astype(float)
    google = df2['Google Rating'].astype(float)
    yelp = df2['Yelp Rating'].astype(float)
    df2['Composite Rating'] = (zomato + google + yelp)/3
    df2['Total Review count'] = (df2['Zomato Review Count'].astype(int) + df2['Yelp Review Count'].astype(int))

comp_drop_na = df2['Composite Rating'].fillna((df2['Zomato Rating'].astype(float) + df2['Yelp Rating'].astype(float))/2)
df2['Composite Rating'] = comp_drop_na
df2['Composite Rating'] = df2['Composite Rating'].astype(float)
df2 = df2.sort_values('Composite Rating', ascending=False)
df2 = df2.reset_index(drop=True)

# export to CSV
df2.to_csv('comp_ratings.csv')

# 2. TWEET RESULTS ####################################################################################################
# tweet top 5 results
tweet_text = 'The top restaurants in %s this week are: 1. %s (%s), \
2. %s (%s), \
3. %s (%s), \
4. %s (%s), \
& 5. %s (%s)' % (
                 city_lookup,
                 df2['Name'][0], round(df2['Composite Rating'][0],2),
                 df2['Name'][1], round(df2['Composite Rating'][1],2),
                 df2['Name'][2], round(df2['Composite Rating'][2],2),
                 df2['Name'][3], round(df2['Composite Rating'][3],2),
                 df2['Name'][4], round(df2['Composite Rating'][4],2))


try:
    api.update_status(tweet_text)
except Exception as e:
    print(e)
    print("Attempted to tweet: {}".format(tweet_text))
```

    Enter a city to query:Austin
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    Querying https://api.yelp.com/v3/businesses/search ...
    [{'code': 187, 'message': 'Status is a duplicate.'}]
    Attempted to tweet: The top restaurants in Austin this week are: 1. Home Slice Pizza (4.67), 2. Torchy's Tacos (4.63), 3. Franklin Barbecue (4.6), 4. Gourdough's (4.6), & 5. Moonshine Patio Bar & Grill (4.57)



```python
# 3. PLOTLY ###########################################################################################################
# build hover column
for index, row in df2.iterrows():
    df2['Hover'] = 'Name: ' + df2['Name'].astype(str) + '<br>Address: ' + df2['Address'].astype(str) \
    + '<br>Composite Rating: ' + round(df2['Composite Rating'], 2).astype(str) + '<br>Total Review Count: ' + df2['Total Review count'].astype(str)
    
# plotly map

# plotly credentials
mapbox_access_token = c.map_box_token
plotly.tools.set_credentials_file(username='kevious', api_key=c.pKey)

# map info
# get city lat and lon
location_geo = geolocator.geocode(city_lookup)
lat_set = location_geo.latitude
lon_set = location_geo.longitude

data = Data([
    Scattermapbox(
        lat=df2['Latitude'],
        lon=df2['Longitude'],
        mode='markers',
        marker=Marker(
            size=10
        ),
        text=df2['Hover'],
    )
])

layout = Layout(
    autosize=True,
    hovermode='closest',
    mapbox=dict(
        accesstoken=mapbox_access_token,
        bearing=0,
        center=dict(
            lat=lat_set,
            lon=lon_set
        ),
        pitch=0,
        zoom=10,
    ),
)

fig = dict(data=data, layout=layout)
py.iplot(fig, filename='top_restaurants')
```




<iframe id="igraph" scrolling="no" style="border:none;" seamless="seamless" src="https://plot.ly/~kevious/6.embed" height="525px" width="100%"></iframe>


