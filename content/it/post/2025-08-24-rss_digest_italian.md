---
title: "Costruire il tuo Digest RSS Alimentato da IA"
date: 2025-08-24T07:42:16+01:00
draft: false  
tags: ["igiene digitale", "openai"]
---

In un'era di sovraccarico informativo, rimanere aggiornati con le ultime notizie preferite può essere impegnativo.
Inoltre, internet è pieno di contenuti clickbait che ci distraggono dalle informazioni davvero importanti.
I feed RSS offrono una soluzione, ma chi ha tempo di leggere decine di articoli ogni giorno? Questo tutorial ti mostra come costruire un digest intelligente RSS che riassume automaticamente i contenuti usando l'IA.

Analizzeremo tre passaggi chiave: analizzare i feed RSS, estrarre il contenuto completo degli articoli e generare riassunti alimentati dall'IA usando l'API di OpenAI.

## Prerequisiti

Prima di iniziare, avrai bisogno di:
- Python 3.7+
- Una chiave API OpenAI
- Familiarità di base con Python

Installa i pacchetti richiesti:

```bash
pip install feedparser requests beautifulsoup4 openai
```

## Passaggio 1: Analizzare i Feed RSS con Python

I feed RSS (Really Simple Syndication) sono file XML che i siti web usano per pubblicare i loro contenuti più recenti. La libreria `feedparser` di Python rende facile lavorare con questi feed.

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

La libreria `feedparser` gestisce la complessità dell'analisi XML e fornisce un'interfaccia pulita per accedere ai metadati degli articoli come titoli, URL, date di pubblicazione e contenuto riassuntivo.

## Passaggio 2: Estrarre il Contenuto Completo con Beautiful Soup

I feed RSS tipicamente includono solo riassunti degli articoli. Per ottenere il contenuto completo per una migliore analisi IA, dobbiamo recuperare e analizzare le pagine web reali.

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

Beautiful Soup eccelle nell'analisi HTML e nell'estrazione di contenuto testuale pulito. Rimuoviamo elementi di navigazione, script e altre parti non-contenuto per concentrarci sul testo dell'articolo.

## Passaggio 3: Generare Riassunti IA con OpenAI

Ora arriva la magia: usare l'API di OpenAI per generare riassunti intelligenti dei nostri articoli raccolti.

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

La chiave per buoni riassunti IA è creare prompt efficaci. Diamo all'IA istruzioni chiare e forniamo dati dell'articolo strutturati per l'analisi.

## Mettere Tutto Insieme

Ecco la funzione principale che mette insieme tutti e tre i componenti:

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
Il programma completo è disponibile [qui](https://gist.github.com/dovidio/a36ad141cfd6f33966a8bbfc2ffcb92f)

## Conclusioni

Con un paio di centinaia di righe di codice Python siamo riusciti a costruire un sistema di digest che raccoglie informazioni da diversi feed e li riassume usando un LLM.
Spero che questo ti abbia ispirato a costruire il tuo feed personale. Se l'argomento ti interessa, dai un'occhiata a [Colino](https://github.com/dovidio/colino), un sistema di digest open source configurabile.
