import tkinter as tk
from tkinter import messagebox
import queue
import threading
import time
import urllib.parse
import urllib.request
from urllib.error import URLError
import ssl
from bs4 import BeautifulSoup
import logging

logging.basicConfig(level=logging.INFO)

class Crawler(threading.Thread):
    active_threads = 0

    def __init__(self, base_url, links_to_crawl, have_visited, error_links, url_lock, max_links=None):
        threading.Thread.__init__(self)
        self.base_url = base_url
        self.links_to_crawl = links_to_crawl
        self.have_visited = have_visited
        self.error_links = error_links 
        self.url_lock = url_lock
        self.max_links = max_links
        self.my_ssl = ssl.create_default_context()
        self.my_ssl.check_hostname = False
        self.my_ssl.verify_mode = ssl.CERT_NONE
        Crawler.active_threads += 1

    def run(self):
        while True:
            self.url_lock.acquire()
            if self.links_to_crawl.empty() or (self.max_links is not None and len(self.have_visited) >= self.max_links):
                self.url_lock.release()
                break
            link = self.links_to_crawl.get()
            self.url_lock.release()

            if link is None:
                break

            if link in self.have_visited:
                logging.info(f"The link {link} is already visited")
                continue

            try:
                link = urllib.parse.urljoin(self.base_url, link)
                req = urllib.request.Request(link, headers={'User-Agent': 'MyCrawler/1.0'})
                response = urllib.request.urlopen(req, context=self.my_ssl)

                logging.info(f"The URL {response.geturl()} crawled with status {response.getcode()}")

                soup = BeautifulSoup(response.read(), "html.parser")

                for a_tag in soup.find_all('a'):
                    href = a_tag.get("href")
                    if href and (urllib.parse.urlparse(href).netloc == urllib.parse.urlparse(self.base_url).netloc):
                        self.links_to_crawl.put(href)

                logging.info(f"Adding {link} to the crawled list")
                self.have_visited.add(link)

            except URLError as e:
                logging.error(f"URL {link} threw this error {e.reason} while trying to parse")
                self.error_links.append(link)

            except Exception as e:
                logging.error(f"Unexpected error {e} occurred while processing {link}")
                self.error_links.append(link)

            finally:
                self.links_to_crawl.task_done()

        Crawler.active_threads -= 1

class WebCrawlerGUI:
    def __init__(self, master):
        self.master = master
        self.master.title("Web Crawler")

        self.create_widgets()

    def create_widgets(self):
        # Label and Entry for Website URL
        self.label_url = tk.Label(self.master, text="Website URL:")
        self.label_url.grid(row=0, column=0, padx=10, pady=5, sticky="w")
        self.entry_url = tk.Entry(self.master, width=50)
        self.entry_url.grid(row=0, column=1, columnspan=2, padx=10, pady=5)

        # Label and Entry for Number of Threads
        self.label_threads = tk.Label(self.master, text="Number of Threads:")
        self.label_threads.grid(row=1, column=0, padx=10, pady=5, sticky="w")
        self.entry_threads = tk.Entry(self.master)
        self.entry_threads.grid(row=1, column=1, padx=10, pady=5)

        # Label and Entry for Maximum Number of Links to Crawl
        self.label_max_links = tk.Label(self.master, text="Maximum Number of Links to Crawl:")
        self.label_max_links.grid(row=2, column=0, padx=10, pady=5, sticky="w")
        self.entry_max_links = tk.Entry(self.master)
        self.entry_max_links.grid(row=2, column=1, padx=10, pady=5)

        # Start Crawling Button
        self.start_button = tk.Button(self.master, text="Start Crawling", command=self.start_crawling)
        self.start_button.grid(row=3, column=1, padx=10, pady=10)

        # Stop Crawling Button
        self.stop_button = tk.Button(self.master, text="Stop Crawling", command=self.stop_crawling, state=tk.DISABLED)
        self.stop_button.grid(row=3, column=2, padx=10, pady=10)

        # Status Label
        self.status_label = tk.Label(self.master, text="", fg="blue")
        self.status_label.grid(row=4, column=0, columnspan=3, padx=10, pady=5)

    def start_crawling(self):
        self.status_label.config(text="Crawling in progress...", fg="blue")
        self.start_button.config(state=tk.DISABLED)
        self.stop_button.config(state=tk.NORMAL)

        base_url = self.entry_url.get()
        number_of_threads = int(self.entry_threads.get())
        max_links = int(self.entry_max_links.get()) if self.entry_max_links.get() else None

        self.links_to_crawl = queue.Queue()
        self.url_lock = threading.Lock()
        self.links_to_crawl.put(base_url)

        self.have_visited = set()
        self.error_links = []
        self.crawler_threads = []

        for i in range(number_of_threads):
            crawler = Crawler(base_url=base_url, links_to_crawl=self.links_to_crawl, have_visited=self.have_visited,
                              error_links=self.error_links, url_lock=self.url_lock, max_links=max_links)
            crawler.start()
            self.crawler_threads.append(crawler)

        self.check_progress()

    def stop_crawling(self):
        for crawler in self.crawler_threads:
            crawler.join()
        self.status_label.config(text="Crawling stopped", fg="red")
        self.start_button.config(state=tk.NORMAL)
        self.stop_button.config(state=tk.DISABLED)

    def check_progress(self):
        if Crawler.active_threads > 0:
            self.master.after(1000, self.check_progress)
        else:
            self.status_label.config(text="Crawling completed", fg="green")
            self.start_button.config(state=tk.NORMAL)
            self.stop_button.config(state=tk.DISABLED)
            messagebox.showinfo("Crawling Completed", f"Total Number of pages visited are {len(self.have_visited)}\nTotal Number of Erroneous links: {len(self.error_links)}")

def main():
    root = tk.Tk()
    gui = WebCrawlerGUI(root)
    root.mainloop()

if __name__ == "__main__":
    main()
