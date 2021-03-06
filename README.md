# Sentinel5P


Berlin NO2 (year avg)           |  Istanbul NO2 (year avg)         | Moscow NO2 (year avg)
:-------------------------:|:-------------------------:|:-------------------------:
![Berlin NO2](/images/Berlin_L2__NO2___.png?raw=true "Berlin NO2")  |  ![Istanbul NO2](/images/Istanbul_L2__NO2___.png?raw=true "Istanbul NO2") | ![Moscow NO2](/images/Moscow_L2__NO2___.png?raw=true "Moscow NO2")

Some scripts that help you downloading and processing satellite data from L2 to L3 products form Sentinel-5P mission:  

"perform atmospheric measurements with high spatio-temporal resolution, to be used for air quality, ozone &amp; UV radiation, and climate monitoring &amp; forecasting."  

Note that L2 products are processed measurements provided by [ESA](https://sentinels.copernicus.eu/web/sentinel/missions/sentinel-5p) and L3 are the projection of these measurements into a common grid  or [raster](https://desktop.arcgis.com/en/arcmap/10.3/manage-data/raster-and-images/what-is-raster-data.htm)

## Contents
- [Download Data](#Download-Data) (using [sentinelsat](https://sentinelsat.readthedocs.io/en/stable/api.html))
- [Create a Common Grid](#Processing-Data-with-Harp-to-Create-a-Common-Grid) (using [Harp](http://stcorp.github.io/harp/doc/html/python.html))
- [Stack Grids into Time Dimension](#Stack-Grids-into-Time-Dimension) (using [xarray](http://xarray.pydata.org/en/stable/why-xarray.html))
- [Data Summary](#Data-Summary)

## Download Data
Sentinel-5P data need a user and password but by August 2020 it is still only a guest one: s5pguest/s5pguest

To download automatically all products available for a certain region, add the region to the dict called `polys` in function `prepare_download` and use the following syntax:
```
python download.py [-h] -c CITY -f FOLDER [-l LEVEL]
```
where `CITY` is the key of the dict in the added region and `FOLDER` is the path to save the data. Use `LEVEL` to download either `L2` or `L1B` products (check the available products in the [official website](https://sentinels.copernicus.eu/web/sentinel/technical-guides/sentinel-5p/products-algorithms)). The script is set to download data from Jan 1st, 2019 to Dec 31st, 2019, if you want data to be in another range, set the variable `date_range` in function `prepare_download` to de desired period.

Unfortunately, The commented products in the dict called `products` in function `prepare_download` did not download successfully.

## Processing Data with Harp to Create a Common Grid
The downloaded data consist of the orbit of the satellite when it passes the specified region. Thus, it contains much more spatial information than the one we are interested in. Furthermore, the locations in the array containing the measurements are not always the same, which means that we want further process the data to have a common grid always representing the same latitude and longitude.

The following script uses the library [Harp](http://stcorp.github.io/harp/doc/html/python.html) to filter our area of interest and create a common grid between different dates and products.
```
python mk_raster.py [-h] -c CITY -p PRODUCT [-f FOLDER] [-f_src FOLDER_SRC] [-d DEGREES]
```
Where `CITY` should be a folder containing the former downloaded data in a city under the parent folder `FOLDER_SRC` (by default `../data`). `PRODUCT` should be one of the downloaded products (e.g., L2__O3____, L2__NO2___, etc.). Finally, `FOLDER`is the target directory to save the processed data by harp (by default it is `../data/crop`). Use `DEGREES` to set the spatial cover of each pixel (by default is set to [0.01~1110m](https://www.usna.edu/Users/oceano/pguth/md_help/html/approx_equivalents.htm#:~:text=1%C2%B0%20%3D%20111%20km%20(or,0.001%C2%B0%20%3D111%20m) )).

Note that the name of the main variable for each of the products is not the same in the data provided by Sentinel-5P and `Harp` products. To find the corresponding names one should check the specific documentation of [S5P in Harp's library](http://stcorp.github.io/harp/doc/html/ingestions/index.html#sentinel-5p-products) or follow the variables that [Google Earth Engine](https://developers.google.com/earth-engine/datasets/catalog/sentinel-5p) uses when converting L2 products to L3 also using `Harp`. As an example, in S5P data, the variable called `nitrogendioxide_tropospheric_column` of `L2__NO2___` product is called `tropospheric_NO2_column_number_density` in `Harp`.

## Stack Grids into Time Dimension

Once we have a common grid for all days in a product, we stack them by the time dimension. This way, we have a single multidimensional object for each of the products instead of single files per day.

The following script would do that for you:
```
python join_by_time.py [-h] -c CITY -p PRODUCT [-f FOLDER] [-f_grid FOLDER_GRID] [-f_src FOLDER_SRC]
```
Where `FOLDER_GRID` is the folder containing the files with a common grid (`../data/crop` with the above example), `FOLDER` is the directory to save the new stacked objects (`../data/final_tensors` by default), and all other parameters are the same as explained before.

As an example, if we consider the product `L2__NO2____`, the script will generate 4 outputs:
- `L2__NO2___.nc`, the year multidimensional [netcdf](http://xarray.pydata.org/en/stable/io.html) file (use this if you want to further process this data)
- `L2__NO2____data.h5`, the values of the product with shape `(date, longitude, latitude)` (use this directly with Pytorch, Numpy, Tensorflow...)
- `L2__NO2____time.h5`, the dates in the same order as the `date` dimension in `L2__NO2____data.h5`
- `Moscow_L2__NO2___.png`, a plot of the average NO2 in over all dates available

**Remark**: Using the above script might result in a non-responding program due to the still open issues related to this warning: `RuntimeWarning: invalid value 
encountered in true_divide
x = np.divide(x1, x2, out)`.
However, if you are using a notebook or a `ipython` (as myself with [Visual Studio Code](https://code.visualstudio.com/docs/python/jupyter-support-py)), the following script allows you to **do the same with no problems** and for all areas of interest and products at once. The pitfall is that you need to modify the code if you use different regions of interest than Moscow, Istanbul, and Berlin, which are the only ones currently defined in the code.
```
python join_by_time_interactive.py
```

## Data Summary

Using the tools explained above I got the following data in the specified areas of interest. As an example, for Moscow's O3 in 2019, I downloaded 799 orbits (`download.py`), but only 481 contained data for the city bounding box (`mk_raster.py`), the other orbits contained only NaN values. However, there is more than one orbit per day since they overlap. Averaging the orbits of the same day gives us a total of 250 unique days (`join_by_time_interactive.py`), which means that there are many days in 2019 with no data... Also, take into account that despite having data in one day, there are spatial locations for that day without data represented by NaNs. Note that CH4 is the product that has less available data from the ones below.

City | Product | Unique Days | Orbits with Data | Total Orbits
:----:|:----:|:----:|:----:|:----:|
Moscow | L2__O3____ | 250 | 481 | 799
Moscow | L2__NO2___ | 353 | 682 | 1062
Moscow | L2__SO2___ | 324 | 559 | 1062
Moscow | L2__CH4___ | 69 | 90 | 1061
Moscow | L2__HCHO__ | 336 | 600 | 1060
Istanbul | L2__O3____ | 250 | 358 | 390
Istanbul | L2__NO2___ | 363 | 519 | 565
Istanbul | L2__SO2___ | 356 | 506 | 565
Istanbul | L2__HCHO__ | 354 | 503 | 564
Berlin | L2__O3____ | 250 | 449 | 616
Berlin | L2__NO2___ | 360 | 636 | 823
Berlin | L2__SO2___ | 335 | 547 | 823
Berlin | L2__CH4___ | 82 | 87 | 820
Berlin | L2__HCHO__ | 340 | 566 | 822

## Disclaimer
I am new to the field of air-quality measurements and satellite images, and it is also the first time using the tools mentioned above (harp & xarray). Part of the code might be wrong and many improvements could be done. I share this repo since I believe it could benefit both, new people trying to work with this data and myself, from your possible suggestions :)
