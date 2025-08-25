---
title: "Come Riassumere Video YouTube con gli LLM"
date: 2025-08-25T07:42:16+01:00
draft: false  
tags: ["igiene digitale", "combatti il clickbait"]
---

Ti è mai capitato di voler ottenere i punti chiave di un lungo video YouTube senza guardarlo per intero? Questo tutorial ti mostra come generare automaticamente riassunti usando le trascrizioni di YouTube e l'AI.

Seguiremo tre semplici passaggi: ottenere le trascrizioni video, darle in pasto a un LLM, e godersi i risultati.

## Prerequisiti

Prima di iniziare, avrai bisogno di:
- Python 3.7+
- Una chiave API OpenAI
- Video YouTube con trascrizioni disponibili

Installa i pacchetti necessari:
```bash
pip install youtube-transcript-api openai
```

## Passaggio 1: Estrarre le Trascrizioni YouTube

YouTube genera automaticamente trascrizioni per la maggior parte dei video. La libreria `youtube-transcript-api` rende l'accesso semplice e diretto.

```python
from youtube_transcript_api import YouTubeTranscriptApi
from youtube_transcript_api.formatters import TextFormatter
from urllib.parse import urlparse, parse_qs

def extract_video_id(url):
    """Extract video ID from various YouTube URL formats"""
    if 'youtu.be/' in url:
        return url.split('youtu.be/')[-1].split('?')[0]
    elif 'youtube.com/watch?v=' in url:
        parsed = urlparse(url)
        return parse_qs(parsed.query)['v'][0]
    return None
    
def get_video_transcript(video_id, languages=['en']):
    """Fetch transcript for a YouTube video"""
    try:
        transcript_api = YouTubeTranscriptApi()
        transcript_list = transcript_api.fetch(video_id=video_id, languages=languages)
        
        if not transcript_list:
            return None
        
        formatter = TextFormatter()
        transcript_text = formatter.format_transcript(transcript_list)
        
        # Clean up the text
        transcript_text = transcript_text.replace('\n', ' ')
        transcript_text = ' '.join(transcript_text.split())
        
        return transcript_text
        
    except Exception as e:
        print(f"Could not retrieve transcript: {e}")
        return None

# Example usage
video_url = "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
video_id = extract_video_id(video_url)
transcript = get_video_transcript(video_id)
print(f"Transcript length: {len(transcript)} characters")
```

L'API delle trascrizioni gestisce automaticamente lingue e formati diversi. La maggior parte dei video popolari ha trascrizioni disponibili.

## Passaggio 2: Generare Riassunti con OpenAI

Ora diamo la trascrizione a un LLM per generare un riassunto intelligente.

```python
import openai

class VideoSummarizer:
    def __init__(self):
        self.client = openai.OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
    
    def summarize_video(self, transcript, model='gpt-5-mini'):
        """Generate summary from video transcript"""
        
        # Truncate if too long (roughly 8000 chars = 2000 tokens)
        if len(transcript) > 8000:
            transcript = transcript[:8000] + "..."
        
        prompt = f"""
Analyze this YouTube video transcript and provide a clear, structured summary.

Instructions:
- Identify the main topic and key points
- Highlight important insights or conclusions
- Use bullet points for clarity
- Keep it concise but comprehensive

Transcript:
{transcript}

Summary:
"""
        
        try:
            response = self.client.chat.completions.create(
                model=model,
                messages=[{"role": "user", "content": prompt}],
                max_completion_tokens=4096
            )

            return response.choices[0].message.content
            
        except Exception as e:
            print(f"Error generating summary: {e}")
            return f"Summary failed. First 500 chars: {transcript[:500]}..."

# Example usage
summarizer = VideoSummarizer()
summary = summarizer.summarize_video(transcript)
print(summary)
```

La chiave è creare un prompt chiaro che dica all'AI esattamente che tipo di riassunto vuoi.

## Passaggio 3: Mettere Tutto Insieme

Ecco la pipeline completa:

```python
def summarize_youtube_video(video_url):
    """Complete pipeline to summarize a YouTube video"""
    
    print(f"Processing: {video_url}")
    
    # Step 1: Extract video ID and transcript
    video_id = extract_video_id(video_url)
    if not video_id:
        return "Invalid YouTube URL"
    
    transcript = get_video_transcript(video_id)
    if not transcript:
        return "No transcript available for this video"
    
    print(f"Transcript extracted: {len(transcript)} characters")
    
    # Step 2: Generate summary
    summarizer = VideoSummarizer()
    summary = summarizer.summarize_video(transcript)
    
    return summary

# Example usage
video_url = "https://www.youtube.com/watch?v=your_video_here"

summary = summarize_youtube_video(video_url)
print("\n" + "="*60)
print("VIDEO SUMMARY")
print("="*60)
print(summary)
```

## Conclusione

Con circa 100 righe di Python, puoi costruire un sistema che riassume automaticamente qualsiasi video YouTube con trascrizioni disponibili. Il processo in tre fasi—estrarre trascrizione, elaborare con LLM, godersi i risultati—funziona in modo affidabile per la maggior parte dei contenuti.

Questo approccio è particolarmente utile per video educativi, interviste e conferenze tecniche dove vuoi capire rapidamente i punti principali senza guardare l'intero video.
Oppure, se sei come me, puoi scoprire cosa si nasconde dietro i titoli clickbait.

Il codice completo è disponibile [qui](https://gist.github.com/dovidio/e03484f23cb01d085e2c9f11346c034c)
