UTA Data Bootcamp Module 6
==========================

Riley Taylor  
2024-Jan-30


### Setup and Related Info

--------------------------

### WeatherPy Scipt/Notebook Config/Setup Notes

I have imported the json library to help test API responses so as to be better able to read them. 


I also noted that the setup of this script didn't work the way it probably ought to. At first I thought the citipy library wasn't working properly (and frankly, I do think I have found one error, but I can't seem to replicate it for any other data points, so I assume it's a niche issue that will rarely be encountered, or a user error I missed). However, I realized that the API calls to OpenWeather using a city name in as the query search parameter as described [here](https://openweathermap.org/current), e.g. 

```
url = "http://api.openweathermap.org/data/2.5/weather?"
...
city_url = url + f"appid={weather_api_key}&units=metric&q={city}"
...
response = requests.get(city_url).json()
```

would encounter naming collisions. If the nearest city returned was say, Paris, Texas (population ~24000, so it should technically be a valid result in the citipy library if the source code is accurate (the author set the threshold population at 5000, see [here](https://github.com/wingchen/citipy): )), then the above API call would yield a response that corresponds to Paris, France. 

This makes sense from OpenWeather's perspective - far more people will care about data for Paris, France than Paris, Texas. It's the natural solution to the fact that the city name is duplicated. However, it technically skews the data we originally generated. Instead of aggregating random cities, we aggregate random city names, and then choose the most populous cities that share those names as our actual data set. We should expect our data to skew, albeit slightly, towards large population centers. When I was testing, it was pretty easy to come up with examples of the collisions. Albany, Australia vs Albany, New York, USA. Santa Monica, California, USA vs Santa Monica, Philippines (by the way, this is where I found [the bug](#citipy-bug)).  

There's a fairly simple fix to this naming collision though. The OpenWeather API allows us to make requests by latitude and longitude by using the following query structure (I have dropped the "&units=metric" from the query, choosing to opt for standard because the only difference is that metric occasionally has some units in C instead of K):

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

I prepended test_ to the variable names of larger scope so that I could compare the outputs in separate cells. Essentially, the solution I used was to add another list `test_cities_location_data` that is initialized as the `test_cities` list is initialized. I referenced the source for citipy [here](https://github.com/wingchen/citipy) and saw that there were public fields for latitude, longitude, and country code we could access for a `city` object. So once a `city` object is returned by `nearest_city()`, we can grab the other location data using dot notation, and have it stored in a tuple under the same index in as the relevant city name `test_cities_location_data`. 

But this solves our collision issue. Now we are grabbing weather data using OpenWeather purely off of latitude and longitude of the nearest city to our original random locations. Paris, Texas is included in our analysis, and before it was always excluded. We still have duplicate names being restricted when the list of cities is created (so once we have Paris, Texas, we can't have Paris, France too), but I think this allows for a more random city distribution. 

You could argue that this change was fairly inconsequential. The nature of our selection process is already not random - coastal cities will be selected more often because of lat-lng pairs that result in ocean locations. There will already be a skew, and that's a rather severe one for weather data. Proximity to water/the coast is one of the largest factors in climate.

### Directory and Path Changes

WeatherPy.ipynb is located in the scipts_and_notebooks directory of this repo. To write to the output_data directory, I had to prepend the provided paths with "../" to get the parent directory before checking for the output_data directory. 


### citipy Bug

I had latitude and longitude values that placed me right next to Manila, Philippines, but for some reason, citipy would return Santa Monica as the nearest city. Even though there is a Santa Monica in the Philippines, it is some 100+ miles away from Manila.

If you want to look into the citipy error I encountered, you can try to recreate it yourself:

```
lat = 15.325736570808928
lng = 120.73129576518153
print(f"{citipy.nearest_city(lat, lng).city_name}")
```

When I run this, it returns `santa monica`. Notice that Santa Monica, Philippines is located at (10.12651, 126.04144). Off by about 5 degrees in both latitude and longitude. Citipy does not work as expected here, and I imagine it is due to some flaw in the  Maxmind data set that citipy is based off of. I haven't tested very many other data points and checked, because it's rather time consuming. 

I submitted the bug as an issue in GitHub. 


### WeatherPy Analysis


Note that the functions to generate linear regressions do not impose units onto the y axis labels. I could probably set up a dictionary based off column names to set these, but I didn't figure it was worth it. Units are from the OpenWeather weather api's [standard unit selection](https://openweathermap.org/weather-data). 

For more in depth analysis for the various plots, check WeatherPy.ipynb in the scripts_and_notebooks directory. 



### VacationPy Notes

This assignment caught me a little bit by surprise - I missed a class while sick and I assume they went over how to use hvplot in more detail. I had to research several libraries and documentation for said libraries before breaking down and asking Chat-GPT how I could use the OSM tile to generate a plot from a pandas dataframe. The resulting mess of import statements result from my attempt to try to integrate the code it returned into my notebook.

Still, I can't help but feel like I've failed to fully comprehend how to use those libraries. Relying on AI for documentation explanation is certainly far faster, but I feel like I haven't learned anything. I feel this integral urge to apologize to myself and to the grader for using it, even if using this tool is encouraged.

I'll read more about this [here](https://hvplot.holoviz.org/user_guide/Introduction.html). I think I also got confused when the assignment description specified using the geoViews Python library, but it wasn't in the original import statements. The assignment also specified that "\[t\]he code needed to import the required libraries and load the CSV filed ... is provided to help you get started." 

---------------------------------------------------------------


### Sources - WeatherPy

citipy/citipy/citipy.py at master · wingchen/citipy
https://github.com/wingchen/citipy/blob/master/citipy/citipy.py

Create a Pandas DataFrame from List of Dicts - GeeksforGeeks
https://www.geeksforgeeks.org/create-a-pandas-dataframe-from-list-of-dicts/

Customize Rc — Matplotlib 3.8.2 documentation
https://matplotlib.org/stable/gallery/misc/customize_rc.html#sphx-glr-gallery-misc-customize-rc-py

python - Matplotlib - Border around scatter plot points - Stack Overflow
https://stackoverflow.com/questions/50706901/matplotlib-border-around-scatter-plot-points

Current weather data - OpenWeatherMap
https://openweathermap.org/current

Weather Data - OpenWeatherMap
https://openweathermap.org/weather-data

More Control Flow Tools — Python 3.12.1 documentation
https://docs.python.org/3/tutorial/controlflow.html#defining-functions

pandas.Series.name — pandas 2.2.0 documentation
https://pandas.pydata.org/docs/reference/api/pandas.Series.name.html

matplotlib.pyplot.annotate — Matplotlib 3.8.2 documentation
https://matplotlib.org/stable/api/_as_gen/matplotlib.pyplot.annotate.html

Annotating Plots — Matplotlib 3.8.2 documentation
https://matplotlib.org/stable/gallery/text_labels_and_annotations/annotation_demo.html

pylab_examples example code: annotation_demo.py — Matplotlib 2.0.2 documentation
https://matplotlib.org/2.0.2/examples/pylab_examples/annotation_demo.html

matplotlib.org/2.0.2/mpl_examples/pylab_examples/annotation_demo.py
https://matplotlib.org/2.0.2/mpl_examples/pylab_examples/annotation_demo.py

Identification of severe wind conditions using a Reynolds Averaged Navier-Stokes solver - IOPscience
https://iopscience.iop.org/article/10.1088/1742-6596/75/1/012053


### Sources - VacationPy

Introduction — hvPlot 0.9.2 documentation
https://hvplot.holoviz.org/user_guide/Introduction.html

Projections — GeoViews v1.11.0
https://geoviews.org/user_guide/Projections.html

Points — HoloViews v1.18.1
https://holoviews.org/reference/elements/bokeh/Points.html

Tiles — HoloViews v1.18.1
https://holoviews.org/reference/elements/bokeh/Tiles.html

holoviews.element Package — HoloViews v1.18.1
https://holoviews.org/reference_manual/holoviews.element.html#holoviews.element.Tiles.lon_lat_to_easting_northing

Tiles — HoloViews v1.18.1
https://holoviews.org/reference/elements/bokeh/Tiles.html

ChatGPT
https://chat.openai.com/

python - Boolean subset in pandas - Stack Overflow
https://stackoverflow.com/questions/28748402/boolean-subset-in-pandas

python - How do I get the row count of a Pandas DataFrame? - Stack Overflow
https://stackoverflow.com/questions/15943769/how-do-i-get-the-row-count-of-a-pandas-dataframe

pandas.DataFrame.dropna — pandas 2.2.0 documentation
https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.dropna.html

python - How to add an empty column to a dataframe? - Stack Overflow
https://stackoverflow.com/questions/16327055/how-to-add-an-empty-column-to-a-dataframe

Places API | Developer Documentation | Geoapify
https://apidocs.geoapify.com/docs/places/#url-examples

Places API | Developer Documentation | Geoapify
https://apidocs.geoapify.com/docs/places/#categories

Colormaps — HoloViews v1.18.1
https://holoviews.org/user_guide/Colormaps.html

