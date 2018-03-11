# Runmember

Perhaps this is a bit unusual for people who live in NYC, but I actually prefer to run through the city streets rather than along the city's designated running paths.  Of course, there is a lot of people- and car-dodging, and a few too many starts and stops at red lights.  Regardless, the best part of running through the middle of the city is getting to explore different neighborhoods, including side streets that I'd never visit if I were walking to a specific location.

As I run by interesting looking stores, bars, and restaurants, I find myself making copious mental notes about places to look up once I'm back at home.  I've added a bunch of spots to my "to try" list using this method, but remembering a bunch of names while trying to focus on running (and the street in front of me) is pretty tedious.  Could there be a way to record all the places I passed on my run, and just browse through the names afterwards?

I already track my runs using an Apple Watch, and the watch already records GPS data for every run.  What if I could use that GPS data to get a list of all the places I saw on my run?

I spent an afternoon tinkering with a few different fitness and location APIs, and I think I've found a pretty good way to automate this using three API services (one is not strictly necessary but is included for convenience).  The basic steps are below:

1. Apple Watch continuously records my GPS coordinates during my run 
2. I use an app called [HealthFIT][1] to export my run data from Apple Health to Strava (you could just as easily track runs directly in the Strava app and skip this step - I prefer the accuracy of the Health app GPS trackng on the AW)
3. Access the Strava API to download the coordinates for my run
4. Look up a subset of the coordinates using the Foursquare API, which returns a list of nearby places
5. Show me the list of places, with name, type of place, and approx. location.  I also use the Mapbox API here to get me the name of the neighborhood associated with each set of coordinates, to make parsing through the list easiser - while not necessary, it's much easier to filter through results by neighborhood.

The first two steps are pretty self-explanatory, so I'll start with step #3.

## Downloading coordinates from Strava

Unfortunately, Apple Health data is pretty siloed.  It is very difficult to manually extract workout data from the Health app on the AW / iPhone to use in other contexts, so it's sometimes easiest to export that data to an online service with an API to get access.  I already track my runs with Strava, so I use the HealthFIT app to get my data up there.

Once the run is uploaded to Strava, we can access its data using the Strava API.  Using the `stravalib` python library, we can get the list of coordinates directly:

```python
streams = strava.get_activity_streams(activity_id, types=['latlng'], resolution="medium")
activity_coords = streams['latlng'].data
```

This gets us a list of coordinate pairs for the run.  I use "medium" resolution for the initial download; I haven't experimented with this parameter but I think this level of definition is fine for the initial list.

Since we will be pinging the Foursquare API for nearby places for each coordinate pair, it's probably best that we limit the number of coordinates we look up so we don't hit any API rate limits.  I do this by calculating the number of coordinate readings that exist per second, and using a sample interval to collect a subset of coordinates from the initial list.  (This concept is the same as downsampling an audio file to save on filesize, for example.)

To do this, we need to know how long the run was, in seconds.  I make a separate request to the Strava API for this info.

```python
activity = strava.get_activity(activity_id)
coords_per_second = len(activity_coords) / activity.elapsed_time.seconds
coords_step = int(coords_per_second * sample_interval)
activity_coords_sample = activity_coords[::coords_step]
```

In my script, I use a `sample_interval` of 60 seconds.  Again, I haven't experimented much with that parameter, but for a half hour run, 30 API requests (and 30 lists of places to review) seems fine.  It's likely that we could optimize both the sample frequency and the Foursquare search radius to cover nearly the entire run, and I may tweak those parameters as I use this script in the future.

## Search for places using Foursquare

Now that we have our list of coordinates, we need to plug each set into the Foursquare API to get a list of nearby stores, bars, and restaurants.

I didn't bother using a Foursquare python library, as we only really need to write code for a single request which is well documented in their API reference.

First, we need to set the search parameters so that Foursquare knows our search radius, the type of places we are looking for, the number of places we want them to spit back, and of course our coordinates:

```python
params = dict(
  client_id=fsq_client_id,
  client_secret=fsq_client_secret,
  v='20180311',
  categoryId = "4d4b7104d754a06370d81259,4d4b7105d754a06373d81259,4d4b7105d754a06374d81259,4d4b7105d754a06376d81259,4d4b7105d754a06377d81259,4d4b7105d754a06378d81259",
  ll = ",".join([str(x) for x in coords]),
  radius="50",
  limit=10
)
```

The `categoryId`s are the general categories for things like `Food`, `Nightlife`, etc.  There is a reference list for categories in the Foursquare API documentation if you want to add any other categories to the list.

The `ll` parameter is simply a string in the form `"lat,lng"`.  I take a coordinate pair, convert the values to strings, and join them into a single string to pass as the `ll` parameter.

`radius` is in meters.  I am using a search radius of 50 meters around each coordinate.

`limit` is the number of results we want back - I am asking for a list of 10 results.

Now, we make the request:

```python
fsq_url = "https://api.foursquare.com/v2/venues/search"
resp = requests.get(url=fsq_url, params=params)
data = json.loads(resp.text)
```

We need to parse through the results to extract our list of places, as well as any additional information we want about each place.  I'm mostly interested in what type of place each result is, as well as it's location, so I try to pull those into our list if they are available.

```python
places = []

for venue in data["response"]["venues"]:
	place = [venue["name"]]
	
	try:
		place.append(venue["categories"][0]["name"])
	except:
		pass
	
	try:
		place.append(venue["location"]["formattedAddress"][0])
	except:
		pass

	places.append(place)
```

## Getting neighborhood names

