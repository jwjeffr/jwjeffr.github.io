+++
author = "Jacob Jeffries"
title = "Maps with folium"
date = "2025-04-11"
description = "Guide to emoji usage in Hugo"
tags = ["github", "python"]
+++

# Maps with folium

Anyone that knows me knows I'm a geography nerd, and have been one for as long as I can remember. This is very unfortunate for me, because I'll waste a lot of time doing cool things related to maps.

But, I also love programming, and love GitHub - especially their CI/CD tools using [GitHub Actions](https://docs.github.com/en/actions), as well as their very generous policy of giving you a free web page for every public repo you have using [GitHub Pages](https://pages.github.com/).

My love for geography matured into a love for travel and culture, and today I'll showcase a really cool way to combine everything, i.e.:

- Make a map using `python` (in particular the [folium](https://python-visualization.github.io/folium/latest/) and [geopandas](https://geopandas.org/en/stable/getting_started/introduction.html) libraries)
- Document my adventures on this map
- Deploy the map to GitHub Pages

## Python code

The `python` code to do all of the above is actually very succint:

```py
import folium
import geopandas as gpd


def main():

    # List of countries I've visited (ISO codes)
    visited_countries = {'US', 'MX', 'ES', 'CA', 'KY', 'BZ', 'GT'}
    about_to_visit = {'QA', 'VN', 'DO', 'JM', 'MX'}

    # Load world geometries
    world = gpd.read_file("ne_10m_admin_0_map_subunits.zip")

    # Create a column to indicate visited status
    world['have visited'] = world["ISO_A2_EH"].apply(lambda x: x in visited_countries)
    world['about to visit'] = world["ISO_A2_EH"].apply(lambda x: x in about_to_visit)

    # Create the folium map
    m = folium.Map(location=[20, 0], zoom_start=2)

    # Define colors
    def style_function(feature):
        if feature['properties']['have visited']:
            color = '#2ecc71'
        elif feature['properties']['about to visit']:
            color = '#ffbf00'
        else:
            color = '#bdc3c7'
        return {
            'fillColor': color,
            'color': 'black',
            'weight': 0.5,
            'fillOpacity': 0.7,
        }

    # Add the GeoJSON layer
    folium.GeoJson(
        world,
        style_function=style_function,
        tooltip=folium.GeoJsonTooltip(fields=("NAME", "have visited", "about to visit"))
    ).add_to(m)

    # Save the map
    m.save("map/index.html")


if __name__ == "__main__":

    main()
```

and that's really it on the `python` end. There's more details below, though.

### Transforming Data

First, I need to indicate which countries I want to color on the map:

```py
visited_countries = {'US', 'MX', 'ES', 'CA', 'KY', 'BZ', 'GT'}
about_to_visit = {'QA', 'VN', 'DO', 'JM', 'MX'}
```

I've defined the countries I have visited already, as well as the countries I plan on visiting soon which are denoted with [ISO-3166](https://en.wikipedia.org/wiki/List_of_ISO_3166_country_codes) country codes. After, I read from an external data source that contains the geometries of the map:

```py
world = gpd.read_file("ne_10m_admin_0_map_subunits.zip")
```

There are actually a lot of different geometries you could download. But, the one I downloaded is from [Natural Earth](https://www.naturalearthdata.com/downloads/10m-cultural-vectors/10m-admin-0-details/), or directly [here](https://www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/cultural/ne_10m_admin_0_map_subunits.zip) (this will download the file!). This creates a `geopandas.GeoDataFrame` object, which behaves a lot like a `pandas.DataFrame` object - i.e. I can add new columns! In these lines:

```py
world['have visited'] = world["ISO_A2_EH"].apply(lambda x: x in visited_countries)
world['about to visit'] = world["ISO_A2_EH"].apply(lambda x: x in about_to_visit)
```

I am simply defining new columns that indicate whether I have visited the country already, or if I am about to visit it, denoted by a boolean `True` or `False`.

### Visualizing Data

Now we get to do the fun part - putting our data on a map. First, I define a `folium` map, as well as a style function:

```py
m = folium.Map(location=[20, 0], zoom_start=2)

def style_function(feature):
    if feature['properties']['have visited']:
        color = '#2ecc71'
    elif feature['properties']['about to visit']:
        color = '#ffbf00'
    else:
        color = '#bdc3c7'
    return {
        'fillColor': color,
        'color': 'black',
        'weight': 0.5,
        'fillOpacity': 0.7,
    }
```

Here, `m` is simply the map object we'll color, and `style_function` defines how we will color the map. These are simply hex codes. Then, I add a layer which styles the map:

```py
folium.GeoJson(
    world,
    style_function=style_function,
    tooltip=folium.GeoJsonTooltip(fields=("NAME", "have visited", "about to visit"))
).add_to(m)
```

and I save the resulting map as an `html` file:

```py
m.save("map/index.html")
```

and that's all! This creates a file that looks something like this:

<div style="position:relative; padding-bottom:56.25%; height:0; overflow:hidden;">
  <iframe src="https://jwjeffr.github.io/visited-countries/index.html" 
  style="position:absolute; top:0; left:0; width:100%; height:100%; border:none;"></iframe>
</div>

### Deploying with GitHub pages

Now that we have an `html` file, we can use GitHub actions to deploy it to a web page! All we need is a file `.github/workflows/publish.yml` file, which defines the things GitHub needs to do to build the page and push it:

```yaml
name: publish

on:
  push:
    branches:
      - main

jobs:

  make-map:
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: False
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Install dependencies
        run: >-
          pip install -r requirements.txt
      - name: Build map
        run: >-
          mkdir map; python make-map.py
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          name: github-pages
          path: map/

  deploy-map:
    needs:
      - make-map
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write

    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Download map artifact
        uses: actions/download-artifact@v4
        with:
          name: github-pages
          path: map/

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

Here, we have the `make-map` step which makes the map and uploads it as an artifact, and then the `deploy-map` step which downloads the artifact, and pushes it to GitHub pages. For me, the resulting page is hosted [here](https://jwjeffr.github.io/visited-countries), and the corresponding GitHub repository is [here](https://github.com/jwjeffr/visited-countries). Feel free to use my repository as a template!