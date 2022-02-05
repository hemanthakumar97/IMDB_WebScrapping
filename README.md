
# Top Voted Movies WebScraping from IMDB

Python script to scrape the top 100 movies for every year from 2011 to 2021 based on the number of votes


## Steps to follow:
- I am going to scrape https://www.imdb.com
- I will get a list of top 100 movies in an year according to the number votes, for each movie I will fetch the movie_url, duration and number of votes.
- For each movie I will fetch some more important details
- For each year data I will save the CSV file and one file for all years of data inside a *Results* folder.
- For each movie the CSV file contains the data in the below format:
```
title,tagline,no_of_directors,directors,no_of_writers,writers,lead_stars,released_year,oscar_wins,genres,award_nominations,award_wins,rating,votes,metascore,duration,county_of_origins,production_companies,url
Dune,"Beyond fear, destiny awaits.",1,Denis Villeneuve,3,"Jon Spaihts(screenplay by),Denis Villeneuve(screenplay by),Eric Roth(screenplay by)","Timothée Chalamet,Rebecca Ferguson,Zendaya",2021,0,"Action,Adventure,Drama,Sci-Fi",245,88,8.1,471926,74,155,"United States,Canada","Warner Bros.,Legendary Entertainment,Villeneuve Films",https://www.imdb.com/title/tt1160419/
Spider-Man: No Way Home,The Multiverse Unleashed.,1,Jon Watts,3,"Chris McKenna,Erik Sommers,Stan Lee(based on the Marvel comic book by)","Tom Holland,Zendaya,Benedict Cumberbatch",2021,0,"Action,Adventure,Fantasy,Sci-Fi",31,4,8.7,439898,71,148,United States,"Columbia Pictures,Pascal Pictures,Marvel Studios",https://www.imdb.com/title/tt10872600/
```
## Fields to scrape
- Title
- Tagline
- No Of Directors
- Directors
- No Of Writers
- Writers
- Lead Stars
- Released Year
- Oscar Wins
- Genres
- Award Nominations
- Award Wins
- Rating
- Votes
- Metascore
- Duration
- County Of Origins
- Production Companies
- URL

## Install BeautifullSoup and Requests
```
pip install beautifulsoup4 --quiet
pip install requests --quiet
```

## Scrape the list of top voted movies from IMDB
- Use requests to download the page
- Use BeautifulSoup to parse and extract information information
- Use pandas to convert this data into a dataframe and to save it in a CSV file

### Let's write a function to download the page
```
from bs4 import BeautifulSoup
import requests, re, time
import pandas as pd

def get_top50_movies_page(year):
  main_page_url = f"https://www.imdb.com/search/title/?title_type=feature&year={year}-01-01,{year}-12-31&sort=num_votes,desc"
  response = requests.get(main_page_url)
  doc = BeautifulSoup(response.text, 'html.parser')
  return doc
```

`get_top50_movies_page(2021)` uses `requests` to download the page which contains the top 50 movies based on the number of votes for the year of `2021`

```
sample_doc = get_top50_movies_page(2021)
```

### Let's create an another function to prase `title`, `movie_url`, `duration` and `votes` for each movie.

