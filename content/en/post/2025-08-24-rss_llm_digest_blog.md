---
title: "Building Your Own LLM-Powered RSS Digest"
date: 2025-08-24T07:42:16+01:00
draft: false  
tags: ["digital hygiene", "openai"]
---

In an era of information overload, staying up-to-date with your favorite blogs and news sources can be overwhelming.
On top of that, the internet is full of click bait content, that distract us from the real important information.
RSS feeds offer a solution, but who has time to read dozens of articles daily? This tutorial shows you how to build an intelligent RSS digest that automatically summarizes content using AI.

We'll walk through three key steps: parsing RSS feeds, extracting full article content, and generating AI-powered summaries using OpenAI's API.

## Prerequisites

Before we start, you'll need:
- Python 3.7+
- An OpenAI API key
- Basic familiarity with Python

Install the required packages:

```bash
pip install feedparser requests beautifulsoup4 openai
```

## Step 1: Parsing RSS Feeds with Python

RSS (Really Simple Syndication) feeds are XML files that websites use to publish their latest content. Python's `feedparser` library makes it easy to work with these feeds.

```python
import feedparser
import requests
from datetime import datetime, timezone

def parse_rss_feed(feed_url):
    """Parse a single RSS feed and extract articles"""
    
    # Fetch the RSS feed
    response = requests.get(feed_url, timeout=30)
    response.raise_for_status()
    
    # Parse the XML content
    feed = feedparser.parse(response.content)
    
    articles = []
    for entry in feed.entries:
        # Extract publication date
        pub_date = None
        if hasattr(entry, 'published_parsed') and entry.published_parsed:
            pub_date = datetime(*entry.published_parsed[:6], tzinfo=timezone.utc)
        
        # Get article content (summary from RSS)
        content = getattr(entry, 'summary', '')
        
        article = {
            'title': entry.get('title', 'No Title'),
            'url': entry.get('link', ''),
            'content': content,
            'published': pub_date,
            'author': entry.get('author', 'Unknown')
        }
        articles.append(article)
    
    return articles

# Example usage
feed_url = "https://feeds.bbci.co.uk/news/rss.xml"
articles = parse_rss_feed(feed_url)
print(f"Found {len(articles)} articles")
```

The `feedparser` library handles the complexity of XML parsing and provides a clean interface to access article metadata like titles, URLs, publication dates, and summary content.

## Step 2: Extracting Full Content with Beautiful Soup

RSS feeds typically only include article summaries. To get the full content for better AI analysis, we need to fetch and parse the actual web pages.

```python
from bs4 import BeautifulSoup
import re

def extract_article_content(url):
    """Fetch and extract main content from a web page"""
    
    headers = {
        'User-Agent': 'Mozilla/5.0 (compatible; RSS Reader/1.0)'
    }
    
    try:
        response = requests.get(url, headers=headers, timeout=30)
        response.raise_for_status()
        
        # Parse HTML with Beautiful Soup
        soup = BeautifulSoup(response.text, 'html.parser')
        
        # Remove unwanted elements
        for element in soup(['script', 'style', 'nav', 'footer', 'header', 'aside']):
            element.decompose()
        
        # Extract text content
        content = soup.get_text(separator=' ', strip=True)
        
        # Clean up whitespace
        content = ' '.join(content.split())
        
        # Limit content length for LLM processing
        max_chars = 8000  # Roughly 2000 tokens
        if len(content) > max_chars:
            content = content[:max_chars] + "..."
        
        return content if len(content) > 100 else None
        
    except Exception as e:
        print(f"Error extracting content from {url}: {e}")
        return None

# Enhance articles with full content
def enhance_articles(articles):
    """Add full content to articles"""
    
    for article in articles:
        if article['url']:
            full_content = extract_article_content(article['url'])
            if full_content and len(full_content) > len(article['content']):
                article['content'] = full_content
                print(f"Enhanced: {article['title']}")
    
    return articles
```

Beautiful Soup excels at parsing HTML and extracting clean text content. We remove navigation elements, scripts, and other non-content parts to focus on the article text.

## Step 3: Generating AI Summaries with OpenAI

Now comes the magic: using OpenAI's API to generate intelligent summaries of our collected articles.

