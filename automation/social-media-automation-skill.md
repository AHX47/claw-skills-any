# Social Media Automation Skills — Official APIs

## YouTube Data API v3
```python
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload
from google.oauth2.credentials import Credentials
import json

# Authenticate with OAuth2
creds = Credentials.from_authorized_user_file("token.json",
    scopes=["https://www.googleapis.com/auth/youtube.upload",
            "https://www.googleapis.com/auth/youtube.readonly"])

youtube = build("youtube","v3", credentials=creds)

def upload_video(video_path: str, title: str, description: str,
                  tags: list[str], category_id: str = "22",
                  privacy: str = "public") -> str:
    body = {"snippet": {"title": title, "description": description,
                          "tags": tags, "categoryId": category_id},
            "status":  {"privacyStatus": privacy}}
    media = MediaFileUpload(video_path, mimetype="video/*", resumable=True,
                             chunksize=1024*1024*5)
    request = youtube.videos().insert(part="snippet,status",
                                        body=body, media_body=media)
    response = None
    while response is None:
        status, response = request.next_chunk()
        if status: print(f"Upload progress: {int(status.progress()*100)}%")
    print(f"Uploaded: https://youtube.com/watch?v={response['id']}")
    return response["id"]

def get_channel_stats(channel_id: str) -> dict:
    r = youtube.channels().list(part="statistics", id=channel_id).execute()
    return r["items"][0]["statistics"]

def get_video_analytics(video_id: str) -> dict:
    r = youtube.videos().list(part="statistics,snippet", id=video_id).execute()
    return r["items"][0]
```

## Instagram Graph API (Business/Creator accounts)
```python
import requests

class InstagramAPI:
    BASE = "https://graph.instagram.com"

    def __init__(self, access_token: str, ig_user_id: str):
        self.token   = access_token
        self.user_id = ig_user_id

    def _get(self, path, **params):
        params["access_token"] = self.token
        r = requests.get(f"{self.BASE}{path}", params=params)
        r.raise_for_status(); return r.json()

    def _post(self, path, **data):
        data["access_token"] = self.token
        r = requests.post(f"{self.BASE}{path}", data=data)
        r.raise_for_status(); return r.json()

    def create_post(self, image_url: str, caption: str) -> str:
        """Create and publish an image post."""
        # Step 1: Create media container
        container = self._post(f"/{self.user_id}/media",
                                 image_url=image_url, caption=caption)
        # Step 2: Publish
        result = self._post(f"/{self.user_id}/media_publish",
                              creation_id=container["id"])
        return result["id"]

    def create_reel(self, video_url: str, caption: str,
                     share_to_feed: bool = True) -> str:
        """Upload a Reel."""
        container = self._post(f"/{self.user_id}/media",
                                 media_type="REELS",
                                 video_url=video_url,
                                 caption=caption,
                                 share_to_feed=str(share_to_feed).lower())
        import time
        # Wait for video processing
        for _ in range(30):
            status = self._get(f"/{container['id']}",
                                fields="status_code,status")
            if status.get("status_code") == "FINISHED": break
            time.sleep(10)
        result = self._post(f"/{self.user_id}/media_publish",
                              creation_id=container["id"])
        return result["id"]

    def get_insights(self, media_id: str) -> dict:
        return self._get(f"/{media_id}/insights",
                          metric="engagement,impressions,reach,saved")
```

## Facebook Graph API (Pages)
```python
import requests

class FacebookPageAPI:
    BASE = "https://graph.facebook.com/v19.0"

    def __init__(self, page_access_token: str, page_id: str):
        self.token   = page_access_token
        self.page_id = page_id

    def post_text(self, message: str) -> str:
        r = requests.post(f"{self.BASE}/{self.page_id}/feed",
                           data={"message": message, "access_token": self.token})
        r.raise_for_status()
        return r.json()["id"]

    def post_photo(self, image_url: str, caption: str) -> str:
        r = requests.post(f"{self.BASE}/{self.page_id}/photos",
                           data={"url": image_url, "caption": caption,
                                  "access_token": self.token})
        r.raise_for_status()
        return r.json()["post_id"]

    def post_video(self, video_url: str, title: str, description: str) -> str:
        r = requests.post(f"{self.BASE}/{self.page_id}/videos",
                           data={"file_url": video_url, "title": title,
                                  "description": description,
                                  "access_token": self.token})
        r.raise_for_status()
        return r.json()["id"]

    def get_page_insights(self, metric: str = "page_views_total",
                           period: str = "day") -> list:
        r = requests.get(f"{self.BASE}/{self.page_id}/insights",
                          params={"metric": metric, "period": period,
                                   "access_token": self.token})
        r.raise_for_status()
        return r.json()["data"]

    def schedule_post(self, message: str, publish_time: int) -> str:
        """publish_time: Unix timestamp in the future."""
        r = requests.post(f"{self.BASE}/{self.page_id}/feed",
                           data={"message": message,
                                  "published": "false",
                                  "scheduled_publish_time": publish_time,
                                  "access_token": self.token})
        r.raise_for_status()
        return r.json()["id"]
```

