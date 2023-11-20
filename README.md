# Scraping-Articles-with-Python
You can use this code to scrape articles from the ZEIT archive. I provide the corresponding data set here.
A subscription to DIE ZEIT or a registered account with online access is required for scraping. The code works in three steps. First, a list is created from the individual archive pages, which always follow the same URL pattern. Then the code scrapes the links to the articles from the individual archive pages. Finally, I generate a function to retrieve the information from these articles: Author, date and text. Note that the printed notifications are in German, but should be easy to adjust if needed.

### creating list of weeks to scrape
```
import requests
from bs4 import BeautifulSoup
from tqdm import tqdm
import time

def generate_zeit_urls():
    base_url = "https://www.zeit.de/{year}/{week}/index"
    zeit_urls = []

    for year in range(1995, 2020):
        for week in range(1, 53):
            week_str = str(week).zfill(2)  # (Must be added since otherwise zeros are missing)
            url = base_url.format(year=year, week=week_str)
            zeit_urls.append(url)

    return zeit_urls

```
Make sure, you installed all packages in advance using pip. I chose the years 1995 to 2020 for a specific task, however, this can easily be adjusted. In the following we now first define a function scraping the links of the artikels that are of interest to you. You need to adjust for your user credentials.
### Scraping relevant article links

```

def scrape_article_links(urls):
    login_url = 'https://meine.zeit.de/anmelden?url=https%3A%2F%2Fwww.zeit.de%2Findex&entry_service=sonstige'
    payload = {
        'username': 'YOURUSERNAME',
        'password': 'YOURPASSWORD',
    }

    article_links_list = []
# from here on we start the session to get the links
    with requests.Session() as session:
        login_request = session.post(login_url, data=payload)
        if login_request.status_code == 200:
            for url in tqdm(urls, desc="Scraping Article Links", unit="link"):
                response = session.get(url)
                if response.status_code == 200:
                    soup = BeautifulSoup(response.content, 'html.parser')
                    article_links = soup.find_all('a', class_='teaser-large__faux-link') + \
                                    soup.find_all('a', class_='teaser-small__faux-link')

                    for link in article_links:
                        article_url = link.get('href')
                        article_links_list.append(article_url)  #saving the links as a list
                else:
                    print(f"Fehler beim Abrufen der Seite {url}")
                time.sleep(0.5)  # to not overcrowd the website use time sleep 
        else:
            print("Anmeldung fehlgeschlagen!")

    return article_links_list

# generate list from initial URLS per year and week
zeit_links = generate_zeit_urls()

# Scraping article links + saving
scraped_article_links = scrape_article_links(zeit_links)
print(scraped_article_links)

```
Great! Now we want to get the infos. For that, I use pandas, to store them into a dataframe afterwards.

### Getting article infos
```

import pandas as pd

def scrape_article_info(article_links):
    article_info = []

    with requests.Session() as session:
        for article_url in tqdm(article_links, desc="Scraping Article Info", unit="article"):
            response = session.get(article_url)
            if response.status_code == 200:
                soup = BeautifulSoup(response.content, 'html.parser')
                article_text = soup.select_one('.article-page')
                article_text = article_text.text.strip() if article_text else ''
                author = soup.select_one('.byline span')
                author = author.text.strip() if author else ''
                date = soup.select_one('.metadata__date')
                date = date.text.strip() if date else ''
                article_info.append({'Text': article_text, 'Author': author, 'Date': date})
            else:
                print(f"Fehler beim Abrufen des Artikels {article_url}")
            time.sleep(1)  # to not overcrowd the website use time sleep of one sec

    return pd.DataFrame(article_info)

# Scraping begins 
articles_info_df = scrape_article_info(scraped_article_links)

# pd dataframe:
print(articles_info_df)

```