To get the list of all movies in the page, we can pick `div` tags with the `class`
![picture](https://drive.google.com/uc?id=1Iv2XlX6EaWkSWVB7Zyo8AuJWYzvUJK0j)

```
def get_movie_basic_info(doc):
  title_list, movie_url_list, duration_list, votes_list = [], [], [], []
  base_url = 'https://www.imdb.com'

  movies_list = sample_doc.find_all('div', {'class':'lister-item-content'})

  for movie in movies_list:
    title = movie.find('a').text
    movie_url = base_url + movie.find('a')['href']
    duration = int(movie.find('span', {'class': 'runtime'}).text.split()[0])
    votes = int(movie.find('span', {'name': 'nv'}).text.replace(',',''))
    title_list.append(title)
    movie_url_list.append(movie_url)
    duration_list.append(duration) 
    votes_list.append(votes)
  return title_list, movie_url_list, duration_list, votes_list
  ```

#### call the `get_movie_basic_info()`function, it will return top voted 50 movies details list with title, url, duration and ni of votes
  ```
  title_list, movie_url_list, duration_list, votes_list = get_movie_basic_info(sample_doc)
  ```

#### Use the above lists to craete a CSV file using pandas
```
df = pd.DataFrame({'Title': title_list,
                   'URL': movie_url_list,
                   'DUration': duration_list,
                   'Votes': votes_list})
df.to_csv("movies_basic_detail.csv", index=False)
```

### Let's write a function to get more details of the movie
```
def get_movie_details(movie_url):
  d = {}
  response = requests.get(movie_url)
  doc = BeautifulSoup(response.text, 'html.parser')

  d['title'] = doc.find('h1').text
  if 'Taglines' in doc.find_all('li', {'class':'ipc-metadata-list__item'})[13].text:
    d['tagline'] = doc.find_all('span', {'class':'ipc-metadata-list-item__list-content-item'})[1].text
  li_tags = doc.find_all('li', {'class':'ipc-metadata-list__item'})[:33]

  directors = [i.text for i in li_tags[0].find_all('li')]
  d['no_of_directors'] = len(directors)
  d['directors'] = ",".join(directors)
  writers = [i.text for i in li_tags[1].find_all('li')]
  d['no_of_writers'] = len(writers)
  d['writers'] = ','.join(writers)
  d['lead_stars'] = ','.join([i.text for i in li_tags[2].find_all('li')])
  try:
    oscars = li_tags[8].find('a').text
    d['oscar_wins'] = int(re.findall('Won \d+ Oscar', oscars)[0].split()[1])
  except:
    d['oscar_wins'] = 0

  genres =  doc.find('li', {'data-testid':'storyline-genres'}).find('ul').find_all('li')
  d['genres'] = ','.join([i.text for i in genres])
  
  try:
    awards = li_tags[8].find('li').text
  except:
    awards = li_tags[9].find('li').text

  try:
    d['award_nominations'] = int(re.findall('\d+ nomination', awards)[0].split()[0])
  except:
    d['award_nominations'] = 0
  try:
    d['award_wins'] = int(re.findall('\d+ win', awards)[0].split()[0])
  except:
    d['award_wins'] = 0

  d['rating'] = float(doc.find('span', {'class':'AggregateRatingButton__RatingScore-sc-1ll29m0-1'}).text)

  try:
    d['metascore'] = int(doc.find_all('span', {'class':'score'})[2].text.replace(',',''))
  except:
    pass

  countries = doc.find('div', {'data-testid':'title-details-section'}).find('ul').find_all('li')[2].find('ul').find_all('li')
  d['county_of_origins'] = ','.join([i.text for i in countries])
  productions  = doc.find('div', {'data-testid':'title-details-section'}).find('ul').find_all('ul')[-1].find_all('li')
  d['production_companies'] = ','.join([i.text for i in productions])
  d['url'] = movie_url

  return d
  ```
#### Now I will give the `movie_url_list[1]` as input url to the function `get_movie_details(movie_url)`. It will return the dictionary of details for that movie

```
title_list[1], movie_url_list[1]
```
```
Output: ('Spider-Man: No Way Home', 'https://www.imdb.com/title/tt10872600/')
```

```
Spider_Man_movie_data = get_movie_details(movie_url_list[1])
print(Spider_Man_movie_data)
```
```
Output: {'title': 'Spider-Man: No Way Home', 'tagline': 'The Multiverse Unleashed.', 'no_of_directors': 1, 'directors': 'Jon Watts', 'no_of_writers': 3, 'writers': 'Chris McKenna,Erik Sommers,Stan Lee(based on the Marvel comic book by)', 'lead_stars': 'Tom Holland,Zendaya,Benedict Cumberbatch', 'oscar_wins': 0, 'genres': 'Action,Adventure,Fantasy,Sci-Fi', 'award_nominations': 32, 'award_wins': 5, 'rating': 8.7, 'metascore': 71, 'county_of_origins': 'United States', 'production_companies': 'Columbia Pictures,Pascal Pictures,Marvel Studios', 'url': 'https://www.imdb.com/title/tt10872600/'}
```

## Putting all together
- We have a function to get top 50 movies of a particular year, wll add pagination in url to get next 50 movies
- We have a function to parse the more details of each movie

```
from bs4 import BeautifulSoup
import requests, re, time
import pandas as pd

base_url = 'https://www.imdb.com'

all_data = []
def get_main_page_data(year):
  global all_data
  main_page_url = f"https://www.imdb.com/search/title/?title_type=feature&year={year}-01-01,{year}-12-31&sort=num_votes,desc"
  l = []
  for url in [main_page_url, main_page_url+'&start=51&ref_=adv_nxt']:
    response = requests.get(url)
    doc = BeautifulSoup(response.text, 'html.parser')
    movies_list = doc.find_all('div', {'class':'lister-item-content'})
    for movie in movies_list:
      movie_url = base_url + movie.find('a')['href']
      duration = int(movie.find('span', {'class': 'runtime'}).text.split()[0])
      votes = int(movie.find('span', {'name': 'nv'}).text.replace(',',''))
      
      # get the movie details by running get_movie_data() for each movie and add it to a list
      l.append(get_movie_data(movie_url, duration, votes, year))
  all_data.extend(l)
  df = pd.DataFrame(l)

  # Save the top 100 voted movies details inside Resulst folder
  df.to_csv(f'./Results/{year}.csv', index=False)
  ```

### Let's add `year`, `duration` and `votes` fields to the `get_movie_details(url)` function and fetch the more details of the movie
```
def get_movie_data(movie_url, duration, votes, year):
  d = {}
  response = requests.get(movie_url)
  doc = BeautifulSoup(response.text, 'html.parser')

  d['title'] = doc.find('h1').text
  if 'Taglines' in doc.find_all('li', {'class':'ipc-metadata-list__item'})[13].text:
    d['tagline'] = doc.find_all('span', {'class':'ipc-metadata-list-item__list-content-item'})[1].text
  li_tags = doc.find_all('li', {'class':'ipc-metadata-list__item'})[:33]

  directors = [i.text for i in li_tags[0].find_all('li')]
  d['no_of_directors'] = len(directors)
  d['directors'] = ",".join(directors)
  writers = [i.text for i in li_tags[1].find_all('li')]
  d['no_of_writers'] = len(writers)
  d['writers'] = ','.join(writers)
  d['lead_stars'] = ','.join([i.text for i in li_tags[2].find_all('li')])
  d['released_year'] = year
  try:
    oscars = li_tags[8].find('a').text
    d['oscar_wins'] = int(re.findall('Won \d+ Oscar', oscars)[0].split()[1])
  except:
    d['oscar_wins'] = 0

  genres =  doc.find('li', {'data-testid':'storyline-genres'}).find('ul').find_all('li')
  d['genres'] = ','.join([i.text for i in genres])
  
  try:
    awards = li_tags[8].find('li').text
  except:
    awards = li_tags[9].find('li').text

  try:
    d['award_nominations'] = int(re.findall('\d+ nomination', awards)[0].split()[0])
  except:
    d['award_nominations'] = 0
  try:
    d['award_wins'] = int(re.findall('\d+ win', awards)[0].split()[0])
  except:
    d['award_wins'] = 0

  d['rating'] = float(doc.find('span', {'class':'AggregateRatingButton__RatingScore-sc-1ll29m0-1'}).text)
  d['votes'] = votes

  try:
    d['metascore'] = int(doc.find_all('span', {'class':'score'})[2].text.replace(',',''))
  except:
    pass

  d['duration'] = duration
  countries = doc.find('div', {'data-testid':'title-details-section'}).find('ul').find_all('li')[2].find('ul').find_all('li')
  d['county_of_origins'] = ','.join([i.text for i in countries])
  productions  = doc.find('div', {'data-testid':'title-details-section'}).find('ul').find_all('ul')[-1].find_all('li')
  d['production_companies'] = ','.join([i.text for i in productions])
  d['url'] = movie_url

  return d
  ```

 #### Now call the `get_movies_data()` function for year 2011 to 2021 using a for loop to get the top voted 100 movies for each year
```
for year in range(2011,2022):
  get_movies_data(year)
  print(f"{year} is done!")
```

  Yearly based data for each year is stored in CSV files. Use `all_data` variable and create a master sheet which contails all years of data in a single file.
```
df  = pd.DataFrame(all_data)
df.to_csv("./Results/Merged.csv", index=False)
```

I can read the top 5 rows of `Merged.csv` file
```
merged_data = pd.read_csv('./Results/Merged.csv')
print(merged_data.head())
```

## Summary
- Parsed the top voted 100 movies for each year from https://www.imdb.com
- For each movie parsed the more important data and stored a yearly based data in CSV files inside `Results` folder
- Created a one more CSV file which contains all years of data

## Reference
- BeautifullSoup documentation to explore more https://www.crummy.com/software/BeautifulSoup/bs4/doc/

## Ideas for future work

- Use this data for Exploratory Data Analysis


## 🔗 Links
[![portfolio](https://img.shields.io/badge/my_portfolio-000?style=for-the-badge&logo=ko-fi&logoColor=white)](https://hemanthakumar.cf/)
[![linkedin](https://img.shields.io/badge/linkedin-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/hemanthakumar-s-40184713b/)
