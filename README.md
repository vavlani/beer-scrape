
## Scrape beer information off BeerAdvocate using BeautifulSoup4

In this walkthrough, we will scrape beer reviews and data fields off [beeradvocate.com](https://www.beeradvocate.com/)

Beer Advocate is a website which collects reviews and data about beers from all over the world. They rate these beers based on user feedback and provides useful information about the type of beer, the brewery which produces it and the region where it is produced.

We are going to be utilizing [BeatifulSoup4](https://www.crummy.com/software/BeautifulSoup/bs4/doc/) which makes parsing HTML documents very easy and also provides the `Request` class which helps us get the web page using the url.

### _At the time of scraping, the website provided a list of about ~6.5k different beers_

### Main concepts used: web scraping, multi-threading

### Output of the exercise: beer_advocate.csv

We are going to create a dataset which has the following pieces of information:
- **beer_name**: _name of the beer_
- **brewery**: _name of the brewery_
- **ba_score**: _beer advocate score: scale of 1 to 5_
- **ranking**: _overall ranking of the beer_
- **reviews**: _number of reviews by users_
- **ratings**: _number of ratings by users_
- **pdev**: _percent deviation in ratings_
- **wants**: _no of users who want this beer_
- **gots**: _no of users who have had/got this beer_
- **trade**: _no of users interested in a trade_
- **location**: _location of the beer or region_
- **style**: _beer style_
- **alcohol**: _alcohol content_
- **availability**: _is it available?_
- **note_desc**: _detailed description of the beer_



```python
from bs4 import BeautifulSoup
from collections import namedtuple
import requests
import re
import pandas as pd
import threading
```


```python
# We define a namedtuple for the beer instead of a class
Beer = namedtuple('Beer', ['beer_name', 'brewery', 'ba_score', 'ranking', 'reviews', 'ratings', 'pdev',
                           'wants', 'gots', 'trade', 'location', 'style', 'alcohol', 'availability',
                           'note_desc'])
```


```python
def scrape_beer_page(page_url):
    """
    THe function scrapes the beer information from a beer page. 
    
    :param page_url: beer page to scrape
    :return: beer object which contains the scraped information
    """

    page_response = requests.get(page_url, timeout=5)

    full_soup = BeautifulSoup(page_response.content, 'lxml')

    title = full_soup.find('div', class_='titleBar')

    beer_name = list(title.h1.strings)[0].strip()
    brewery = title.span.string.replace('|', '').strip()

    ba_content = full_soup.find(id='ba-content')
    ba_score = ba_content.find(id='score_box').find('span', class_='ba-ravg').string

    item_stats_content = ba_content.find(id='item_stats')

    stats = ['Ranking', 'Reviews', 'Ratings', 'pDev', 'Wants', 'Gots', 'Trade']

    for elem in item_stats_content.find_all('dt'):
        stat_name = elem.next_element.string.replace(':', '')
        stat_value = elem.find_next('dd').string.strip()
        if stat_name == 'Ranking':
            ranking = stat_value
        elif stat_name == 'Reviews':
            reviews = stat_value
        elif stat_name == 'Ratings':
            ratings = stat_value
        elif stat_name == 'pDev':
            pdev = stat_value
        elif stat_name == 'Wants':
            wants = stat_value
        elif stat_name == 'Gots':
            gots = stat_value
        elif stat_name == 'Trade':
            trade = stat_value

    # Info-box
    info_box = full_soup.find(id='info_box')

    location_marker = info_box.find(string=re.compile("Brewed by"))
    location = location_marker.find_next('a').find_next('a').string

    style_marker = info_box.find('b', string='Style:')
    style = style_marker.find_next('a').string.strip()

    alc_marker = info_box.find('b', string=re.compile("Alcohol by"))
    alcohol = alc_marker.next_sibling.strip()

    avail_marker = info_box.find('b', string=re.compile("Availability"))
    availability = avail_marker.next_sibling.strip()

    note_marker = info_box.find('b', string=re.compile("Notes / Commercial"))
    note_desc = note_marker.find_next('br').next_element

#     print(f'Beer Name: {beer_name}')
#     print(f'Brewery: {brewery}')
#     print(f'BA Score: {ba_score}')

#     print(f'Ranking: {ranking}')
#     print(f'Reviews: {reviews}')
#     print(f'Ratings: {ratings}')
#     print(f'pDev: {pdev}')
#     print(f'Wants {wants}')
#     print(f'Gots: {gots}')
#     print(f'Trade: {trade}')

#     print(f'Location: {location}')
#     print(f'Style: {style}')
#     print(f'Alcohol Percentage: {alcohol}')
#     print(f'Availability: {availability}')
#     print(f'Notes: {note_desc}')

    return Beer(beer_name, brewery, ba_score, ranking, reviews, ratings, pdev,
                wants, gots, trade, location, style, alcohol, availability,
                note_desc)
```


```python
page_url = 'https://www.beeradvocate.com/beer/profile/16043/112329/'
beer = scrape_beer_page(page_url)
print(beer)
```

    Beer(beer_name='Chrysopolis', brewery='Birrificio Del Ducato', ba_score='4.1', ranking='-', reviews='19', ratings='106', pdev='10.24%', wants='3', gots='17', trade='0', location='Italy', style='Belgian Lambic', alcohol='5.00%', availability='Limited (brewed once)', note_desc='\nNo notes at this time.')
    


```python
def scrape_beer_urls(page_url):
    """
    This function gets the urls from a list of beer urls and puts them in a list.
    """
    page_response = requests.get(page_url, timeout=5)

    full_soup = BeautifulSoup(page_response.content, 'lxml')

    beer_table = full_soup.find('div', id='ba-content').table

    def is_a_and_parent_is_td(tag):
        return tag.parent.name == 'td' and tag.name == 'a'

    all_beers = beer_table.find_all(is_a_and_parent_is_td)
    beer_urls = []

    for beer_tag in all_beers:
        beer_urls.append('https://www.beeradvocate.com' + beer_tag.get('href'))
        print('https://www.beeradvocate.com' + beer_tag.get('href'))

    return beer_urls
```


```python
url = 'https://www.beeradvocate.com/lists/top/'
beer_urls = scrape_beer_urls(url)
```

    https://www.beeradvocate.com/beer/profile/23222/78820/
    https://www.beeradvocate.com/beer/profile/26/42349/
    https://www.beeradvocate.com/beer/profile/25888/87246/
    https://www.beeradvocate.com/beer/profile/28743/87846/
    https://www.beeradvocate.com/beer/profile/17981/110635/
    https://www.beeradvocate.com/beer/profile/46317/16814/
    https://www.beeradvocate.com/beer/profile/28743/146770/
    https://www.beeradvocate.com/beer/profile/28743/237238/
    https://www.beeradvocate.com/beer/profile/2216/188570/
    https://www.beeradvocate.com/beer/profile/33824/172669/
    https://www.beeradvocate.com/beer/profile/23222/162502/
    https://www.beeradvocate.com/beer/profile/863/21690/
    https://www.beeradvocate.com/beer/profile/23222/76421/
    https://www.beeradvocate.com/beer/profile/32409/271487/
    https://www.beeradvocate.com/beer/profile/28743/122114/
    https://www.beeradvocate.com/beer/profile/1146/57747/
    https://www.beeradvocate.com/beer/profile/28743/207976/
    https://www.beeradvocate.com/beer/profile/36710/263446/
    https://www.beeradvocate.com/beer/profile/1199/47658/
    https://www.beeradvocate.com/beer/profile/28743/86237/
    https://www.beeradvocate.com/beer/profile/32319/168050/
    https://www.beeradvocate.com/beer/profile/17980/64545/
    https://www.beeradvocate.com/beer/profile/396/123286/
    https://www.beeradvocate.com/beer/profile/28743/259249/
    https://www.beeradvocate.com/beer/profile/32319/157262/
    https://www.beeradvocate.com/beer/profile/388/5281/
    https://www.beeradvocate.com/beer/profile/28743/174063/
    https://www.beeradvocate.com/beer/profile/23222/78660/
    https://www.beeradvocate.com/beer/profile/28019/136652/
    https://www.beeradvocate.com/beer/profile/31805/77563/
    https://www.beeradvocate.com/beer/profile/32319/126669/
    https://www.beeradvocate.com/beer/profile/863/7971/
    https://www.beeradvocate.com/beer/profile/17980/120372/
    https://www.beeradvocate.com/beer/profile/20681/115317/
    https://www.beeradvocate.com/beer/profile/22511/58299/
    https://www.beeradvocate.com/beer/profile/23222/113674/
    https://www.beeradvocate.com/beer/profile/18149/51116/
    https://www.beeradvocate.com/beer/profile/388/3659/
    https://www.beeradvocate.com/beer/profile/31540/169616/
    https://www.beeradvocate.com/beer/profile/2210/41815/
    https://www.beeradvocate.com/beer/profile/13371/209691/
    https://www.beeradvocate.com/beer/profile/22511/69522/
    https://www.beeradvocate.com/beer/profile/1146/53980/
    https://www.beeradvocate.com/beer/profile/32319/104649/
    https://www.beeradvocate.com/beer/profile/1199/19960/
    https://www.beeradvocate.com/beer/profile/1146/10672/
    https://www.beeradvocate.com/beer/profile/313/1545/
    https://www.beeradvocate.com/beer/profile/30654/186936/
    https://www.beeradvocate.com/beer/profile/22511/126517/



```python
def scrape_beer_lists(page_url):
    """
    This scrapes the urls of the beer lists on the right pane of the page
    """
    page_response = requests.get(page_url, timeout=5)

    full_soup = BeautifulSoup(page_response.content, 'lxml')

    lists_section = full_soup.find('div', class_='secondaryContent')

    def is_a_and_href_list(tag):
        return tag.name == 'a' and tag.get('href').startswith('/lists/')

    all_beers = lists_section.find_all(is_a_and_href_list)
    beer_lists = []

    for beer_tag in all_beers:
        beer_lists.append('https://www.beeradvocate.com' + beer_tag.get('href'))
        # print('https://www.beeradvocate.com' + beer_tag.get('href'))

    return beer_lists
```


```python
link = 'https://www.beeradvocate.com/lists/top/'
scrape_beer_lists(link)
```




    ['https://www.beeradvocate.com/lists/top/',
     'https://www.beeradvocate.com/lists/new/',
     'https://www.beeradvocate.com/lists/fame/',
     'https://www.beeradvocate.com/lists/popular/',
     'https://www.beeradvocate.com/lists/bottom/',
     'https://www.beeradvocate.com/lists/us/',
     'https://www.beeradvocate.com/lists/great-lakes/',
     'https://www.beeradvocate.com/lists/mid-atlantic/',
     'https://www.beeradvocate.com/lists/midwest/',
     'https://www.beeradvocate.com/lists/mountain/',
     'https://www.beeradvocate.com/lists/new-england/',
     'https://www.beeradvocate.com/lists/northwest/',
     'https://www.beeradvocate.com/lists/pacific/',
     'https://www.beeradvocate.com/lists/south/',
     'https://www.beeradvocate.com/lists/south-atlantic/',
     'https://www.beeradvocate.com/lists/southwest/',
     'https://www.beeradvocate.com/lists/state/al/',
     'https://www.beeradvocate.com/lists/state/ak/',
     'https://www.beeradvocate.com/lists/state/az/',
     'https://www.beeradvocate.com/lists/state/ar/',
     'https://www.beeradvocate.com/lists/state/ca/',
     'https://www.beeradvocate.com/lists/state/co/',
     'https://www.beeradvocate.com/lists/state/ct/',
     'https://www.beeradvocate.com/lists/state/de/',
     'https://www.beeradvocate.com/lists/state/dc/',
     'https://www.beeradvocate.com/lists/state/fl/',
     'https://www.beeradvocate.com/lists/state/ga/',
     'https://www.beeradvocate.com/lists/state/hi/',
     'https://www.beeradvocate.com/lists/state/id/',
     'https://www.beeradvocate.com/lists/state/il/',
     'https://www.beeradvocate.com/lists/state/in/',
     'https://www.beeradvocate.com/lists/state/ia/',
     'https://www.beeradvocate.com/lists/state/ks/',
     'https://www.beeradvocate.com/lists/state/ky/',
     'https://www.beeradvocate.com/lists/state/la/',
     'https://www.beeradvocate.com/lists/state/me/',
     'https://www.beeradvocate.com/lists/state/md/',
     'https://www.beeradvocate.com/lists/state/ma/',
     'https://www.beeradvocate.com/lists/state/mi/',
     'https://www.beeradvocate.com/lists/state/mn/',
     'https://www.beeradvocate.com/lists/state/ms/',
     'https://www.beeradvocate.com/lists/state/mo/',
     'https://www.beeradvocate.com/lists/state/mt/',
     'https://www.beeradvocate.com/lists/state/ne/',
     'https://www.beeradvocate.com/lists/state/nv/',
     'https://www.beeradvocate.com/lists/state/nh/',
     'https://www.beeradvocate.com/lists/state/nj/',
     'https://www.beeradvocate.com/lists/state/nm/',
     'https://www.beeradvocate.com/lists/state/ny/',
     'https://www.beeradvocate.com/lists/state/nc/',
     'https://www.beeradvocate.com/lists/state/nd/',
     'https://www.beeradvocate.com/lists/state/oh/',
     'https://www.beeradvocate.com/lists/state/ok/',
     'https://www.beeradvocate.com/lists/state/or/',
     'https://www.beeradvocate.com/lists/state/pa/',
     'https://www.beeradvocate.com/lists/state/ri/',
     'https://www.beeradvocate.com/lists/state/sc/',
     'https://www.beeradvocate.com/lists/state/sd/',
     'https://www.beeradvocate.com/lists/state/tn/',
     'https://www.beeradvocate.com/lists/state/tx/',
     'https://www.beeradvocate.com/lists/state/ut/',
     'https://www.beeradvocate.com/lists/state/vt/',
     'https://www.beeradvocate.com/lists/state/va/',
     'https://www.beeradvocate.com/lists/state/wa/',
     'https://www.beeradvocate.com/lists/state/wv/',
     'https://www.beeradvocate.com/lists/state/wi/',
     'https://www.beeradvocate.com/lists/state/wy/',
     'https://www.beeradvocate.com/lists/au/',
     'https://www.beeradvocate.com/lists/be/',
     'https://www.beeradvocate.com/lists/br/',
     'https://www.beeradvocate.com/lists/ca/',
     'https://www.beeradvocate.com/lists/fr/',
     'https://www.beeradvocate.com/lists/de/',
     'https://www.beeradvocate.com/lists/it/',
     'https://www.beeradvocate.com/lists/jp/',
     'https://www.beeradvocate.com/lists/mx/',
     'https://www.beeradvocate.com/lists/nl/',
     'https://www.beeradvocate.com/lists/scand/',
     'https://www.beeradvocate.com/lists/uk/']




```python
page_url = r'https://www.beeradvocate.com/lists/top/'

beer_list_urls = scrape_beer_lists(page_url)
unique_beers = set()
non_unique_beers = list()

for num, beer_list_url in enumerate(beer_list_urls):
    beers = scrape_beer_urls(beer_list_url)
    unique_beers = unique_beers.union(beers)
    non_unique_beers.append(beers)

with open('beer_urls.csv', 'w+') as f:
        f.write(str(unique_beers))

with open('non_unique_beer_urls.csv', 'w+') as f:
        f.write(str(non_unique_beers))
```

    https://www.beeradvocate.com/beer/profile/23222/78820/
    https://www.beeradvocate.com/beer/profile/26/42349/
    https://www.beeradvocate.com/beer/profile/25888/87246/
    https://www.beeradvocate.com/beer/profile/28743/87846/
    https://www.beeradvocate.com/beer/profile/17981/110635/
    https://www.beeradvocate.com/beer/profile/46317/16814/
    https://www.beeradvocate.com/beer/profile/28743/146770/
    https://www.beeradvocate.com/beer/profile/28743/237238/
    https://www.beeradvocate.com/beer/profile/2216/188570/
    https://www.beeradvocate.com/beer/profile/33824/172669/
    https://www.beeradvocate.com/beer/profile/23222/162502/
    https://www.beeradvocate.com/beer/profile/863/21690/
    https://www.beeradvocate.com/beer/profile/23222/76421/


```python
all_beers = []
beers = list(unique_beers)

# For each beer page within start and end index pull the info and append it
def scrape_all_beers(beers, start, end):
    for beer in beers[start:end]:
        try:
            my_beer = scrape_beer_page(beer)
            all_beers.append(my_beer)
        except Exception:
            print('error with item')

            
# This utilizes multi-threading to start scraping all the beer pages from the list we got earlier
def split_processing(beers, num_splits=40):
    split_size = len(beers) // num_splits
    threads = []
    for i in range(num_splits):
        # determine the indices of the list this thread will handle
        start = i * split_size
        # special case on the last chunk to account for uneven splits
        end = None if i+1 == num_splits else (i+1) * split_size
        # create the thread
        threads.append(
            threading.Thread(target=scrape_all_beers, args=(beers, start, end))
        )
        threads[-1].start() # start the thread we just created

    # wait for all threads to finish                                            
    for t in threads:
        t.join()

split_processing(beers)
```

    error with item
    error with item
    error with item
    error with item
    error with itemerror with item
    
    error with item
    error with item
    error with item
    error with itemerror with itemerror with item
    
    
    error with item
    error with item
    error with item
    error with item
    error with itemerror with item
    
    error with itemerror with item
    
    error with item
    error with item
    error with item
    error with itemerror with item
    
    error with itemerror with item
    
    error with item
    error with item
    error with itemerror with item
    
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    error with item
    


```python
# This page writes the beer information into a csv file
col_names = Beer._fields
beer_data = pd.DataFrame.from_records(all_beers, columns = col_names)
beer_data.to_csv('beer_advocate.csv', index=False)
```
