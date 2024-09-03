# brokenlinkfinder
Broken Link Checker
## Overview
This Python script automates the process of checking for broken links on websites. It uses requests, BeautifulSoup, and pandas to crawl through web pages, identify broken links, and log the results. The script also saves the broken links to an Excel file for easy reference.

## Key Features
Multithreading: Speeds up the link-checking process by using multiple threads.<br>
Logging: Keeps a detailed log of all checked links.<br>
Data Export: Exports broken links to an Excel file with a timestamp.

## Prerequisites
Python 3.x<br>
Install the required libraries

`pip install requests beautifulsoup4 pandas`
## Usage

1. Install the required libraries

`pip install requests beautifulsoup4 pandas`

2. Save the script

```
import requests
from bs4 import BeautifulSoup
from urllib.parse import urljoin, urlparse
import threading
import queue
import logging
import pandas as pd
from datetime import datetime

# Configure logging
logging.basicConfig(filename='link_checker.log', level=logging.INFO, format='%(asctime)s - %(message)s')

class LinkChecker:
    def __init__(self, base_url):
        self.base_url = base_url
        self.visited = set()
        self.broken_links = []
        self.q = queue.Queue()

    def check_link(self, url, referring_page, anchor_text):
        try:
            response = requests.head(url, allow_redirects=True)
            if response.status_code == 404:
                self.broken_links.append((url, referring_page, anchor_text))
                logging.info(f"Broken link: {url} (found on {referring_page})")
            else:
                logging.info(f"Valid link: {url} (found on {referring_page})")
        except requests.RequestException as e:
            self.broken_links.append((url, referring_page, anchor_text))
            logging.info(f"Error checking {url}: {e} (found on {referring_page})")

    def extract_links(self, url):
        try:
            response = requests.get(url)
            soup = BeautifulSoup(response.text, 'html.parser')
            for link in soup.find_all('a', href=True):
                full_url = urljoin(self.base_url, link['href'])
                anchor_text = link.get_text(strip=True)
                if self.base_url in full_url and full_url not in self.visited:
                    self.visited.add(full_url)
                    self.q.put((full_url, url, anchor_text))
        except requests.RequestException as e:
            logging.info(f"Error extracting links from {url}: {e}")

    def worker(self):
        while True:
            url, referring_page, anchor_text = self.q.get()
            if url is None:
                break
            self.check_link(url, referring_page, anchor_text)
            self.extract_links(url)
            self.q.task_done()

    def run(self, num_threads=10):
        self.q.put((self.base_url, self.base_url, ''))
        threads = []
        for _ in range(num_threads):
            t = threading.Thread(target=self.worker)
            t.start()
            threads.append(t)
        self.q.join()
        for _ in range(num_threads):
            self.q.put((None, None, None))
        for t in threads:
            t.join()

        if self.broken_links:
            df = pd.DataFrame(self.broken_links, columns=['Broken Link', 'Referring Page', 'Anchor Text'])
            print("Broken links found:")
            print(df)
            # Save to Excel file with a unique name
            timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
            filename = f'broken_links_{timestamp}.xlsx'
            df.to_excel(filename, index=False)
            print(f"Broken links have been saved to '{filename}'.")
        else:
            print("No broken links found.")

if __name__ == "__main__":
    website_url = 'https://example.com'  # Replace with the target website URL (don't remove '')
    checker = LinkChecker(website_url)
    checker.run()
```
3. Save the script

   Change the `https://example.com` with your target url. Save the script and run it. 