```python
import openai

class DigestGenerator:
    def __init__(self, api_key):
        self.client = openai.OpenAI(api_key=api_key)
    
    def create_digest(self, articles):
        """Generate an AI-powered digest of articles"""
        
        # Prepare articles for the prompt
        article_summaries = []
        for i, article in enumerate(articles[:10], 1):  # Limit to 10 articles
            summary = {
                'title': article['title'],
                'content': article['content'][:1500],  # Truncate for token limits
                'url': article['url'],
                'published': article['published'].strftime('%Y-%m-%d') if article['published'] else 'Unknown'
            }
            article_summaries.append(summary)
        
        # Create the prompt
        prompt = self._build_digest_prompt(article_summaries)
        
        try:
            response = self.client.chat.completions.create(
                model="gpt-3.5-turbo",
                messages=[{"role": "user", "content": prompt}],
                temperature=0.3,
                max_tokens=2000
            )
            
            return response.choices[0].message.content
            
        except Exception as e:
            print(f"Error generating digest: {e}")
            return self._fallback_digest(article_summaries)
    
    def _build_digest_prompt(self, articles):
        """Build the prompt for AI digest generation"""
        
        prompt = f"""Create a comprehensive daily digest from these {len(articles)} articles. 
        
Instructions:
- Summarize the key themes and trends
- Group related topics together
- Highlight the most important developments
- Keep it engaging and informative
- Use markdown formatting

Articles:

"""
        
        for i, article in enumerate(articles, 1):
            prompt += f"""
## Article {i}: {article['title']}
**URL:** {article['url']}
**Published:** {article['published']}

{article['content']}

---
"""
        
        return prompt
    
    def _fallback_digest(self, articles):
        """Simple fallback if AI fails"""
        digest = f"# Daily Digest - {datetime.now().strftime('%Y-%m-%d')}\n\n"
        
        for article in articles:
            digest += f"## {article['title']}\n"
            digest += f"**Published:** {article['published']}\n"
            digest += f"**Link:** {article['url']}\n\n"
            digest += f"{article['content'][:200]}...\n\n---\n\n"
        
        return digest
```

The key to good AI summaries is crafting effective prompts. We give the AI clear instructions and provide structured article data for analysis.

## Putting It All Together

Here's the main function putting all three components together:

```python
import os

def main():
    RSS_FEEDS = [
        "https://feeds.bbci.co.uk/news/rss.xml",
        "http://rss.cnn.com/rss/cnn_latest.rss/"
    ]

    print("Fetching articles from RSS feeds...")
    all_articles = []
    
    # Step 1: Parse RSS feeds
    for feed_url in RSS_FEEDS:
        try:
            articles = parse_rss_feed(feed_url)
            all_articles.extend(articles)
            print(f"Fetched {len(articles)} articles from {feed_url}")
        except Exception as e:
            print(f"Error processing {feed_url}: {e}")
    
    if not all_articles:
        print("No articles found!")
        return
    
    # Sort by publication date (newest first)
    all_articles.sort(key=lambda x: x['published'] or datetime.min, reverse=True)
    
    # Step 2: Enhance with full content
    print("Extracting full article content...")
    enhanced_articles = enhance_articles(all_articles[:15])  # Process top 15
    
    # Step 3: Generate AI digest
    print("Generating AI digest...")

    openai_api_key = os.getenv("OPENAI_API_KEY")
    generator = DigestGenerator(openai_api_key)
    digest = generator.create_digest(enhanced_articles)
    
    # Save and display results
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    filename = f"digest_{timestamp}.md"
    
    with open(filename, 'w', encoding='utf-8') as f:
        f.write(digest)
    
    print(f"\n✅ Digest saved to {filename}")
    print("\n" + "="*60)
    print(digest)
    print("="*60)

if __name__ == "__main__":
    main()
```
The full program is available [here](https://gist.github.com/dovidio/a36ad141cfd6f33966a8bbfc2ffcb92f)

## Key Considerations

When building your own RSS digest system, keep these points in mind:

**Rate Limiting**: Be respectful to websites by adding delays between requests and using proper User-Agent headers.

**Error Handling**: RSS feeds can be unreliable. Always include try-catch blocks and graceful fallbacks.

**Content Filtering**: Consider adding keyword filtering to focus on topics that interest you most.

**Token Limits**: OpenAI models have token limits. Truncate content appropriately and batch large requests.

**Caching**: Store processed articles in a database to avoid re-processing and respect API usage limits. Sqlite3 is a good candidate

## Conclusion

With a couple hundred lines of Python code we were able to build a digest system that scrapes information from different feeds, and summarize those using an LLM.
I hope this inspired you to byukd your own feed. If the topic is interesting to you, check out [Colino](https://github.com/dovidio/colino), an configurable open source digest system.
