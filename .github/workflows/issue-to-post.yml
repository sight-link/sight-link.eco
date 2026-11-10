#!/usr/bin/env python3
"""Turn an issue opened by the repository owner into one entry in updates.json.

Convention for the issue:
  title                 -> text   (required, the whole post)
  body line "tag: life" -> tag    (optional)
  body line "link: URL" -> link   (optional, http/https only)
  a pasted image        -> downloaded into images/ and referenced by relative path
Any other body text is ignored on purpose, so notes never leak into the post.
"""

import json
import os
import re
import sys
import urllib.parse
import urllib.request
from datetime import datetime, timedelta, timezone
from pathlib import Path

ROOT = Path(".")
DATA = ROOT / "updates.json"
IMAGES = ROOT / "images"
MAX_IMAGE_BYTES = 10 * 1024 * 1024
ALLOWED_HOSTS = ("github.com", "githubusercontent.com")
SAFE_EXT = {
    "image/jpeg": "jpg",
    "image/png": "png",
    "image/webp": "webp",
    "image/gif": "gif",
}


def fail(message):
    print("ERROR: " + message, file=sys.stderr)
    sys.exit(1)


def set_output(key, value):
    path = os.environ.get("GITHUB_OUTPUT")
    if path:
        with open(path, "a", encoding="utf-8") as handle:
            handle.write("%s=%s\n" % (key, value))


def host_allowed(url):
    host = (urllib.parse.urlparse(url).hostname or "").lower()
    return any(host == allowed or host.endswith("." + allowed) for allowed in ALLOWED_HOSTS)


def download_image(url, today, number):
    if not host_allowed(url):
        print("Image host not allowed; falling back to grey placeholder.")
        return ""
    request = urllib.request.Request(url, headers={"User-Agent": "sight-link-post-bot"})
    try:
        with urllib.request.urlopen(request, timeout=20) as response:
            content_type = (response.headers.get("Content-Type") or "").split(";")[0].strip().lower()
            extension = SAFE_EXT.get(content_type)
            if not extension:
                print("Unsupported image type %s; falling back to placeholder." % content_type)
                return ""
            raw = response.read(MAX_IMAGE_BYTES + 1)
    except Exception as error:
        print("Image download failed (%s); falling back to placeholder." % error)
        return ""
    if len(raw) > MAX_IMAGE_BYTES:
        print("Image exceeds 10 MB; falling back to placeholder.")
        return ""
    IMAGES.mkdir(exist_ok=True)
    suffix = "-%s" % number if number is not None else ""
    relative = "images/%s%s.%s" % (today, suffix, extension)
    (ROOT / relative).write_bytes(raw)
    return relative


def main():
    event_path = os.environ.get("GITHUB_EVENT_PATH")
    if not event_path:
        fail("GITHUB_EVENT_PATH is not set")
    event = json.loads(Path(event_path).read_text(encoding="utf-8"))
    issue = event.get("issue") or {}
    author = (issue.get("user") or {}).get("login", "")
    owner = os.environ.get("REPO_OWNER", "")
    if owner and author != owner:
        fail("issue author %r is not the repository owner" % author)

    number = issue.get("number")
    title = (issue.get("title") or "").strip()
    # Strip template comments so example lines inside them are never parsed as metadata.
    body = re.sub(r"<!--[\s\S]*?-->", "", issue.get("body") or "")
    if not title:
        fail("issue title is empty")

    posts = json.loads(DATA.read_text(encoding="utf-8"))
    if not isinstance(posts, list):
        fail("updates.json must contain a JSON array")
    if number is not None and any(
        isinstance(post, dict) and str(post.get("source_issue")) == str(number)
        for post in posts
    ):
        print("Issue #%s already published; nothing to do." % number)
        set_output("changed", "false")
        return

    tag, link = "", ""
    for line in body.splitlines():
        match = re.match(r"^\s*tag\s*[:：]\s*(\S+)\s*$", line, re.IGNORECASE)
        if match:
            tag = match.group(1)
        match = re.match(r"^\s*link\s*[:：]\s*(\S+)\s*$", line, re.IGNORECASE)
        if match:
            link = match.group(1)
    if link and not re.match(r"^https?://", link, re.IGNORECASE):
        fail("link must start with http:// or https://")

    # Fixed UTC+8: China has no daylight saving, and this avoids a tzdata dependency.
    today = datetime.now(timezone(timedelta(hours=8))).date().isoformat()

    image = ""
    image_match = re.search(r"!\[[^\]]*\]\((https?://[^)\s]+)\)", body)
    if image_match:
        image = download_image(image_match.group(1), today, number)

    post = {
        "date": today,
        "text": title,
        "image": image,
        "link": link,
        "tag": tag,
    }
    if number is not None:
        post["source_issue"] = number
    posts.insert(0, post)
    DATA.write_text(json.dumps(posts, ensure_ascii=False, indent=2) + "\n", encoding="utf-8")
    print("Added post: %s | %s | image=%s" % (today, title[:40], image or "none"))
    set_output("changed", "true")


if __name__ == "__main__":
    main()
