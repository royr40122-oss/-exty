import asyncio
import json
import cloudscraper
from typing import Dict, List, Optional

class PWFreeScraper:
    def init(self, token: str):
        self.base_urls = [
            "https://api.penpencil.co/v3/batches/",
            "https://api.penpencil.co/v2/batches/",
            "https://api.penpencil.co/v1/batches/"
        ]
        self.headers = {
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36",
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json"
        }
        self.scraper = cloudscraper.create_scraper()
        self.max_retries = 3

    async def fetch_url(self, url: str) -> Optional[Dict]:
        for attempt in range(self.max_retries):
            try:
                response = await asyncio.to_thread(
                    self.scraper.get,
                    url,
                    headers=self.headers
                )
                response.raise_for_status()
                return response.json()
            except Exception as e:
                print(f"Attempt {attempt + 1} failed for {url}: {str(e)}")
                if attempt == self.max_retries - 1:
                    return None
                await asyncio.sleep(2 ** attempt)
        return None

    async def process_topic(self, batch_id: str, topic_slug: str) -> List[str]:
        urls = []
        content_types = ["videos", "DppVideos"]
        
        for content_type in content_types:
            api_url = f"https://api.penpencil.co/v1/batches/{batch_id}/topic/{topic_slug}/contents?page=1&contentType={content_type}"
            
            data = await self.fetch_url(api_url)
            if not data or not data.get('data'):
                continue

            for item in data['data']:
                if video_id := item.get('videoId'):
                    urls.append(
                        f"https://www.pw.live/study/batches/{batch_id}/"
                        f"{topic_slug}/videos/{video_id}"
                    )
        return urls

    async def handle_pw_free_logic(self, batch_url: str) -> str:
        try:
            batch_slug = batch_url.split("/")[-2]
            batch_api_url = f"https://api.penpencil.co/v1/batches/{batch_slug}/details"
            
            batch_data = await self.fetch_url(batch_api_url)
            if not batch_data:
                return "Failed to fetch batch details"

            batch_id = batch_data['data']['id']
            topics = batch_data['data'].get('subjects', [])
            
            tasks = []
            for topic in topics:
                if topic_slug := topic.get('slug'):
                    tasks.append(self.process_topic(batch_id, topic_slug))

            results = await asyncio.gather(*tasks)
            all_urls = [url for sublist in results for url in sublist]

            if not all_urls:
                return "No URLs found for this batch"

            filename = f"pw_batch_{batch_id}_urls.txt"
            with open(filename, "w") as f:
                f.write("\n".join(all_urls))
            
            return f"Successfully wrote {len(all_urls)} URLs to {filename}"

        except Exception as e:
            return f"Error processing batch: {str(e)}"

async def main():
    token = input("Enter your PW auth token: ")
    batch_url = input("Enter batch overview URL: ")
    
    scraper = PWFreeScraper(token)
    result = await scraper.handle_pw_free_logic(batch_url)
    print(result)

    # Cleanup
    scraper.scraper.close()

if name == "main":
    asyncio.run(main())
