---
title: "How to Summarize YouTube Videos with LLMs"
date: 2025-08-25T07:42:16+01:00
draft: false  
tags: ["digital hygiene", "fight clickbait"]
---

Ever find yourself wanting to get the key points from a long YouTube video without watching the entire thing? This tutorial shows you how to automatically generate summaries using YouTube transcripts and AI.

We'll walk through three simple steps: getting video transcripts, feeding them to an LLM, and enjoying the results.

## Prerequisites

Before we start, you'll need:
- Python 3.7+
- An OpenAI API key
- YouTube videos with available transcripts

Install the required packages:
```bash
pip install youtube-transcript-api openai
```

## Step 1: Extract YouTube Transcripts

YouTube automatically generates transcripts for most videos. The `youtube-transcript-api` library makes accessing them straightforward.

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

The transcript API handles different languages and formats automatically. Most popular videos have transcripts available.

## Step 2: Generate Summaries with OpenAI

Now we feed the transcript to an LLM to generate an intelligent summary.

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

The key is crafting a clear prompt that tells the AI exactly what kind of summary you want.

## Step 3: Put It All Together

Here's the complete pipeline:

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

## Conclusion

With about 100 lines of Python, you can build a system that automatically summarizes any YouTube video with available transcripts. The three-step process—extract transcript, process with LLM, enjoy results—works reliably for most content.

This approach is particularly useful for educational videos, interviews, and technical talks where you want to quickly understand the main points without watching the entire video.
Or, if you're like me, you can find out what's behind clickbait titles.

The complete code is available [here](https://gist.github.com/dovidio/e03484f23cb01d085e2c9f11346c034c)
