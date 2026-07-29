# Geospatial Analysis of Noise experienced by Buildings from the Chicago Transit Authority "El" Trains

> <img src="imgs/apt_view.JPG" width="75%"><br>
> The view from my sophomore year dorm room at Loyola University Chicago <br>
> <i>(A.K.A. my firsthand experience living right next to the 24/7 CTA Red Line)</i>

## Contents:
1. [The Project](#the-project) <br>
2. [The Data](#the-data) <br>
3. [Noise Measurements](#noise-measurements) <br>
4. [My Function](#my-function) <br>
5. [Results](#results) <br>
6. [Appendix](#appendix)


## 1. The Project
This repository is one piece of the data analysis for my Columbia Journalism School master's thesis.

In this project, I conduct a spatial analysis of buildings in Chicago to estiamte the noise each building experiences from nearby Chicago Transit Authority train lines. The project uses sound measurements I took around the city and the Shapely Python package to estimate each building's noise from trains, taking into account the average noise levels around the city.

The ultimate goal of this project is to create a reliable measure of expected noise per building to incorporate into a regression model along with housing price data and other factors that cause rental/home prices to fluctuate (neighborhood desireability/building amenities/proximity to city center/etc.) that will allow me to quantify the "train tax" or the optimal distance to live from the train to maximize accessibility while minimzing noise exposure.

<br>

## 2. The Data
My analysis largely relies on two data sets, both of them public and maintained by the city.

### [Building Footprints](https://data.cityofchicago.org/Buildings/Building-Footprints/syp8-uezg/about_data):
This data set contains information about every building in the city. Most importantly, it includes a geometry element for each building that shows the footprint of that building. This data is the foundation of my analysis and provides the specific locations and shapes of each of the building's I'm seeking to analyze.

The building footprints data came pretty clean from the city. The only change I made was removing NA geometries (6 rows) and translating the remaining geometries from WGS 84 projection to [NAD83/Illinois East](https://epsg.io/3435)*.

You can see my exploratory analysis of this dataset in [1_eda_w_data.ipynb](1_eda_w_data.ipynb).

**Note: I use the EPSG code 26971 to change the Coordinate Reference System, so one unit in my projection = one meter, not one survey foot*.


### [CTA Line Geometries](https://data.cityofchicago.org/Transportation/CTA-L-Rail-Lines/xbyr-jnvx/about_data):
This data set contains 153 line elements, each representing one section of the CTA system's track network. I use it here in conjungtion with the building footprint data to determine how far each building is from the CTA system and which routes it's closest to.

<div align="center">
    <img src="imgs/mapshaper_1.png" width=49%> <img src="imgs/mapshaper_2.png" width=49%><br>
    <p><i>Two segments from the CTA geoJSON file in MapShaper</i></p>
</div>

I interacted with the data using [MapShaper](https://mapshaper.org), an online mapping tool that can parse and visualize geoJSON files. I used the unique `:id` field for each line segment (highlighted in yellow in the above images) to match each segment in a Google Sheet, which I then updated with each line segment's group name. Using this setup, I manually classified each line segment into a group based on two criteria:

### 1. Grouped line segments based on their track type:
For each line segment, I used satellite map imagery and street-view tools in order to determine which type of track section it was:
- "**subway**" if the tracks were underground for that stretch
- "**hwy**" if the tracks were in the median of a highway or immediately adjacent to a highway
- "**eag**" if the tracks were not "subway" or "hwy" types (commonly this looks like train tracks above roadways/alleys, and represents the majority of the CTA system)

I later decided to exclude fully underground track segments from my noise calculation. While trains running underground do not generate *some* noise that is perceptible on the surface, my own experiences in Chicago led me to conclude that this noise is negligible. This is partially due to the fact that trains run underground primarily in the noisier parts of Chicago; the Red and Blue lines, for example, run underground in the central business district (the Loop), where high-rise apartment/condo buildings and increased pedestrian and vehicle traffic mean both that residents are further elevated above subway entrances where noise could escape to the street, and that the subway noise that does escape is competing with higher ambient sound levels.

I chose to distinguish between non-subway lines that are or aren't adjacent to a highway for similar reasons: the ambient noise level near highways is higher due to the increased vehicle traffic. In addition to increased noise from other sources, some stretches of highway in Chicago have additional noise-reduction measures (they are constructed below-grade or with additional noise walls so as to minimize the sound affecting nearby commnities).

### 2. Grouped line segments based on which train lines travel that section
I used maps from the CTA and other transit-focused map views to classify each line segment based on which of the CTA's train routes run along that track section. For example, on the north side of Chicago, one 2-mile section of track carries traffic for the Brown, Purple and Red lines. Other sections (particularly those further from the city center) only carry traffic from one of the CTA's routes.

Later in my analysis I factor in the average number of trains that run on each segment daily, so that the aforementioned 2-mile stretch that is used by three different CTA routes is not generating the same expected noise values as a far less-busy stretch of track that only carries traffic for one route.

### Grouping Results:
Based and the above two criteria, I created 23 distinct track groups. They follow the naming format: **firstTwoLettersOfRouteColors_elevation_Notes_#**

>As an example, this is how I broke the Blue line into four segments. The Blue line **first** travels from the western suburbs into the city center along the highway, then **second** through the Loop on an underground track, **third** out from the city above ground through the near-northwest neighborhoods, and finally **fourth** to O'Hare airport along the highway again.
>
>**bl_hwy_south_12**: "bl_" indicates that the only line using this section is the Blue line. "hwy_" indicates that this particualr stretch runs in the median or a highway or else adjacent to a highway. "south_" is an additional note that indicates this is the southern portion of the Blue line (as opposed to bl_hwy_north_14, which runs from the northwestern neighborhoods to O'Hare airport). "12" simply indicates the order in which I created the track groups, but also creates an additional check in case my other naming criteria all resulted in two track sections having identical names. <br><br>
>**bl_subway_all_11**: As before, "bl_" indicates this track segment is only used by the Blue line. "subway_" shows this track section runs exclusively underground. "all_" here is used to indicate that all sections of the Blue line that run underground are included in this section. And this was the "11"th group I created. <br><br>
>**bl_eag_north_10**: This section is the only one of the Blue line's path that runs above ground and not adjacent to a highway. As such, I use the "eag_" category here. <br><br>
>**bl_hwy_north_9** Finally, this section mirrors the syntax of "bl_hwy_south_12", except that this section is the northern stretch of the Blue line that runs along the highway to O'Hare.

See [Table 1](#table-1-cta-line-segment-groups-with-the-number-of-individual-segments-that-are-included-in-each-group-and-the-average-daiy-number-of-trains-that-run-on-that-section) in the appendix for each group's name, as well as the number of line segments that are included in each group.

### Calculating Average Daily Trains:
As mentioned before, I added another column to my grouped CTA line segment data. This additional column represents the average daily number of trains that run along each track section. This is an important variable to consider because some tracks have a relatively low number of trains that pass daily (like a stretch of the Green line that runs between the Ashland/63rd and Garfield and experiences 122 average daily train trips), and some line segments have significantly more daily traffic (like the northern/eastern sides of the central business district's Loop, which experiences over 1,000 daily train trips).

<div align="center">
    <img src="imgs/1_rail-tt_blue.png" width=49%> <img src="imgs/2_rail-tt_blue.png" width=49%><br>
    <p><i>The front and back sides of the CTA's official Blue line time table brochure</i></p>
</div>

I used official [CTA timetables](/cta_schedule_brochures/) to calculate the number of train trips for each day of the week (most routes have a set schedule for weekday service, a different schedule for Saturday service, and another schedule for Sunday & Holiday service). I then used the occurrences of each of these different days in the calendar year 2025 (how many times holidays fall on Sundays vs other days, etc) and divided that by 365 to get the average daily number of trains.

[Table 2](#table-2-the-number-of-times-each-day-of-the-week-or-holiday-schedule-applied-to-the-cta-trains-in-2025) in the appendix shows the specific counts of each day in 2025. [Table 1](#table-1-cta-line-segment-groups-with-the-number-of-individual-segments-that-are-included-in-each-group-and-the-average-daiy-number-of-trains-that-run-on-that-section) shows the daily average number of train trips made on each of my section groups.

<br>

## 3. Noise Measurements
In order to accurately measure the specific impact of noise from the CTA trains, I needed to measure both the noise of CTA trains and the ambient noise levels around the city. After putting these measured values into formulas used for calculating noise levels at various distances, I could calculate how much louder than the ambient noise level a passing train would sound.

I recorded noise using the National Institute for Occupational Safety and Health's (NIOSH) Sound Level Meter (SLM) app.

FINISH THIS FINISH THIS
<br><br><br><br><br><br><br><br><br><br><br>

<br>

## 4. My Function
The current iteration of this project is my second try at estimating noise for buildings. The code for my first attempt can be found in the [first_attempt_notebooks](/first_attempt_notebooks/) folder.

In this approach, I draw up to 8 lines extending from each building's centroid to more accurately measure noise when it comes from multiple directions (eg. if a building is situated near the intersection of two perpendicular line segments). The below illustration outlines the basic logic of my function:

<img src="imgs/circular_noise_calculations.png" width="90%" caption="my box in relation to Chicago">

The [entire annotated function](#my-full-annotated-function) can be found in the appendix, and the un-annotated version is in [3_running_my_function.ipynb](3_running_my_function.ipynb). I successfully ran my function on all ~800,000 buildings in my data in just under 7 hours.

<br>

## 5. Results


## 6. Appendix:

### Table 1: CTA line segment groups with the number of individual segments that are included in each group and the average daily number of trains that run on that section

| Group Name | # of Line Segments | Avg. Daily Trains |
| --- | --- | --- |
| pu_eag_north_1 | 8 | 305.630137 |
| yl_eag_all_2 | 2 | 152.767123 |
| redpur_eag_north_3 | 13 | 479.901370 |
| br_eag_north_4 | 11 | 320.013699 |
| bropurred_eag_north_5 | 6 | 799.915068 |
| bropur_eag_north_6 | 4 | 401.054795 |
| rd_subway_all_7 | 10 | 398.860274 |
| red_hwy_south_8 | 8 | 398.860274 |
| bl_hwy_north_9 | 9 | 389.901370 |
| bl_eag_north_10 | 5 | 389.901370 |
| bl_subway_all_11 | 8 | 389.901370 |
| bl_hwy_south_12 | 12 | 292.424658 |
| gr_eag_west_13 | 11 | 246.926027 |
| pi_eag_west_14 | 11 | 235.750685 |
| or_eag_mdw_15 | 6 | 264.131507 |
| or_hwy_mdw_16 | 1 | 264.131507 |
| gr_eag_ashland63_17 | 2 | 122.065753 |
| gr_eag_cottgro63_18 | 2 | 124.860274 |
| gr_eag_south_19 | 8 | 246.926027 |
| gror_eag_south_20 | 2 | 511.057534 |
| grpi_eag_west_21 | 4 | 483.312329 |
| brorpipu_eag_loopsouth_22 | 5 | 900.936986 |
| brgrorpipu_eag_loopnorth_23 | 5 | 1147.863014 |

### Table 2: The number of times each day of the week or holiday schedule applied to the CTA trains in 2025
| Day | # in 2025 |
| --- | --- |
| Saturday | 52 |
| Sunday | 52 |
| Monday | 50 |
| Tuesday | 52 |
| Wednesday | 52 |
| Thursday | 50 |
| Friday | 51 |
| Holiday | 6 |

### Table 3: Location, measurement focus, duration, and measured level for my **highway** ambient noise measurements
| Report # | Location | Duration (mm:ss) | dB Level (LAeq) |
| --- | --- | --- | --- |
| [1](/noise_reports/1_hwy_ambient_1.pdf) | 5865–5899 W Railroad Ave | 02:39 | 73.2 |
| [5](/noise_reports/5_hwy_ambient_3.pdf) | 5128 S. Wells St | 03:02 | 79.1 |
| | | **Average:** | **76.15** |

### Table 4: Location, measurement focus, duration, and measured level for my **elevated/at grade** ambient noise measurements
| Report # | Location | Duration (mm:ss) | dB Level (LAeq) |
| --- | --- | --- | --- |
| [4](/noise_reports/4_eag_ambient_2.pdf) | 5213 S. Avers Ave | 03:02 | 67.1 |
| [6](/noise_reports/6_eag_ambient_4.pdf) | E. 54th St & S. Prarie Ave | 03:53 | 64.9 |
| [7](/noise_reports/7_eag_ambient_5.pdf) | 77 E. Adams St | 03:00 | 69.9 |
| [12](/noise_reports/12_eag_ambient_6.pdf) | 6302 N. Winthrop Ave | 02:12 | 57.8 |
| | | **Average:** | **64.93** |

### Table 5: Location, distance from train, train route measured, duration, measured level, and calculated dB at 1-meter from train for each of my train noise measurements
| Report # | Location | Train Route | Distance from train (m) | Duration (mm:ss) | dB Level (LAeq) | Calculated dB @ 1 meter |
| --- | --- | --- | --- | --- | --- | --- |
| [2](/noise_reports/2_train_1_52_ft.pdf) | 2124 S 47th Ct | Pink | 15.85 | 00:11 | 82.6 | 94.60 |
| [3](/noise_reports/3_train_2_40_ft.pdf) | 2124 S 47th Ct | Pink | 12.19 | 00:09 | 79.5 | 90.36 |
| [8](/noise_reports/8_train_3_31_ft.pdf) | 200 W. Randolph St (2nd floor) | Brown | 9.45 | 00:11 | 89.6 | 99.35 |
| [9](/noise_reports/9_train_4_43_ft.pdf) | 200 W. Randolph St (2nd floor) | Purple | 13.11 | 00:17 | 77.7 | 88.88 |
| [10](/noise_reports/10_train_5_75_ft.pdf) | 1110 W. Sheridan Rd (2nd floor) | Purple | 22.86 | 00:07 | 84.9 | 98.49 |
| [11](/noise_reports/11_train_6_47_ft.pdf) | 1110 W. Sheridan Rd (2nd floor) | Red | 14.33 | 00:09 | 80.6 | 92.16 |
| | | | | | **Average:** | **93.97** |

### My full annotated function:

``` python
noise_radius = 802 ## meters
num_radials = 8 ## how many lines I want to draw between the center of the circle and its perimeter – likely to be somewhere around 8
radial_angles = [0, 45, 90, 135, 180, 225, 270, 315] ## equally spaced angles to fill the circle with my radians
cta_line_one_geom = cta_lines['geom'].union_all ## make a unified geom element to check for intersections against, as opposed to iterating through each of the individual cta line segments

hwy_ambient = 76.15 ## dB
eag_ambient = 64.93 ## dB
train_avg = 93.97 ## dB


def radial_noise_calc(row):
    ## could also calculate centroid outside the function as one line of code, but i think this way is pretty fine also
    building = row['geometry']
    building_stories = row['stories']
    centroid = building.centroid
    
    ## get the x and y coordinates of the centroid individually 
    center_x = shapely.get_x(centroid)
    center_y = shapely.get_y(centroid)

    ## create my circle from the building's centroid with a radius based on my noise_radius value
    circle = centroid.buffer(noise_radius)
    
    ## Check if my circle even intersects the CTA line; otherwise it's not close enough to get any noise from trains and can just exit
    if not circle.intersects(cta_line_one_geom):
        ## if the circle as drawn does not intersect with the CTA line geometry, then we can exit at this stage (assume 0 noise from CTA trains for that building)
        return  {'geometry': building,
                 'centroid': centroid,
                 'noise_from_one_train': 0,
                 'daily_noise': 0,
                 'intersected_noise_from_one_train': 0,
                 'intersected_daily_noise': 0,
                 'num_radials': 0}
    
    
    ## Start a blank radials list
    radials = []

    ## Make the radial lines one by one and append them to the radials list ONLY if they intersect the CTA_line_one_geom
    for angle in radial_angles:
        ## convert angles to radians
        angle_radian = np.radians(angle)

        ## find the x and y coordinates for where my radial will intersect the circle
        point_x = center_x + noise_radius * np.cos(angle_radian)
        point_y = center_y + noise_radius * np.sin(angle_radian)

        ## make the radial itself based on the coordinates of the centroid and the x and y coordinates I just made
        radial_line = LineString([centroid, Point(point_x, point_y)])

        ## if the radial line intersects the CTA_line_one_geom, then append it to the radials list
        if radial_line.intersects(cta_line_one_geom):
            radials.append(radial_line)

    num_radials = (len(radials))
    
    ## creating all these variable and setting them equal to 0 before calculating them for each radial and then adding them back to themselves to total for each building
    total_noise_from_one_train = 0
    total_daily_noise = 0
    total_intersected_noise_from_one_train = 0
    total_intersected_daily_noise = 0
    noise_from_one_train = 0
    daily_noise = 0
    intersected_noise_from_one_train = 0
    intersected_daily_noise = 0

    ## now go through my list of radials to calculate the expected noise for the building along that radial
    for radial in radials:
        intersection = radial.intersection(cta_line_one_geom)

        ### FIRST need to deal with how to find the closest intersection to the centroid. It's theoretically possible (though practically unlikely) for my intersection variable to be: a Point (most likely), a MultiPoint (second most likely), or any of a LineString, MultiLineString, or Geometry Collection (all highly unlikely)
        ### Rather than hope some of these geometry types aren't created when I make my intersection variable, I created the logic below to handle each of these cases in case it does happen

        ## if the geometry type is just a point it's simplest (and what I expect to happen most of the time)
        if intersection.geom_type == 'Point':
            intersection_one = intersection

        ## if there are multiple Point-type intersections, get the nearest one (actually want the nearest non-subway one) – likely the second most common option
        elif intersection.geom_type == 'MultiPoint':
            intersection_one = min(intersection.geoms, key=lambda p: radial.project(p))

        ## it also could just be a Line if the radial and the CTA_line_one_geom run on exactly the same path for any length
        ## in that case I can find the shortest distance between the intersecting line and the centroid and then find the point where the shortest distance line intersects with the radial/cta_line_one_geom intersecting line 
        elif intersection.geom_type == 'LineString':
            shortest_line_geom = shortest_line(centroid, intersection)
            intersection_one = shortest_line_geom.intersection(intersection)
        
        ## if it's a MultiLineString I will have to do the same logic as for a single line for each of the lines, then return the minimum of the resulting points
        elif intersection.geom_type == 'MultiLineString':
            
            multiline_intersections = []

            for line in intersection.geoms:
                shortest_line_geom = shortest_line(centroid, line)
                multiline_intersection = shortest_line_geom.intersection(line)
                multiline_intersections.append(multiline_intersection)
            
            intersection_one = min(multiline_intersections, key=lambda p: radial.project(p))
        
        ## and if there is a GeometryCollection I'll have to do a little bit of everything – find the close st of the points and find the closest intersection of the line strings
        elif intersection.geom_type == 'GeometryCollection':
            points = [geom for geom in intersection.geoms if geom.geom_type == 'Point']
            linestrings = [geom for geom in intersection.geoms if geom.geom_type == 'LineString']

            ## first find the single closest point object to the centroid
            if points:
                min_intersection_point = min(points, key=lambda p: radial.project(p))

            ## then do more complex stuff if there are linestrings:
            if linestrings:
                ## create a blank list
                multiline_intersections = []
                
                for line in linestrings:
                    ## find the shortest line btwn the centroid and the line
                    shortest_line_geom = shortest_line(centroid, line)
                    ## find the place the shortest_line and the line intersect (the closest point along the line to the centroid)
                    multiline_intersection = shortest_line_geom.intersection(line)
                    ## then append that point to my list of points called "multiline_intersections"
                    multiline_intersections.append(multiline_intersection)
                ## now take the single closest point in my list to the centroid
                min_intersection_line = min(multiline_intersections, key=lambda p: radial.project(p))
            
            ## then return intersection_one as the smaller of the two points I made in the above steps
            intersection_one = min((min_intersection_point, min_intersection_line), key=lambda p: radial.project(p))
        
        ## make a new version of the radial that only includes the distance between the building centroid and the intersection point on the CTA line
        line_segment = split(radial, intersection_one)
        ## and calculate the distance to the train by taking the length of that line
        distance_to_train = centroid.distance(intersection_one)
        
        ## also make a df of only the buildings that the line intersects (and make sure not to include the original building)
        df_intersected_buildings = df_full_projected[(df_full_projected['geometry'].intersects(line_segment)) &
                                        (not df_full_projected['geometry'].equals(building))]
        
        intersected_buildings = df_intersected_buildings['geometry'].to_list()

        ## and find the # of intersected buildings and the max height of the buildings intersected
        if len(intersected_buildings) > 0:
            max_height = max(df_intersected_buildings['stories'])
        else:
            max_height = 0

        ## NOISE CALCULATION PART:
        for index, cta_row in cta_lines_projected.iterrows():
            segment_geom = cta_row['geometry']
            
            if segment_geom.distance(intersection_one) > 1e-6:
                continue
        
            daily_trains = cta_row['avg_daily_trains']
            
            ## I need the segment type to know which noise threshold to apply (ambient noise values around highways are higher than those not near highways, so I have two different thresholds that apply depending on if the train is in the median of or immediately adjacent to a highway or not)
            segment_type = cta_row['segment_type']
            if segment_type == 'hwy':
                db_threshold = hwy_ambient
            elif segment_type == 'eag':
                db_threshold = eag_ambient
        
            ## now applying the function of (1/r)*(10^(dB/10)) = (10^expected dB at distance/10), where r is the distance from the train and dB is the volume of the train at 1 meter
            ## OR
            ## expected dB at distance = 10*(log10((1/r)*(10^(dB/10))))
            noise_from_one_train = 10*(math.log10((1/distance_to_train)*(10**(train_avg/10))))

            ## if the noise from one train is not higher than my ambient sound threshold, then I can assume that the sound of the train is imperceptible above the ambient noise of the building's surroundings and thereby the expected noise is 0
            if noise_from_one_train <= db_threshold:
                noise_from_one_train = 0
                daily_noise = 0

            ## otherwise, some more calculation needs to be done
            elif noise_from_one_train > db_threshold:
            ## now factoring in the daily avg # of trains on that line segment to estimate the daily total noise from trains
                daily_noise = daily_trains * noise_from_one_train

            ## I'm also using the values stored earlier about the building's height and the height of the tallest building that is intersected by the radial line
            ## if the building is shorter than a building inbetween it and the train, I'm assuming 0 noise (the intersected building blocks all the noise in this assumption)
            if building_stories == 0:
                intersected_noise_from_one_train = 0
                intersected_daily_noise = 0
            elif building_stories > 0:
                if building_stories <= max_height:
                    intersected_noise_from_one_train = 0
                    intersected_daily_noise = 0
                ## otherwise I'm simply calculating the share of noise blocked by the intersected building as one minus the ratio of the in
                elif building_stories > max_height:
                    intersected_noise_from_one_train = noise_from_one_train*(1-(max_height/building_stories))
                    intersected_daily_noise = daily_noise*(1-(max_height/building_stories))
        
        total_noise_from_one_train = total_noise_from_one_train + noise_from_one_train
        total_daily_noise += daily_noise
        total_intersected_noise_from_one_train += intersected_noise_from_one_train
        total_intersected_daily_noise += intersected_daily_noise
    
    return  {'geometry': building,
             'centroid': centroid,
             'noise_from_one_train': total_noise_from_one_train,
             'daily_noise': total_daily_noise,
             'intersected_noise_from_one_train': total_intersected_noise_from_one_train,
             'intersected_daily_noise': total_intersected_daily_noise,
             'num_radials': num_radials}
```