While street addresses are useful, I find it a lot more convenient to think about locations in terms of neighborhood, and it would be great if we could show which neighborhood each pair of coordinates corresponds to.  Fortunately, the Mapbox API includes a reverse geocoding service which returns neighborhood names - all we need to do is pass it our coordinates, and extract the neighborhood name from the results:

```python
mapbox_url = "https://api.mapbox.com/geocoding/v5/mapbox.places/"
lonlat = ",".join(str(x) for x in coords[::-1]) # need to reverse coords to long, lat for mapbox
request_url = mapbox_url + lonlat + ".json?access_token=" + mapbox_access_token

resp = requests.get(url=request_url)
data = json.loads(resp.text)
neighborhood = data["features"][0]["context"][0]["text"] 
```

## Printing results

At this point, this is still in a rough script form, so I'll just be printing the list to stdout (which can also be redirected to a text file for browsing/saving).  This is the main app logic that calls the other pieces of the script:

```python
activity_coords = getCoords(activity_id, 60) # takes Strava activity ID and preferred sampling frequency in seconds
for i in range(0, len(activity_coords)):
	coordinates = activity_coords[i]
	places = getPlaces(coordinates) # executes the Foursquare search for the given coordinates

	neighborhood_name = getNeighborhoodName(coordinates) # executed the Mapbox search to get neighborhood
	print(neighborhood_name + " (" + ", ".join(str(c) for c in coordinates) + ")")

	for place in places:
		print(", ".join(place))
	
	print("\n")
```

Here is a sample set of search results for one coordinate pair:

```
Flatiron District (40.738717, -73.988523)
Le Coq Rico, French Restaurant, 30 E 20th St (btw Broadway & Park Ave S)
Gramercy Tavern, American Restaurant, 42 E 20th St (btwn Broadway & Park Ave)
Mari Vanna, Russian Restaurant, 41 E 20th St (btwn Broadway & Park Ave S)
Casa Neta, Cocktail Bar, 40 E 20th St
Sugarfish, Japanese Restaurant, 33 E 20th St (btwn Broadway & Park Ave S)
Trattoria Il Mulino, Italian Restaurant, 36 E 20th St (btwn Park Ave and Broadway)
Nur, Israeli Restaurant, 34 E 20th St
Barbounia, Mediterranean Restaurant, 250 Park Ave S (at E 20th St)
Craft, American Restaurant, 43 E 19th St (btwn Park Ave S & Broadway)
Mizu Japanese & Thai Cuisine, Japanese Restaurant, 29 E 20th St (at Broadway)
```

## Wrap up, full script

So that's pretty much it - using a few free APIs, we are able to get a list of places that were passed during the course of a run.  There is probably a way to optimize the results by adjusting the search radius and sampling parameters, which I may explore in the future.  This script and idea could also be applied to anything else that could be represented by a stream of GPS coordinates, such as a hike, bike ride, car trip, etc.

The full script is below, for reference, with API identifiers removed:

```python
from stravalib.client import Client
import json, requests, sys

# CONSTANTS / SECRETS
fsq_url = "https://api.foursquare.com/v2/venues/search"
fsq_client_id = ""
fsq_client_secret = ""
strava_access_token = ""
mapbox_access_token = ""
mapbox_url = "https://api.mapbox.com/geocoding/v5/mapbox.places/"

strava = Client()
strava.access_token = strava_access_token

# Get activity ID from command line params
activity_id = sys.argv[1]

def getCoords(activity_id, sample_interval, resolution='medium'):
	activity = strava.get_activity(activity_id)

	streams = strava.get_activity_streams(activity_id, types=['latlng'], resolution=resolution)
	activity_coords = streams['latlng'].data

	coords_per_second = len(activity_coords) / activity.elapsed_time.seconds
	coords_step = int(coords_per_second * sample_interval)
	activity_coords_sample = activity_coords[::coords_step]

	return activity_coords_sample

def getNeighborhoodName(coords):
	lonlat = ",".join(str(x) for x in coords[::-1])
	request_url = mapbox_url + lonlat + ".json?access_token=" + mapbox_access_token

	resp = requests.get(url=request_url)
	data = json.loads(resp.text)
	neighborhood = data["features"][0]["context"][0]["text"]
	return neighborhood

def getPlaces(coords):
	params = dict(
	  client_id=fsq_client_id,
	  client_secret=fsq_client_secret,
	  v='20180311',
	  categoryId = "4d4b7104d754a06370d81259,4d4b7105d754a06373d81259,4d4b7105d754a06374d81259,4d4b7105d754a06376d81259,4d4b7105d754a06377d81259,4d4b7105d754a06378d81259",
	  ll = ",".join([str(x) for x in coords]),
	  radius="50",
	  limit=10
	)

	resp = requests.get(url=fsq_url, params=params)
	data = json.loads(resp.text)

	places = []
	for venue in data["response"]["venues"]:
		place = [venue["name"]]
		
		try:
			place.append(venue["categories"][0]["name"])
		except:
			pass
		
		try:
			place.append(venue["location"]["formattedAddress"][0])
		except:
			pass

		places.append(place)

	return places

activity_coords = getCoords(activity_id, 60)
for i in range(0, len(activity_coords)):
	coordinates = activity_coords[i]
	places = getPlaces(coordinates)

	neighborhood_name = getNeighborhoodName(coordinates)
	print(neighborhood_name + " (" + ", ".join(str(c) for c in coordinates) + ")")

	for place in places:
		print(", ".join(place))
	
	print("\n")
```

[1]: https://itunes.apple.com/us/app/healthfit/id1202650514?mt=8