## Content Scheduler
```python
import schedule, time, threading
from datetime import datetime, timedelta

class ContentScheduler:
    def __init__(self):
        self.queue   = []
        self.running = False

    def add(self, platform: str, content: dict, publish_at: datetime):
        self.queue.append({
            "platform": platform, "content": content,
            "publish_at": publish_at, "status": "pending"
        })
        self.queue.sort(key=lambda x: x["publish_at"])

    def start(self):
        self.running = True
        def loop():
            while self.running:
                now = datetime.now()
                for job in [j for j in self.queue if j["status"]=="pending"]:
                    if job["publish_at"] <= now:
                        self._publish(job)
                time.sleep(60)
        threading.Thread(target=loop, daemon=True).start()

    def _publish(self, job: dict):
        try:
            platform = job["platform"]
            # Route to correct API
            if platform == "youtube": upload_video(**job["content"])
            elif platform == "instagram": ig_api.create_post(**job["content"])
            elif platform == "facebook": fb_api.post_text(**job["content"])
            job["status"] = "published"
            print(f"Published to {platform} at {datetime.now()}")
        except Exception as e:
            job["status"] = "failed"
            job["error"] = str(e)
            print(f"Failed {platform}: {e}")
```

## Video Generation (FFmpeg)
```python
import subprocess
from pathlib import Path

def create_slideshow(images: list[str], audio: str,
                      output: str, fps: int = 24,
                      duration_per_image: float = 3.0) -> str:
    """Create a video from images + audio using FFmpeg."""
    # Write image list file
    list_file = Path("tmp_images.txt")
    with open(list_file, "w") as f:
        for img in images:
            f.write(f"file '{img}'\nduration {duration_per_image}\n")
        f.write(f"file '{images[-1]}'\n")  # FFmpeg requires last file repeated

    cmd = [
        "ffmpeg", "-y",
        "-f", "concat", "-safe", "0", "-i", str(list_file),  # images input
        "-i", audio,                                            # audio input
        "-vf", f"scale=1080:1920:force_original_aspect_ratio=decrease,"
               f"pad=1080:1920:(ow-iw)/2:(oh-ih)/2,fps={fps}",
        "-c:v", "libx264", "-preset", "fast", "-crf", "23",
        "-c:a", "aac", "-b:a", "192k",
        "-shortest",  # match audio duration
        output
    ]
    subprocess.run(cmd, check=True)
    list_file.unlink()
    return output

def add_text_overlay(input_video: str, text: str,
                      output: str, font_size: int = 48) -> str:
    """Add Arabic/text overlay to video."""
    cmd = [
        "ffmpeg", "-y", "-i", input_video,
        "-vf", f"drawtext=text='{text}':fontsize={font_size}:"
               f"fontcolor=white:x=(w-text_w)/2:y=h-100:"
               f"box=1:boxcolor=black@0.5:boxborderw=10",
        "-c:a", "copy", output
    ]
    subprocess.run(cmd, check=True)
    return output
```

## Rate Limits Reference
```
YouTube Data API:    10,000 units/day (upload = 1600 units)
Instagram Graph:     200 calls/hour per user
Facebook Graph:      200 calls/hour per user
Twitter/X API v2:   500K tweets/month (Basic), 100M (Pro)

Always:
- Implement exponential backoff on 429 responses
- Cache API responses to reduce quota usage
- Use webhooks instead of polling where available
```
