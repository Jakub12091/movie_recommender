Program prepares recommendations using TMDB database (https://www.themoviedb.org) 
![image](https://user-images.githubusercontent.com/115424802/230683372-15697587-577d-4b82-b8d3-daced7944190.png)

You can use already existing datafile or get the most up-to-date data using an API key.

Recommendations are made based on two pieces of information obtained from the user: favorite director and favorite actor. The director information allows the program to get the type of movies the user likes, while the favorite actor will be related to the ROI indicator.

One of the goals is to suggest as many movies as possible that the user has not yet seen. So, choosing Christopher Nolan as a favourite director and Harrison Ford as a favourtie actor will result in excluding Nolan’s movies, movies starring H. Ford and, inter alia, every Star Wars movie - despite the fact H. Ford doesn’t act as Han Solo in every one of them, the user is probably familiar with those movies. Please see the details below. 

<b>1. get_data() function </b>

Getting the most up-to-date data using TMDB API key requires making many requests. Function "get_data" contains a few smaller functions that get needed data and merges it into one dataframe.

The first function "api_absorb_ids" gets info about the 1000 most voted movies, one row describes one movie. Absorbed info is not complex enough, we use this function only for movie’s IDs. A single request gets info about 20 movies, so 50 requests will be made. Once it's done, we will have 1000 IDs. 

The second function "api_get_details" makes 1000 requests, using every ID, to get detailed information about the movie, like budget, revenue, etc. One of the most important columns (genres) contains nested data: 

![image](https://user-images.githubusercontent.com/115424802/230684555-fc6629bf-e2c3-422d-9b94-77bd14a2a01c.png)

Program uses pd.json_normalize, groupby and aggregation to get genres names data into a separate dataframe and merges it with the main dataframe:

![image](https://user-images.githubusercontent.com/115424802/230684754-399110de-4492-4327-9b93-a848d6f1175f.png)

Function "api_get_artists" absorbs info about the cast and crew of every movie using IDs while requesting. Absorbed data is divided into columns ‘cast’ and ‘crew’:

![image](https://user-images.githubusercontent.com/115424802/230684847-f0a626cf-8df0-4133-ba3e-f5776fb551ae.png)

Function gets most important jobs from crew columns ('Director','Screenplay','Director of Photography') and actors from the cast column, then merges new data with the primary dataframe:

![image](https://user-images.githubusercontent.com/115424802/230684977-050d5705-fa09-42bd-8136-4abd758011b9.png)

(later columns will be renamed, so there will be no whitespaces or capitalized letters)

Before returning dataframe, "get_data" will create an additional column "ROI", describing return on budget (revenue / budget). Some cleaning will be done too - making sure there are no duplicates, turning "0" in numeric columns into pd.NA, replacing " " in string columns with pd.NA, dropping rows with pd.NA in crucial columns.
Finally, function "get_data" will return a dataframe prepared to be used in the recommendation process.

<b>2. preferences and masks </b>

The second chunk of code contains another functions. 

First of them, "get_users_preferences" will obtain user’s favourite director (skippable) and favourite actor. This function will make a list of the director's most common genres (maximum 4, list called "favourite genres") using value_counts. If no director was obtained, genres list will be made based on the favourite actor. Function returns dict containing favourite director, favourite actor, favourite genres, and number of favourite genres that will be used while creating recommendations:

![image](https://user-images.githubusercontent.com/115424802/230685738-41ac8532-6ef0-4373-8b83-6050ed059047.png)

Function "create_genres_mask" returns a mask based on favourite genres. While preparing recommendations, the number of genres included in mask might be reduced in order to get more recommendations - function will be called again, but decreased genres_to_use parameter will be passed to it. The first mask created, including all favourite genres, will be most specific, for example: if user’s favourite director is Christopher Nolan, then only movies described by all 4 genres 'Action', 'Drama', 'Thriller', 'Science Fiction' will be recommended:

![image](https://user-images.githubusercontent.com/115424802/230685930-f016f505-9929-40d9-a387-a2c95ef27ef9.png)

After decreasing genres_to_use parameter, 'Science Fiction' will be excluded from the mask and more movies might be recommended.

"create_ROI_mask" creates an ROI threshold, based on favourtie actor. ROI describes the financial success of a movie, but many factors affect it: budget, marketing, casting, quality of the movie, etc. That’s why I decided to use the ROI threshold as a determinant of "popularity mixed with quality" for each movie. Why is the ROI threshold based on an actor? Most often, people watch more then one movie starring their favourite actor, so that is a valuable clue (at the same time, many people are not able to name their favourite director). 
ROI threshold is made using median ROI of all movies an actor starred in. Median choice is obvious here - there might be ROI outliers like Avatar. 
Also, it is worth mentioning that in our case ROI is a better indicator than budget or revenue alone because of inflation over the years. 
ROI mask might be excluded in order to get more recommendations. 

"create_collection_mask" function excludes movies that are part of a collection that is probably already known by user (if a favourtie director created one of the collection’s movies or favourite actor starred in one of these movies - whole collection is excluded from recommendations)

<b>3. movie recommendations </b>

Finally, function "movie_recommendation" calls functions described in subsection two to get preferences and create masks. Also, - empty dataframe (df_seen_already) is created that stores already seen recommendations, so the user will not see the same recommendation twice during one session. Not only the masks mentioned above are used, but also vote_average mask, which excludes movies with an average rating below 6.5. 

The first recommendations are prepared using all of the masks. If the user is not satisfied with the recommendations or there are no movies meeting the criteria - another set of recommendations is made, but without ROI threshold. After that, it is possible to get even more recommendations if the user wants to (or if still no movies meet the criteria) - this time recommendations will be prepared using a reduced genres mask: create_genres_mask will be called again, but passed parameter describing number of genres to use is going to be reduced by one. ROI threshold will be included again. If the user is still not satisfied, another set of recommendations will be made without the ROI mask. This loop might last until there are no more genres to reduce in the genres mask. If this happens - user will be informed and given the number of seen recommendations. 

As mentioned before, first set of recommended movies is very specific and adjusted to the user's preferences. The last possible set, being only dependent on one genre, might be very diversified. Before seeing the last possible set, most users should definitely find a movie worth watching among the already seen recommendations. 

Below you may see an example of how the program displays recommendations:

![image](https://user-images.githubusercontent.com/115424802/230686956-2f411eae-6c7b-4e11-b6b1-a0786120314f.png)
