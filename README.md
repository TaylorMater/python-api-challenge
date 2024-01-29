UTA Data Bootcamp Module 6
==========================

Riley Taylor  
2024-Jan-30


### Setup and Related Info

--------------------------

### WeatherPy Scipt/Notebook Config/Setup Notes

I have imported the json library to help test API responses so as to be better able to read them. 


I also noted that the setup of this script didn't work the way it probably ought to. At first I thought the citipy library wasn't working properly (and frankly, I do think I have found one error, but I can't seem to replicate it for any other data points, so I assume it's a niche issue that will rarely be encountered, or a user error I missed). However, I realized that the API calls to openweather using a city name in as the query search parameter, e.g. 

```
url = "http://api.openweathermap.org/data/2.5/weather?"
...
city_url = url + f"appid={weather_api_key}&units=metric&q={city}"
...
response = requests.get(city_url).json()
```

would encounter naming collisions. If the nearest city returned was say, Paris, Texas (population ~24000, so it should technically be a valid result in the citipy library if the source code is accurate (the author set the threshold population at 5000, see [here](https://github.com/wingchen/citipy): )), then the above API call would yield a response that corresponds to Paris, France. 

This makes sense from OpenWeather's perspective - far more people will care about data for Paris, France than Paris, Texas. It's the natural solution to the fact that the city name is duplicated. However, it technically skews the data we originally generated. Instead of aggregating random cities, we aggregate random city names, and then choose the most populous cities that share those names as our actual data set. We should expect our data to skew, albeit slightly, towards large population centers. When I was testing, it was pretty easy to come up with examples of the collisions. Albany, Australia vs Albany, New York, USA. Santa Monica, California, USA vs Santa Monica, Philippines (by the way, this is where I found [the bug](#citipy-bug)).  

There's a fairly simple fix to this naming collision though. The OpenWeather API allows us to make requests by latitude and longitude by using the following query structure:

```
url = "https://api.openweathermap.org/data/2.5/weather?"
query_url = alt_url+f"lat=<latitude>&lon=<longitude>&appid={weather_api_key}"
response = requests.get(query_url).json()
```

The question is, how do we get the latitude and longitude data, or perhaps some other location data that mitigates the naming collision, for the city returned from `citipy.nearest_city(lat,lng)`? In the starter code, we had latitudes and longitudes randomly selected from uniform distributions across some ranges, and then added to `lats[]` and `lngs[]`. But these don't correlate with the nearest city longitudes technically - a location in the middle of the ocean will still have a nearest city, and that city if most definitely not located in the middle of the ocean. 

The simplest, most effective fix I thought of would be to modify the starter code so that instead of having:

```
for lat_lng in lat_lngs:
    city = citipy.nearest_city(lat_lng[0], lat_lng[1]).city_name
    
    # If the city is unique, then add it to a our cities list
    if city not in cities:
        cities.append(city)
```

we instead put the following:

```
test_cities_location_data = []

...

for test_lat_lng in test_lat_lngs:
    city = citipy.nearest_city(lat_lng[0], lat_lng[1])
    city_name = city.city_name
    # If the city is unique, then add it to a our cities list
    if city not in test_cities:
        test_cities.append(city_name)
        test_cities_location_data.append((city.lat, city.lng, city.country_code))
```

I prepended test_ to the variable names of larger scope so that I could compare the outputs in separate cells. Essentially, the solution I used was to add another list `test_cities_location_data` that is initialized as the `test_cities` list is initialized. I referenced the source for citipy [here](https://github.com/wingchen/citipy) and saw that there were public fields for latitude, longitude, and country code we could access for a `city` object. So once a `city` object is returned by `nearest_city()`, we can grab the other location data using dot notation, and have it stored in a tuple under the same index in as the relevant city name `test_cities_location_data`. Alternatively, I could have added the city name as another item in the tuple, but it was already in the starter code so I didn't bother trying to refactor too hard. 

But this solves our collision issue. Now we are grabbing weather data using OpenWeather purely off of latitude and longitude of the nearest city to our original random locations. Paris, Texas is included in our analysis, and before it was always excluded.


### citipy Bug

I had latitude and longitude values that placed me right next to Manila, Philippines, but for some reason, citipy would return Santa Monica as the nearest city. Even though there is a Santa Monica in the Philippines, it is some 100+ miles away from Manila.

If you want to look into the citipy error I encountered, you can try to recreate it yourself:

```
lat = 15.325736570808928
lng = 120.73129576518153
print(f"{citipy.nearest_city(lat, lng).city_name}")
```

When I run this, it returns `santa monica`. Notice that Santa Monica, Philippines is located at (10.12651, 126.04144). Off by about 5 degrees in both latitude and longitude. Citipy does not work as expected here, and I imagine it is due to some flaw in the  Maxmind data set that citipy is based off of. I haven't tested very many other data points and checked, because it's rather time consuming. 




### Sources - WeatherPy

